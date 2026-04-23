---
author: Dmytro Toronchenko
pubDatetime: 2026-04-20T10:00:00Z
title: Understanding Vulkan Render Passes
slug: vulkan-render-passes
featured: true
draft: false
tags:
  - vulkan
  - graphics
  - cpp
description: A deep-dive into how Vulkan render passes work, why they exist, and how to structure them for real-world rendering workloads.
---

Vulkan's render pass system is one of the most explicit parts of the API — and one of the most misunderstood. Unlike OpenGL where the driver figures out load/store operations on your behalf, Vulkan forces you to declare upfront what you intend to do with each attachment. This pays off in tile-based GPU architectures (think mobile and Apple Silicon), where the driver can keep the framebuffer in on-chip memory for the entire pass without ever touching main memory.

## What is a render pass?

A render pass describes the structure of rendering in terms of *attachments* and *subpasses*. An attachment is a buffer that will be read or written — your color buffer, depth buffer, or an intermediate G-buffer target. A subpass is a single rendering phase within the pass that reads from and writes to those attachments.

```cpp
VkAttachmentDescription colorAttachment{};
colorAttachment.format         = swapchainFormat;
colorAttachment.samples        = VK_SAMPLE_COUNT_1_BIT;
colorAttachment.loadOp         = VK_ATTACHMENT_LOAD_OP_CLEAR;
colorAttachment.storeOp        = VK_ATTACHMENT_STORE_OP_STORE;
colorAttachment.initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
colorAttachment.finalLayout    = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
```

`loadOp` and `storeOp` are the key levers. Setting `loadOp = CLEAR` tells the driver it can initialize the attachment with a clear value — no need to load prior contents from memory. On a tile-based GPU this avoids a round-trip to DRAM entirely. `storeOp = DONT_CARE` on a depth attachment after shading is done is another common optimization: the depth data is only ever needed within the pass, so there's no point writing it back.

## Subpass dependencies

The trickier part is expressing ordering between subpasses and between the render pass and surrounding commands. This is done with `VkSubpassDependency`.

```cpp
VkSubpassDependency dependency{};
dependency.srcSubpass    = VK_SUBPASS_EXTERNAL;
dependency.dstSubpass    = 0;
dependency.srcStageMask  = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.srcAccessMask = 0;
dependency.dstStageMask  = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
```

`VK_SUBPASS_EXTERNAL` refers to work happening outside the render pass — in this case, the presentation engine finishing with the image before we start writing to it. Without this dependency the validation layers will complain and you risk a race condition on the attachment.

## Deferred shading with multiple subpasses

Where subpasses really shine is deferred shading. The G-buffer geometry pass writes to multiple color attachments (albedo, normals, material IDs). The lighting pass reads those same attachments as *input attachments* — a special Vulkan mechanism that lets fragment shaders read from the current framebuffer pixel without an image load, staying entirely on-chip.

```cpp
// Lighting subpass reads from G-buffer attachments
VkAttachmentReference inputRef{};
inputRef.attachment = 1; // G-buffer normal attachment index
inputRef.layout     = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;

subpassDesc.inputAttachmentCount = 1;
subpassDesc.pInputAttachments    = &inputRef;
```

In GLSL the lighting shader accesses it via `subpassInput`:

```glsl
layout(input_attachment_index = 0, set = 0, binding = 0)
uniform subpassInput inNormals;

void main() {
    vec3 N = subpassLoad(inNormals).rgb * 2.0 - 1.0;
    // ... lighting calculation
}
```

This pattern is extremely efficient on Mali and PowerVR GPUs. On desktop (NVIDIA, AMD) the gains are smaller but the code structure is still cleaner than ping-ponging between two render passes with explicit image barriers.

## Dynamic rendering

Vulkan 1.3 introduced `VK_KHR_dynamic_rendering`, which lets you skip render pass objects entirely for most cases. Instead of creating a `VkRenderPass` and `VkFramebuffer` you just call `vkCmdBeginRendering` with inline attachment info. It's simpler and perfectly fine for forward rendering pipelines.

For deferred shading on tile-based hardware you still want explicit render passes to get the on-chip input attachment benefit — the dynamic rendering extension doesn't expose subpass input attachments. So the old API isn't going anywhere.

## Takeaways

- Use `LOAD_OP_CLEAR` or `LOAD_OP_DONT_CARE` whenever you can — never `LOAD_OP_LOAD` unless you actually need prior contents.
- `STORE_OP_DONT_CARE` on depth after a pass saves bandwidth on every GPU.
- Subpass input attachments are the right tool for deferred shading on tile-based GPUs.
- `VK_KHR_dynamic_rendering` is worth using for forward passes; stick with render pass objects when you need subpass inputs.
