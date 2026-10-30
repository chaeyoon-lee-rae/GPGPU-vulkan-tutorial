# 10. Command buffers

[Previous: Compute pipeline](../09-compute-pipeline/README.md) | [Next: Dispatch and synchronization](../11-dispatch-synchronization/README.md) | [All tutorials](../../README.md#tutorials)

Lesson 09 created the compute pipeline. This lesson binds the pipeline and descriptor set, passes the element count, and records a dispatch command in a command buffer.

Recording commands first and submitting them together avoids making the CPU wait for every GPU operation to finish. If the operation order and resources stay the same, the recording can also be reused. The next lesson changes only the input data and submits the same command buffer again.

Write the code in [main.cpp](main.cpp) in the order below. This lesson ends at recording; queue submission and result verification are added in the next lesson. The [completed code](../99-completed/10-command-buffers.cpp) is available for comparison.

## Store the pool and command buffer

A command pool manages memory for recording commands and allocates command buffers. Keeping this allocation separate allows recording memory from the same pool to be reused or reset together. In the `private` section, add these members after `computePipeline`.

```cpp
vk::raii::CommandPool commandPool = nullptr;
std::vector<vk::raii::CommandBuffer> commandBuffers;
```

Store one command buffer in `commandBuffers[0]`. Members are destroyed in reverse declaration order, so the command buffer is freed before the pool that allocated it.

Add an empty pool creation function at the end of the class's `private` section.

```cpp
bool createCommandPool()
{
}
```

Inside the function, specify the queue family and create the pool.

```cpp
vk::CommandPoolCreateInfo poolInfo{};
poolInfo.queueFamilyIndex = computeQueueFamilyIndex;
commandPool = vk::raii::CommandPool(device, poolInfo); // C API: vkCreateCommandPool
return true;
```

Command buffers allocated from this pool must be submitted to a queue in the specified family. Use the same `computeQueueFamilyIndex` selected in lesson 03 and used to retrieve the queue in lesson 04.

Leave `flags` unset. The commands will be recorded once and resubmitted, so support for individually resetting and recording command buffers again is not needed.

## Allocate a primary command buffer

After the pool creation function, add an empty allocation function.

```cpp
bool allocateCommandBuffer()
{
}
```

Inside it, specify the pool, command buffer level, and count.

```cpp
vk::CommandBufferAllocateInfo allocInfo{};
allocInfo.commandPool = *commandPool;
allocInfo.level = vk::CommandBufferLevel::ePrimary;
allocInfo.commandBufferCount = 1;
```

An `ePrimary` command buffer can be submitted directly to a queue. `eSecondary` command buffers execute from other command buffers, but this example does not use them.

Next, store the allocated RAII object in the member.

```cpp
commandBuffers = device.allocateCommandBuffers(allocInfo); // C API: vkAllocateCommandBuffers
return true;
```

A newly allocated command buffer is in the `initial` state. Starting recording with `begin()` and finishing with `end()` puts it in the `executable` state, ready for submission. See [Command buffer lifecycle](https://docs.vulkan.org/spec/latest/chapters/cmdbuffers.html#commandbuffers-lifecycle).

## Begin recording

In the `private` section, add the shader workgroup size directly below `elementCount`.

```cpp
static constexpr uint32_t workgroupSize = 64;
```

This must match `local_size_x = 64` in [add.comp](add.comp). Changing only the CPU constant does not change the shader's group size.

After the allocation function, add an empty recording function.

```cpp
bool recordCommandBuffer()
{
}
```

Inside the function, first calculate the workgroup count along x. Each group runs 64 invocations, so divide the element count by the group size and round up.

```cpp
const uint32_t groupCount = (elementCount + workgroupSize - 1) / workgroupSize;
```

Compare this with the device limits queried in lesson 03.

```cpp
const auto limits = physicalDevice.getProperties().limits; // C API: vkGetPhysicalDeviceProperties
if (groupCount > limits.maxComputeWorkGroupCount[0])
{
    std::cerr << "Input exceeds device limits.\n";
    return false;
}
```

Do not record if the group count along x exceeds the device limit. This exercise uses a small, fixed-size input.

> [!NOTE]
> **Questions to consider**
>
> How is a subgroup related to a workgroup, and how does it compare with a CUDA warp? Who determines the workgroup size and the subgroup size?

Below that, reference the first command buffer and begin recording.

```cpp
auto& cmd = commandBuffers[0];
vk::CommandBufferBeginInfo beginInfo{};
cmd.begin(beginInfo); // C API: vkBeginCommandBuffer
```

Calls corresponding to `vkCmd*` now add commands to the command buffer. These calls do not execute GPU computation immediately.

Leave `beginInfo.flags` unset as well. With `eOneTimeSubmit`, the recording cannot be submitted again unchanged after completion. The next lesson runs the same recording twice with only the input data changed.

## Bind the pipeline and descriptor set

Below the code that begins recording, bind the compute pipeline.

```cpp
cmd.bindPipeline(vk::PipelineBindPoint::eCompute, *computePipeline); // C API: vkCmdBindPipeline
```

`eCompute` selects the pipeline for compute commands. The dispatch recorded later uses this pipeline's shader. See [Pipeline binding](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdBindPipeline.html).

Next, bind the descriptor set connected to the input and output buffers in lesson 07.

```cpp
cmd.bindDescriptorSets(vk::PipelineBindPoint::eCompute, *pipelineLayout,
                       0, *descriptorSets[0], {}); // C API: vkCmdBindDescriptorSets
```

`pipelineLayout` is the interface created in lesson 09. The following `0` is the **set number** of the first set to bind. It corresponds to the shader's `set = 0`, which is distinct from `binding = 0, 1, 2` within the set. That one set contains `inputA`, `inputB`, and `output`.

The final `{}` is the dynamic offset list. This example uses ordinary storage buffer descriptors, so there are no dynamic offsets to pass. Binding selects the set for a dispatch without copying buffer data. See [Descriptor set binding](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdBindDescriptorSets.html).

## Pass the element count

After binding the descriptor set, record the push constant value.

```cpp
const uint32_t count = elementCount;
cmd.pushConstants<uint32_t>(*pipelineLayout, vk::ShaderStageFlagBits::eCompute,
                                0, count); // C API: vkCmdPushConstants
```

This supplies the actual value for the range declared in lesson 09's layout. It passes 4 bytes, the size of `uint32_t`, starting at offset `0`. The shader reads this value to check whether an element index is valid.

This call records the push constant value in the command buffer. Changing the CPU variable afterward does not change the recorded value. To change the element count, record the commands containing that value again. See [Recording push constants](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdPushConstants.html).

## Record the dispatch

Record the dispatch using the workgroup count calculated at the start of the function.

```cpp
cmd.dispatch(groupCount, 1, 1); // C API: vkCmdDispatch
```

The arguments to `dispatch()` are **workgroup counts**, not invocation counts. Use one group each along y and z. For 65 elements, two groups along x provide 128 invocations in total, and indices 65 and above return at the shader's bounds check. See [Dispatch group counts](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdDispatch.html).

Finally, end recording.

```cpp
cmd.end(); // C API: vkEndCommandBuffer
std::cout << "Command buffer recorded: " << groupCount << " workgroups.\n";
return true;
```

If `end()` succeeds, the recorded commands are ready for submission. Since nothing has been submitted to the queue yet, the output buffer does not contain GPU results.

> [!NOTE]
> **Questions to consider**
>
> - For 130 elements and a workgroup size of 64, how many workgroups and invocations are needed? What happens to the invocations that do not perform addition?
> - Does calling `dispatch()` also start GPU computation? How does this differ from CUDA?

## Connect initialization and execution

Replace the final `&& createComputePipeline();` in `initVulkan()` with the following.

```cpp
    && createComputePipeline()
    && createCommandPool()
    && allocateCommandBuffer();
```

In `run()`, replace the `return true;` after successful initialization with this line.

```cpp
return recordCommandBuffer();
```

The program prepares buffers, the set, and the pipeline, then allocates a command buffer and records commands. The next lesson adds a memory barrier so the CPU can read the results, then submits the command buffer to the queue.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), change directories from the repository root. CMake compiles the shader to SPIR-V in the same way as in lesson 08.

```sh
cd examples/10-command-buffers
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target command_buffers_example
.\build\Debug\main.exe
```

To run the completed example, use the same configuration to build and run it as follows.

```powershell
cmake --build build --config Debug --target command_buffers_completed
.\build\completed\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target command_buffers_example
./build/main
```

To run the completed example, use the same configuration to build and run it as follows.

```sh
cmake --build build --target command_buffers_completed
./build/completed/main
```

### Completion criteria

With the default element count of 8, these messages appear.

```text
Command buffer recorded: 1 workgroups.
Resources destroyed.
```

- Command buffer recording completes without validation errors.
- The printed workgroup count equals `elementCount` divided by `workgroupSize`, rounded up.
- The code shows the order `bindPipeline -> bindDescriptorSets -> pushConstants -> dispatch` and the distinction between recording and submission.

Submit a screenshot of the terminal showing the output.

## References

This lesson was written for the repository's vector addition example. For pool, allocation, and recording rules, see [Vulkan Specification: Command Buffers](https://docs.vulkan.org/spec/latest/chapters/cmdbuffers.html).
