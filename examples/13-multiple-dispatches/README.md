# 13. Multiple dispatches

[Previous: Staging buffers](../12-staging-buffers/README.md) | [All tutorials](../../README.md#tutorials)

Lesson 12 uploaded inputs and brought the result of one dispatch back to the CPU. This lesson keeps the first operation's output on the GPU and connects it as input to the next operation. Use the existing addition pipeline twice to perform these calculations.

```text
Dispatch 1: T = A + B
Dispatch 2: output = T + B
```

The final result is `A + 2B`. Since the next operation also runs on the GPU, the intermediate value `T` does not need to be read back to the CPU and uploaded again. Keeping it on the GPU reduces intermediate transfers and CPU waits. In inference, the same structure connects one operation's output to the next operation's input.

These additions could also be combined into one shader to reduce dispatches and intermediate buffer accesses. This lesson uses two dispatches to examine the buffer connections and synchronization also needed when connecting different operations.

Write the code in [main.cpp](main.cpp) in the order below. The [completed code](../99-completed/13-multiple-dispatches.cpp) is available for comparison.

## Distinguish the pipeline from its input bindings

The pipeline selects the shader to run, while the descriptor set selects the buffers that shader uses. Both operations use the same addition shader, so only one pipeline is needed. Instead, use two sets with different buffer connections for the two dispatches.

| Binding | Shader role | `descriptorSets[0]` | `descriptorSets[1]` |
|---|---|---|---|
| 0 | First input | `inputA` | `intermediate` |
| 1 | Second input | `inputB` | `inputB` |
| 2 | Output | `intermediate` | `output` |

`intermediate` holds the intermediate value `T`. The first dispatch writes this buffer, and the second reads it. Both sets use the same descriptor set layout, so the shader and pipeline layout do not need to change.

Could one set be bound for the first dispatch, updated with different buffers, and then used to record the second dispatch? Descriptor binding does not snapshot the set's contents at each call. Updating the ordinary descriptors used here after binding can invalidate a command buffer that is recording or executable. Fill both sets during initialization and select the set to use when recording. See [Descriptor update rules](https://docs.vulkan.org/refpages/latest/refpages/source/vkUpdateDescriptorSets.html).

## The dependency between two dispatches

Recording one command earlier in the same command buffer does not by itself make its writes available to a later shader. Record this dependency for `intermediate`.

```text
Dispatch 1
    | COMPUTE_SHADER / SHADER_WRITE
Buffer memory barrier for intermediate
    | COMPUTE_SHADER / SHADER_READ
Dispatch 2
```

This is a dependency between GPU operations. The fence waited on by the CPU checks whether the entire submission has finished; it does not replace the barrier between the two dispatches. Both operations only read `inputB`, so that buffer has no additional write dependency between them. See [Khronos compute-to-compute examples](https://docs.vulkan.org/guide/latest/synchronization_examples.html#_compute_to_compute_dependencies).

## Add an intermediate buffer

Declare the intermediate buffer below the `output` member.

```cpp
BufferResource intermediate;
```

In the chain of `return createBuffer(...)` calls in `createStorageBuffers()`, insert this line after creating `output` and before creating `uploadA`.

```cpp
    && createBuffer(bufferSize, vk::BufferUsageFlagBits::eStorageBuffer, deviceMemory, intermediate)
```

`intermediate` is only written and read by shaders. It is not copied to or from the CPU, so neither `eTransferSrc` nor `eTransferDst` is added. Its size is the same `bufferSize` as the other compute buffers, and its requested memory property is `eDeviceLocal`.

## Increase the descriptor pool capacity

Each set uses three storage buffer descriptors. Allocating two sets requires capacity for six descriptors in the pool. Change the existing `poolSize.descriptorCount = 3;` in `createDescriptorPool()`.

```cpp
poolSize.descriptorCount = 6;
```

In the same function, also change `poolInfo.maxSets = 1;`.

```cpp
poolInfo.maxSets = 2;
```

Even though both sets use the same `inputB`, each set needs its own descriptor slot. Buffer count and descriptor count are different. Keep the pool's `eFreeDescriptorSet` flag and the RAII declaration order unchanged.

## Allocate two sets with the same layout

Rewrite the body of `createDescriptorSets()`. Remove the existing body and first prepare an array of two layouts to use.

```cpp
const std::array<vk::DescriptorSetLayout, 2> layouts = {*descriptorSetLayout, *descriptorSetLayout};
```

Both entries refer to the same layout, so bindings 0, 1, and 2 have the same structure in both sets. Fill in the allocation information below it.

```cpp
vk::DescriptorSetAllocateInfo allocInfo{};
allocInfo.descriptorPool = *descriptorPool;
allocInfo.descriptorSetCount = static_cast<uint32_t>(layouts.size());
allocInfo.pSetLayouts = layouts.data();
descriptorSets = device.allocateDescriptorSets(allocInfo); // C API: vkAllocateDescriptorSets
```

Having two sets does not mean that the shader uses `set = 0` and `set = 1` simultaneously. Both sets are candidates for the same `set = 0` slot; each dispatch binds a different one. The pipeline layout's `setLayoutCount` remains 1.

## Connect different buffers to each set

Below the allocation code, collect the buffer handles for each set. The outer array indexes the allocated sets, and the inner array indexes their bindings.

```cpp
const std::array<std::array<vk::Buffer, 3>, 2> buffers = {{
    {*inputA.buffer, *inputB.buffer, *intermediate.buffer},
    {*intermediate.buffer, *inputB.buffer, *output.buffer}
}};
```

Add a loop to update three descriptors for each set.

```cpp
for (uint32_t set = 0; set < buffers.size(); ++set)
{
    std::array<vk::DescriptorBufferInfo, 3> bufferInfos{};
    std::array<vk::WriteDescriptorSet, 3> writes{};
}
```

After the two array declarations inside that loop, add an inner loop to fill the information for each binding.

```cpp
for (uint32_t binding = 0; binding < bufferInfos.size(); ++binding)
{
    bufferInfos[binding].buffer = buffers[set][binding];
    bufferInfos[binding].offset = 0;
    bufferInfos[binding].range = bufferSize;
}
```

In the same inner loop, add the update information immediately after setting `range`.

```cpp
writes[binding].dstSet = *descriptorSets[set];
writes[binding].dstBinding = binding;
writes[binding].descriptorType = vk::DescriptorType::eStorageBuffer;
writes[binding].descriptorCount = 1;
writes[binding].pBufferInfo = &bufferInfos[binding];
```

Immediately after the inner loop, apply the updates while still inside the outer `set` loop.

```cpp
device.updateDescriptorSets(writes, {}); // C API: vkUpdateDescriptorSets
```

`bufferInfos` and `writes` must remain valid until the call returns. The buffers themselves are class members and stay alive until both dispatches have finished. Return success after the outer loop, at the end of the function.

```cpp
return true;
```

This function runs during initialization. Both sets' references are ready before command recording begins.

## Connect writes and reads of the intermediate buffer

`recordCommandBuffer()` already contains lesson 12's input copies and upload barriers. Keep the first `dispatch()` that follows them. Since `descriptorSets[0]` now writes to `intermediate`, this records the first calculation, `T = A + B`.

Add `intermediateBarrier` after the first `dispatch()`. It must precede the existing `downloadBarrier`.

```cpp
vk::BufferMemoryBarrier intermediateBarrier{};
intermediateBarrier.srcAccessMask = vk::AccessFlagBits::eShaderWrite;
intermediateBarrier.dstAccessMask = vk::AccessFlagBits::eShaderRead;
intermediateBarrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
intermediateBarrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
intermediateBarrier.buffer = *intermediate.buffer;
intermediateBarrier.offset = 0;
intermediateBarrier.size = bufferSize;
```

Below it, record the compute-to-compute dependency.

```cpp
cmd.pipelineBarrier(vk::PipelineStageFlagBits::eComputeShader,
                    vk::PipelineStageFlagBits::eComputeShader, {}, {}, intermediateBarrier, {}); // C API: vkCmdPipelineBarrier
```

Even though both stages are compute, the earlier shader write and later shader read are separate accesses. This barrier connects writes to `intermediate` with the later reads. Workgroup synchronization inside a shader cannot replace this dependency between separate dispatches.

> [!NOTE]
> **Questions to consider**
>
> If an earlier operation writes X and a later operation reads it, would the same dependency be needed if the earlier operation instead read X and the later one overwrote it? Assume that the two operations share no resources other than X.

## Bind the second set and dispatch again

After the barrier just recorded, bind the second set and add a dispatch.

```cpp
cmd.bindDescriptorSets(vk::PipelineBindPoint::eCompute, *pipelineLayout,
                       0, *descriptorSets[1], {}); // C API: vkCmdBindDescriptorSets
cmd.dispatch(groupCount, 1, 1); // C API: vkCmdDispatch
```

The binding slot remains 0. The pipeline and push constant element count are unchanged from the first dispatch, and that state is retained, so they do not need to be set again. Only the descriptor set changes: the second set connects `intermediate + inputB -> output`.

The existing `downloadBarrier`, `output -> readback` copy, `hostBarrier`, and `cmd.end()` follow. The complete flow is as follows.

```text
Input uploads -> upload barriers
-> bind descriptorSets[0] -> dispatch 1: intermediate = A + B
-> intermediate buffer barrier
-> bind descriptorSets[1] -> dispatch 2: output = intermediate + B
-> download barrier -> copy output -> host barrier
-> submit once -> wait for fence -> verify on CPU
```

Update the recording message to match the actual dispatch count. Replace the single `std::cout` statement after `cmd.end()` in `recordCommandBuffer()`.

```cpp
std::cout << "Command buffer recorded: " << groupCount << " workgroups per dispatch, 2 dispatches.\n";
```

## Update the CPU reference value

Change the `expected` expression in `verifyOutput()` as follows.

```cpp
const float expected = (a + b) + b;
```

This is mathematically equal to `a + 2 * b`, but the CPU performs the additions in the same order as the two GPU operations. Floating-point rounding can differ when operation order changes, so keep the existing tolerance comparison as well.

Keep `run()` unchanged. Each round changes the inputs, submits the command buffer containing both dispatches once, and waits for the fence. After the previous round's fence wait finishes, the buffers and command buffer can be reused. The first dispatch overwrites the entire valid range of `intermediate` each round, so CPU initialization and readback of that buffer are unnecessary.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), change from the repository root to the example directory.

```sh
cd examples/13-multiple-dispatches
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target multiple_dispatches_example
.\build\Debug\main.exe
```

Build and run the completed example with these commands.

```powershell
cmake --build build --config Debug --target multiple_dispatches_completed
.\build\completed\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target multiple_dispatches_example
./build/main
```

Build and run the completed example with these commands.

```sh
cmake --build build --target multiple_dispatches_completed
./build/completed/main
```

### Expected output

Both runs with different inputs must match the CPU's `A + 2B` results. Check that the program exits without validation errors. If the result is `A + B`, first check the second dispatch's descriptor set and output buffer connection.

The command buffer must contain a write-to-read barrier for `intermediate` between the two dispatches. Also check that only the final `output` is copied back, without CPU readback of the intermediate buffer, and that both operations complete with one submission and one fence wait.

Run with [lesson 11's synchronization validation settings](../11-dispatch-synchronization/README.md#check-for-synchronization-errors). Correct results by chance do not prove that the read and write dependencies between dispatches are valid. If a validation message contains `SYNC-HAZARD-READ-AFTER-WRITE`, check the target buffer, stages, and access masks of `intermediateBarrier`.

With the default `elementCount = 8`, verification prints the following.

```text
Command buffer recorded: 1 workgroups per dispatch, 2 dispatches.
Round 1: verified 8 elements; first=20, last=41
Round 2: verified 8 elements; first=23, last=44
Resources destroyed.
```

After completing the code for this lesson, submit a screenshot of the terminal showing the output.
