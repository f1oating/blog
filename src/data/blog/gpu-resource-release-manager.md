---
author: Dmytro Toronchenko
pubDatetime: 2026-05-10T10:00:00Z
title: "Safe Deferred Resource Release in Vulkan and DirectX 12"
slug: gpu-resource-release-manager
featured: true
draft: false
tags:
  - vulkan
  - directx12
  - cpp
  - graphics
  - rhi
description: How to build a timeline-based release manager that safely defers GPU resource destruction until the GPU is actually done with them — without stalling the CPU.
---

Every Vulkan and DirectX 12 renderer eventually hits the same wall: you want to destroy a resource — a buffer, an image, a pipeline — but the GPU is still reading from it. Call the destructor too early and you get a validation error or, worse, silent corruption. Stall the CPU until the GPU is idle and you've thrown away your pipelining budget. The right answer is deferred release: schedule the destruction, let the GPU finish, then clean up.

This post walks through a production-grade `ReleaseManager` that handles this precisely, using timeline semaphore values (Vulkan) or fence values (D3D12) to drive cleanup.

## The problem

In a pipelined renderer, the CPU is always ahead of the GPU. Frame N's commands are recorded and submitted while the GPU is still executing frame N-2. If you delete a buffer the moment the CPU is done with it, you pull the rug out from under the in-flight GPU work.

The naive fix — `vkDeviceWaitIdle` before every free — serializes the entire pipeline. That's acceptable during teardown, not during a frame.

The correct model is: **record when a resource becomes unreferenced, bind that moment to a timeline value, and free the resource once the GPU signals that value.**

## Two-queue architecture

The manager maintains two queues:

```
SafeReleaseResource(res, cmdNum)
         │
         ▼
┌─────────────────────────────┐
│         StaleQueue          │  keyed by command list number
│   [cmd=4, res_A]            │
│   [cmd=4, res_B]            │
│   [cmd=5, res_C]            │
└──────────────┬──────────────┘
               │ DiscardStaleResources(cmdNum=4, fenceValue=7)
               │  moves entries with key ≤ cmdNum
               ▼
┌─────────────────────────────┐
│        ReleaseQueue         │  keyed by timeline/fence value
│   [fence=7, res_A]          │
│   [fence=7, res_B]          │
└──────────────┬──────────────┘
               │ Purge(completedFenceValue)
               │  pops entries with key ≤ completedFenceValue
               ▼
          destructor runs ✓
```

**StaleQueue** holds resources that belong to a command list that hasn't been submitted yet. The key is a command list number — a monotonically increasing counter assigned by the [command list allocator](#) (see upcoming post) each time a new list is opened.

**ReleaseQueue** holds resources that have been submitted and are waiting for the GPU to pass a fence checkpoint. The key is a timeline semaphore value (or a D3D12 fence value).

This two-stage design prevents a common mistake: promoting a resource directly to the release queue with the *current* fence value before the command list is even submitted, which would release it too early if the fence advances from a different queue.

## StaleResourceBase — the RAII interface

Every destroyable resource inherits from this:

```cpp
struct StaleResourceBase
{
    std::atomic<int> RefCount { 1 };
    virtual ~StaleResourceBase() = default;
    virtual void Destroy() = 0;

    void Release()
    {
        if (--RefCount == 0)
        {
            Destroy();
            delete this;
        }
    }
};
```

`Destroy` is where the actual API call lives. `Release` decrements the atomic ref count and self-destructs when it hits zero. The ref count matters for multi-queue scenarios: if the same resource is handed to two `ReleaseManager` instances (one per queue), you initialize it with `RefCount = 2` and the destructor only fires after both queues have purged their entry.

A concrete example:

```cpp
struct VkSamplerResource : vk::StaleResourceBase
{
    VkSampler Sampler;
    VkDevice  Device;

    VkSamplerResource(VkSampler s, VkDevice d) : Sampler(s), Device(d) {}

    void Destroy() override { vkDestroySampler(Device, Sampler, nullptr); }
};
```

One struct per resource type, minimal overhead. The virtual dispatch cost is paid exactly once, at the moment the GPU is done — not in the hot path.

## StaleResourceWrapper — the move-only handle

The queues store `StaleResourceWrapper`, a move-only smart pointer over `StaleResourceBase`:

```cpp
class StaleResourceWrapper
{
public:
    static StaleResourceWrapper Create(StaleResourceBase* p, int numRefs = 1)
    {
        p->RefCount.store(numRefs);
        return StaleResourceWrapper(p);
    }

    StaleResourceWrapper(const StaleResourceWrapper& o) : m_Ptr(o.m_Ptr) {}
    StaleResourceWrapper(StaleResourceWrapper&& o) : m_Ptr(o.m_Ptr) { o.m_Ptr = nullptr; }

    StaleResourceWrapper& operator=(const StaleResourceWrapper&) = delete;
    StaleResourceWrapper& operator=(StaleResourceWrapper&&) = delete;

    ~StaleResourceWrapper() { if (m_Ptr) m_Ptr->Release(); }

    void GiveUpOwnership() { m_Ptr = nullptr; }

private:
    StaleResourceWrapper(StaleResourceBase* p) : m_Ptr(p) {}
    StaleResourceBase* m_Ptr = nullptr;
};
```

The copy constructor increments nothing — it shares the raw pointer. The move constructor nulls out the source. The destructor calls `Release`, which decrements the ref count. `GiveUpOwnership` lets you detach the pointer without triggering a release, useful when ownership has been transferred to the queue via `std::move`.

`Create` is the preferred entry point because it stamps the ref count before any copy can observe it, avoiding a race on freshly allocated objects.

## ReleaseManager

```cpp
class ReleaseManager
{
    using Entry = std::pair<uint64_t, StaleResourceWrapper>;

public:
    void SafeReleaseResource(StaleResourceWrapper&& wrapper, uint64_t cmdNum);
    void SafeReleaseResource(const StaleResourceWrapper& wrapper, uint64_t cmdNum);

    void DiscardStaleResources(uint64_t fenceValue);
    void DiscardStaleResources(uint64_t cmdNum, uint64_t fenceValue);

    void Purge(uint64_t fenceValue);
    void Flush();

private:
    std::deque<Entry> m_StaleQueue;
    std::deque<Entry> m_ReleaseQueue;
};
```

`Entry` pairs a `uint64_t` key with the wrapper. `std::deque` gives O(1) push-back and pop-front — exactly the access pattern here.

### SafeReleaseResource

```cpp
void ReleaseManager::SafeReleaseResource(StaleResourceWrapper&& wrapper, uint64_t cmdNum)
{
    m_StaleQueue.emplace_back(cmdNum, std::move(wrapper));
}

void ReleaseManager::SafeReleaseResource(const StaleResourceWrapper& wrapper, uint64_t cmdNum)
{
    m_StaleQueue.emplace_back(cmdNum, wrapper);
}
```

Enqueues the resource into the stale queue. The `cmdNum` is the current command list's number at the point of recording — not at submit time.

The copy overload exists for the multi-queue case: both release managers receive the same wrapper (same raw pointer, bumped ref count), so both hold a separate `StaleResourceWrapper` that will eventually call `Release`.

### DiscardStaleResources

Called at submit time, after the command list is handed to the GPU driver.

```cpp
void ReleaseManager::DiscardStaleResources(uint64_t fenceValue)
{
    while (!m_StaleQueue.empty())
    {
        m_ReleaseQueue.emplace_back(fenceValue, std::move(m_StaleQueue.front().second));
        m_StaleQueue.pop_front();
    }
}

void ReleaseManager::DiscardStaleResources(uint64_t cmdNum, uint64_t fenceValue)
{
    while (!m_StaleQueue.empty() &&
           m_StaleQueue.front().first <= cmdNum)
    {
        m_ReleaseQueue.emplace_back(fenceValue, std::move(m_StaleQueue.front().second));
        m_StaleQueue.pop_front();
    }
}
```

The no-argument overload promotes *all* stale resources — use this when you submit everything in the queue at once (e.g. end of frame, single queue renderer).

The `cmdNum` overload promotes only the resources that belong to command lists up to and including `cmdNum`. This is important for renderers that record multiple command lists per frame and submit them in batches: you only promote resources that are actually covered by the current submission.

`fenceValue` is the timeline value that will be signaled when this submission completes on the GPU.

### Purge

```cpp
void ReleaseManager::Purge(uint64_t fenceValue)
{
    while (!m_ReleaseQueue.empty() &&
           m_ReleaseQueue.front().first <= fenceValue)
    {
        m_ReleaseQueue.pop_front();
    }
}
```

Pop-front destroys the `StaleResourceWrapper`, which calls `Release` on the base, which calls `Destroy` when the ref count hits zero.

Call this once per frame with the latest completed timeline value:

```cpp
uint64_t completed = vkGetSemaphoreCounterValue(device, timelineSemaphore);
releaseManager.Purge(completed);
```

### Flush

```cpp
void ReleaseManager::Flush()
{
    m_StaleQueue.clear();
    m_ReleaseQueue.clear();
}
```

Clears both queues unconditionally. Use this during shutdown after `vkDeviceWaitIdle` / `ID3D12Fence::SetEventOnCompletion` has confirmed the GPU is idle. Everything left in the queues is safe to destroy immediately.

## Usage in a render loop

Putting it together in a typical single-queue forward renderer:

```cpp
void Renderer::RenderFrame()
{
    // Open a new command list — the allocator bumps its internal counter
    CommandList* cmd = m_CmdAllocator.Allocate();
    uint64_t cmdNum  = m_CmdAllocator.CurrentCmdNum();

    // Record work. Somewhere mid-recording you realise a buffer is no longer needed:
    auto* stale = new VkBufferResource(m_OldVertexBuffer, m_Device);
    m_ReleaseManager.SafeReleaseResource(
        StaleResourceWrapper::Create(stale), cmdNum);

    // Submit. The timeline semaphore will be signaled to `signalValue` when done.
    uint64_t signalValue = m_Timeline.NextValue();
    m_GraphicsQueue.Submit(cmd, m_TimelineSemaphore, signalValue);

    // Promote stale resources for this submission into the release queue.
    m_ReleaseManager.DiscardStaleResources(cmdNum, signalValue);

    // Free anything the GPU has already finished with.
    uint64_t completedValue = m_Timeline.CompletedValue(); // queries semaphore counter
    m_ReleaseManager.Purge(completedValue);
}

void Renderer::Shutdown()
{
    vkDeviceWaitIdle(m_Device);
    m_ReleaseManager.Flush();
}
```

## Multi-queue note

When a resource is shared across multiple queues (graphics + compute + transfer), create the wrapper with `numRefs` matching the number of queues and pass a copy to each manager:

```cpp
auto* res = new VkBufferResource(buffer, device);
auto  wrapper = StaleResourceWrapper::Create(res, /*numRefs=*/2);

m_GraphicsReleaseManager.SafeReleaseResource(wrapper,          graphicsCmdNum);
m_ComputeReleaseManager.SafeReleaseResource(std::move(wrapper), computeCmdNum);
```

The resource will only be destroyed after both managers have purged their entry, regardless of which queue finishes first.

## Full listing

```cpp
// ReleaseManager.h
#pragma once

#include <atomic>
#include <deque>
#include <volk/volk.h>

namespace mad::rhi::vk {

struct StaleResourceBase
{
    std::atomic<int> RefCount { 1 };
    virtual ~StaleResourceBase() = default;
    virtual void Destroy() = 0;

    void Release()
    {
        if (--RefCount == 0)
        {
            Destroy();
            delete this;
        }
    }
};

class StaleResourceWrapper
{
public:
    static StaleResourceWrapper Create(StaleResourceBase* p, int numRefs = 1)
    {
        p->RefCount.store(numRefs);
        return StaleResourceWrapper(p);
    }

    StaleResourceWrapper(const StaleResourceWrapper& o) : m_Ptr(o.m_Ptr) {}
    StaleResourceWrapper(StaleResourceWrapper&& o) : m_Ptr(o.m_Ptr) { o.m_Ptr = nullptr; }

    StaleResourceWrapper& operator=(const StaleResourceWrapper&) = delete;
    StaleResourceWrapper& operator=(StaleResourceWrapper&&) = delete;

    ~StaleResourceWrapper() { if (m_Ptr) m_Ptr->Release(); }

    void GiveUpOwnership() { m_Ptr = nullptr; }

private:
    StaleResourceWrapper(StaleResourceBase* p) : m_Ptr(p) {}
    StaleResourceBase* m_Ptr = nullptr;
};

class ReleaseManager
{
    using Entry = std::pair<uint64_t, StaleResourceWrapper>;

public:
    void SafeReleaseResource(StaleResourceWrapper&& wrapper, uint64_t cmdNum);
    void SafeReleaseResource(const StaleResourceWrapper& wrapper, uint64_t cmdNum);

    void DiscardStaleResources(uint64_t fenceValue);
    void DiscardStaleResources(uint64_t cmdNum, uint64_t fenceValue);

    void Purge(uint64_t fenceValue);
    void Flush();

private:
    std::deque<Entry> m_StaleQueue;
    std::deque<Entry> m_ReleaseQueue;
};

} // namespace mad::rhi::vk
```

```cpp
// ReleaseManager.cpp
#include "Mad-RHI/Backend/Vulkan/Vk/ReleaseManager.h"

namespace mad::rhi::vk {

void ReleaseManager::SafeReleaseResource(StaleResourceWrapper&& wrapper, uint64_t cmdNum)
{
    m_StaleQueue.emplace_back(cmdNum, std::move(wrapper));
}

void ReleaseManager::SafeReleaseResource(const StaleResourceWrapper& wrapper, uint64_t cmdNum)
{
    m_StaleQueue.emplace_back(cmdNum, wrapper);
}

void ReleaseManager::DiscardStaleResources(uint64_t fenceValue)
{
    while (!m_StaleQueue.empty())
    {
        m_ReleaseQueue.emplace_back(fenceValue, std::move(m_StaleQueue.front().second));
        m_StaleQueue.pop_front();
    }
}

void ReleaseManager::DiscardStaleResources(uint64_t cmdNum, uint64_t fenceValue)
{
    while (!m_StaleQueue.empty() &&
           m_StaleQueue.front().first <= cmdNum)
    {
        m_ReleaseQueue.emplace_back(fenceValue, std::move(m_StaleQueue.front().second));
        m_StaleQueue.pop_front();
    }
}

void ReleaseManager::Purge(uint64_t fenceValue)
{
    while (!m_ReleaseQueue.empty() &&
           m_ReleaseQueue.front().first <= fenceValue)
    {
        m_ReleaseQueue.pop_front();
    }
}

void ReleaseManager::Flush()
{
    m_StaleQueue.clear();
    m_ReleaseQueue.clear();
}

} // namespace mad::rhi::vk
```