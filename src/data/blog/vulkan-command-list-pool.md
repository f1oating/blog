---
author: Dmytro Toronchenko
pubDatetime: 2026-05-15T10:00:00Z
title: "Command List Pool: Recycling Vulkan Command Buffers Without Stalling"
slug: vulkan-command-list-pool
featured: true
draft: false
tags:
  - vulkan
  - cpp
  - graphics
  - rhi
description: How to build a two-queue command buffer pool that recycles in-flight buffers safely using timeline semaphore values, avoiding both stalls and data races.
---

In a pipelined Vulkan renderer the CPU records commands into a `VkCommandBuffer` while the GPU might still be executing the previous frame's buffer. Reusing the same buffer for the next frame without waiting for the GPU to finish is a data race. The naive fix is `vkQueueWaitIdle` — which stalls the entire pipeline.

The right answer is a pool: keep a set of command buffers in circulation, track which ones the GPU is still consuming, and return them to the free pool only once the GPU has signaled past their submission point.

## The problem

Vulkan command buffers have no built-in CPU–GPU lifetime tracking. The API lets you call `vkResetCommandBuffer` the moment you want, regardless of whether the GPU has finished reading it. It's your responsibility to know when that's safe.

A pool solves this by splitting buffers into two groups:

- **Free (acquire queue):** reset, ready to record.
- **In-flight (release queue):** submitted to the GPU, waiting for a timeline value.

When the GPU signals a timeline semaphore value, every buffer submitted at or before that point can be safely reset and moved back to the free group.

## Two-queue design

```
AcquireCommandBuffer()
         │
         ▼
┌────────────────────────────┐
│       AcquireQueue         │  free, reset command buffers
│   [cb_A]                   │
│   [cb_B]                   │ ◄── Purge() recycles completed buffers
└────────────┬───────────────┘
             │ returns cb to caller
             │ (caller records, submits)
             ▼
ReleaseCommandBuffer(cb, fenceValue)
             │
             ▼
┌────────────────────────────┐
│       ReleaseQueue         │  in-flight command buffers
│   [cb_A, fence=7]          │
│   [cb_C, fence=9]          │
└────────────┬───────────────┘
             │ Purge(completedFenceValue)
             │  pops entries where FenceValue ≤ completedFenceValue
             ▼
      back to AcquireQueue ↑
```

This is the same two-queue pattern used in the [release manager](./gpu-resource-release-manager) — it appears throughout Vulkan and D3D12 code whenever deferred cleanup needs to be tied to a timeline value.

## The class

```cpp
class CommandListPool
{
    struct Entry
    {
        VkCommandBuffer CommandBuffer;
        uint64_t        FenceValue;
    };

public:
    void Init(VkDevice device, uint32_t queueIndex);
    void Shutdown();

    VkCommandBuffer AcquireCommandBuffer();
    void ReleaseCommandBuffer(VkCommandBuffer cb, uint64_t fenceValue);

    void Purge(uint64_t fenceValue);

private:
    VkDevice    m_Device      = nullptr;
    VkCommandPool m_CommandPool = nullptr;

    std::deque<VkCommandBuffer> m_AcquireQueue;
    std::deque<Entry>           m_ReleaseQueue;
};
```

`m_AcquireQueue` holds free buffers — handles only, no timeline association. `m_ReleaseQueue` pairs each in-flight buffer with the fence value that marks when it's safe to recycle.

## Init

```cpp
void CommandListPool::Init(VkDevice device, uint32_t queueIndex)
{
    m_Device = device;

    VkCommandPoolCreateInfo cpci{};
    cpci.sType            = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
    cpci.queueFamilyIndex = queueIndex;
    cpci.flags            = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;
    vkCreateCommandPool(device, &cpci, nullptr, &m_CommandPool);
}
```

`VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT` is mandatory here. Without it, individual buffers cannot be reset via `vkResetCommandBuffer` — only the entire pool can be reset at once via `vkResetCommandPool`. Since we recycle individual buffers independently, the flag is required.

## Shutdown

```cpp
void CommandListPool::Shutdown()
{
    if (m_CommandPool != nullptr)
        vkDestroyCommandPool(m_Device, m_CommandPool, nullptr);
}
```

`vkDestroyCommandPool` implicitly frees all command buffers allocated from it. No need to free them individually first — and no need to drain the queues manually, as long as you've already confirmed the GPU is idle (e.g. via `vkDeviceWaitIdle`).

## AcquireCommandBuffer

```cpp
VkCommandBuffer CommandListPool::AcquireCommandBuffer()
{
    if (!m_AcquireQueue.empty())
    {
        VkCommandBuffer cb = m_AcquireQueue.front();
        m_AcquireQueue.pop_front();
        vkResetCommandBuffer(cb, 0);
        return cb;
    }

    VkCommandBufferAllocateInfo cbai{};
    cbai.sType              = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    cbai.commandPool        = m_CommandPool;
    cbai.level              = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    cbai.commandBufferCount = 1;

    VkCommandBuffer cb = nullptr;
    vkAllocateCommandBuffers(m_Device, &cbai, &cb);
    return cb;
}
```

If the acquire queue has a free buffer, reset it and hand it back. Resetting here rather than in `Purge` means the reset cost is paid lazily — only when the buffer is actually needed, not when it's recycled.

If the queue is empty, allocate a new buffer from the pool. Over time the pool reaches a steady state where allocations stop entirely and every frame just rotates existing buffers through the two queues.

## ReleaseCommandBuffer

```cpp
void CommandListPool::ReleaseCommandBuffer(VkCommandBuffer cb, uint64_t fenceValue)
{
    m_ReleaseQueue.push_back({ cb, fenceValue });
}
```

Called immediately after submit. `fenceValue` is the timeline semaphore value that will be signaled when the GPU finishes executing this buffer's commands.

## Purge

```cpp
void CommandListPool::Purge(uint64_t fenceValue)
{
    while (!m_ReleaseQueue.empty())
    {
        auto& entry = m_ReleaseQueue.front();
        if (entry.FenceValue > fenceValue) break;
        m_AcquireQueue.push_back(entry.CommandBuffer);
        m_ReleaseQueue.pop_front();
    }
}
```

The release queue is ordered by fence value (entries are always appended after submit, which is monotonically increasing), so the loop can stop at the first entry that isn't ready yet.

Note that `Purge` moves the buffer handle into the acquire queue without resetting it. The reset happens in `AcquireCommandBuffer` when the buffer is actually reused. This keeps `Purge` cheap — it's called every frame, while a buffer might sit in the acquire queue for several frames before being reused.

## Usage

```cpp
// Initialization
pool.Init(device, graphicsQueueFamilyIndex);

// Per-frame render loop
void RenderFrame()
{
    uint64_t completedValue = QueryTimelineSemaphoreValue(m_TimelineSemaphore);

    // Recycle command buffers the GPU is done with
    pool.Purge(completedValue);

    // Acquire a free, reset command buffer
    VkCommandBuffer cmd = pool.AcquireCommandBuffer();

    VkCommandBufferBeginInfo bi{};
    bi.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    bi.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;
    vkBeginCommandBuffer(cmd, &bi);

    // ... record commands ...

    vkEndCommandBuffer(cmd);

    uint64_t signalValue = m_Timeline.NextValue();

    VkCommandBufferSubmitInfo cmdInfo{};
    cmdInfo.sType         = VK_STRUCTURE_TYPE_COMMAND_BUFFER_SUBMIT_INFO;
    cmdInfo.commandBuffer = cmd;

    VkSemaphoreSubmitInfo signalInfo{};
    signalInfo.sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO;
    signalInfo.semaphore = m_TimelineSemaphore;
    signalInfo.value     = signalValue;
    signalInfo.stageMask = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT;

    VkSubmitInfo2 submit{};
    submit.sType                    = VK_STRUCTURE_TYPE_SUBMIT_INFO_2;
    submit.commandBufferInfoCount   = 1;
    submit.pCommandBufferInfos      = &cmdInfo;
    submit.signalSemaphoreInfoCount = 1;
    submit.pSignalSemaphoreInfos    = &signalInfo;

    vkQueueSubmit2(m_GraphicsQueue, 1, &submit, VK_NULL_HANDLE);

    // Return the buffer — safe to reuse once the GPU reaches signalValue
    pool.ReleaseCommandBuffer(cmd, signalValue);
}

// Shutdown
vkDeviceWaitIdle(device);
pool.Shutdown();
```

## Full listing

```cpp
// CommandListPool.h
#pragma once

#include <cstdint>
#include <deque>
#include <volk/volk.h>

namespace mad::rhi::vk {

class CommandListPool
{
    struct Entry
    {
        VkCommandBuffer CommandBuffer;
        uint64_t        FenceValue;
    };

public:
    void Init(VkDevice device, uint32_t queueIndex);
    void Shutdown();

    VkCommandBuffer AcquireCommandBuffer();
    void ReleaseCommandBuffer(VkCommandBuffer cb, uint64_t fenceValue);

    void Purge(uint64_t fenceValue);

private:
    VkDevice      m_Device      = nullptr;
    VkCommandPool m_CommandPool = nullptr;

    std::deque<VkCommandBuffer> m_AcquireQueue;
    std::deque<Entry>           m_ReleaseQueue;
};

} // namespace mad::rhi::vk
```

```cpp
// CommandListPool.cpp
#include "Mad-RHI/Backend/Vulkan/Vk/CommandListPool.h"

namespace mad::rhi::vk {

void CommandListPool::Init(VkDevice device, uint32_t queueIndex)
{
    m_Device = device;

    VkCommandPoolCreateInfo cpci{};
    cpci.sType            = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
    cpci.queueFamilyIndex = queueIndex;
    cpci.flags            = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;
    vkCreateCommandPool(device, &cpci, nullptr, &m_CommandPool);
}

void CommandListPool::Shutdown()
{
    if (m_CommandPool != nullptr)
        vkDestroyCommandPool(m_Device, m_CommandPool, nullptr);
}

VkCommandBuffer CommandListPool::AcquireCommandBuffer()
{
    if (!m_AcquireQueue.empty())
    {
        VkCommandBuffer cb = m_AcquireQueue.front();
        m_AcquireQueue.pop_front();
        vkResetCommandBuffer(cb, 0);
        return cb;
    }

    VkCommandBufferAllocateInfo cbai{};
    cbai.sType              = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    cbai.commandPool        = m_CommandPool;
    cbai.level              = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    cbai.commandBufferCount = 1;

    VkCommandBuffer cb = nullptr;
    vkAllocateCommandBuffers(m_Device, &cbai, &cb);
    return cb;
}

void CommandListPool::ReleaseCommandBuffer(VkCommandBuffer cb, uint64_t fenceValue)
{
    m_ReleaseQueue.push_back({ cb, fenceValue });
}

void CommandListPool::Purge(uint64_t fenceValue)
{
    while (!m_ReleaseQueue.empty())
    {
        auto& entry = m_ReleaseQueue.front();
        if (entry.FenceValue > fenceValue) break;
        m_AcquireQueue.push_back(entry.CommandBuffer);
        m_ReleaseQueue.pop_front();
    }
}

} // namespace mad::rhi::vk
```
