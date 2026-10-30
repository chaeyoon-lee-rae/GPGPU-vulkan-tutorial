# 11. Dispatch and synchronization

[Previous: Command buffers](../10-command-buffers/README.md) | [Next: Staging buffers](../12-staging-buffers/README.md) | [All tutorials](../../README.md#tutorials)

Submit the command buffer recorded in lesson 10 to the compute queue, then read the output buffer on the CPU after GPU computation finishes. Compare the results with `A[i] + B[i]`, change the inputs, and execute the same commands once more.

Write the code in [main.cpp](main.cpp) in the order below. The [completed code](../99-completed/11-dispatch-synchronization.cpp) is available for comparison.

## What is needed before reading the results

`dispatch()` records a command in a command buffer. `submit()` passes that recording to a queue but does not wait for GPU computation to finish. Connect the following conditions so the CPU can read the results safely.

| Mechanism | Role in this example |
|---|---|
| Memory barrier | Makes compute shader writes visible to subsequent host reads |
| Fence wait | Makes the CPU wait for the queue submission to complete |
| `HOST_COHERENT` memory | Removes the need for explicit CPU cache flush and invalidate calls |

GPU work completing and its writes becoming visible to the CPU are separate conditions. Use a fence to confirm completion and a memory barrier to connect the writes to host reads.

Lesson 04 did not enable the `synchronization2` feature, so this lesson uses APIs corresponding to `vkCmdPipelineBarrier` and `vkQueueSubmit`.

## Record a barrier for host reads

In `recordCommandBuffer()`, add a barrier after `dispatch()` and before `end()`. First, specify the earlier and later access types.

```cpp
vk::BufferMemoryBarrier hostBarrier{};
hostBarrier.srcAccessMask = vk::AccessFlagBits::eShaderWrite;
hostBarrier.dstAccessMask = vk::AccessFlagBits::eHostRead;
```

> [!NOTE]
> **Questions to consider**
>
> What happens if `dstAccessMask` specifies only `eHostRead`, but the CPU writes or a shader reads afterward?

The earlier access is the shader writing to `output`, and the later access is the CPU reading those results. Next, specify the buffer region covered by the barrier.

```cpp
hostBarrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
hostBarrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
hostBarrier.buffer = *output.buffer;
hostBarrier.offset = 0;
hostBarrier.size = bufferSize;
```

The barrier covers all of `output`. No buffer ownership transfer between queue families is needed, so set both family indices to `VK_QUEUE_FAMILY_IGNORED`.

Next, record the barrier in the command buffer.

```cpp
cmd.pipelineBarrier(vk::PipelineStageFlagBits::eComputeShader,
                    vk::PipelineStageFlagBits::eHost, {}, {}, hostBarrier, {}); // C API: vkCmdPipelineBarrier
```

The first two arguments specify the **stages** to synchronize, while the access masks set earlier specify the **memory access types** within those stages. This connects `ComputeShader / ShaderWrite -> Host / HostRead`. The empty arguments specify no dependency flags, global memory barriers, or image barriers.

The barrier call does not make the CPU wait. Use a fence for the CPU to confirm that execution has reached and completed the recorded barrier. See [Buffer memory barrier access scopes](https://docs.vulkan.org/spec/latest/chapters/synchronization.html#synchronization-buffer-memory-barrier).

> [!NOTE]
> **Questions to consider**
>
> - What do pipeline stages and access masks each specify?
> - What roles do waiting for a fence and adding a `ShaderWrite -> HostRead` barrier each have in this example?

## Create a fence

A fence lets the host check completion of queue work. It is used here to wait for the submission whose results will be read, instead of waiting for every queue on the device with `device.waitIdle()`. In the `private` section, add a member after `commandBuffers`.

```cpp
vk::raii::Fence fence = nullptr;
```

Add the creation function at the end of the class's `private` section.

```cpp
bool createFence()
{
    fence = vk::raii::Fence(device, vk::FenceCreateInfo{}); // C API: vkCreateFence
    return true;
}
```

Leaving `flags` unset creates the fence in the `unsignaled` state. Pass this fence with the queue submission; it becomes `signaled` when the submitted work completes. This example does not wait before the first submission, so `eSignaled` is not needed at creation. See [Fence creation information](https://docs.vulkan.org/refpages/latest/refpages/source/VkFenceCreateInfo.html).

Replace the final `&& allocateCommandBuffer();` in `initVulkan()` with the following.

```cpp
    && allocateCommandBuffer()
    && createFence();
```

## Separate buffer creation from input writes

The program will run twice with different inputs, so change `createStorageBuffers()` to allocate buffers only. Remove the value array initialization and `writeAndVerifyBuffer()` calls, leaving the function as follows.

```cpp
bool createStorageBuffers()
{
    const std::array buffers = {&inputA, &inputB, &output};
    for (auto* buffer : buffers)
    {
        if (!createBuffer(bufferSize, *buffer))
        {
            return false;
        }
    }
    return true;
}
```

The buffers use the same `HOST_VISIBLE | HOST_COHERENT` memory type as before. Keep the buffers and descriptors, and change only the memory contents before each execution.

Next, replace the entire `writeAndVerifyBuffer()` function with `writeBuffer()` below. This function only writes data; `verifyOutput()`, added later, checks the GPU results.

```cpp
bool writeBuffer(BufferResource& resource, const std::array<float, elementCount>& values)
{
    void* mapped = resource.memory.mapMemory(0, bufferSize); // C API: vkMapMemory
    std::memcpy(mapped, values.data(), static_cast<size_t>(bufferSize));
    resource.memory.unmapMemory(); // C API: vkUnmapMemory
    return true;
}
```

Add the headers needed to check results and generate initial values.

```cpp
#include <cmath>
#include <limits>
```

Add `writeInputs()` at the end of the class. Call it with `round = 0` for the first run and `round = 1` for the second.

```cpp
bool writeInputs(uint32_t round)
{
}
```

Inside the function, prepare two input arrays and initial output values.

```cpp
std::array<float, elementCount> valuesA{};
std::array<float, elementCount> valuesB{};
std::array<float, elementCount> initialOutput{};
initialOutput.fill(std::numeric_limits<float>::quiet_NaN());
for (uint32_t i = 0; i < elementCount; ++i)
{
    valuesA[i] = static_cast<float>(i + round);
    valuesB[i] = 10.0f + static_cast<float>(i + round);
}
```

Filling the output with NaN makes unwritten elements fail verification. It also prevents a correct result from the previous run from making a failed rerun appear successful. NaN does not equal itself, so the previous array equality comparison cannot be reused unchanged.

Next, write the three arrays to their respective buffers.

```cpp
return writeBuffer(inputA, valuesA)
    && writeBuffer(inputB, valuesB)
    && writeBuffer(output, initialOutput);
```

When host writes finish before queue submission, the submission's memory dependency makes those writes available to subsequent device accesses. `HOST_COHERENT` memory allows `flushMappedMemoryRanges()` to be omitted. If the GPU is still using the same buffer from a previous run, that work must finish first. The execution order below ensures this. See [Host Write Ordering Guarantees](https://docs.vulkan.org/spec/latest/chapters/synchronization.html#synchronization-submission-host-writes).

## Submit and wait for completion

Add a function at the end of the class to submit commands and wait.

```cpp
bool submitAndWait()
{
}
```

Inside the function, retrieve the command buffer handle and connect it to the submission information.

```cpp
const vk::CommandBuffer commandBuffer = *commandBuffers[0];
vk::SubmitInfo submitInfo{};
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffer;
computeQueue.submit(submitInfo, *fence); // C API: vkQueueSubmit
```

This call submits the commands recorded in lesson 10 to the queue. Only one queue is used, and the CPU waits for completion, so semaphores for synchronization between queues are not needed. The submission's fence must be `unsignaled`. See [Queue submission rules](https://docs.vulkan.org/refpages/latest/refpages/source/vkQueueSubmit.html).

Immediately afterward, wait for the fence to become signaled.

```cpp
const auto result = device.waitForFences(*fence, VK_TRUE, UINT64_MAX); // C API: vkWaitForFences
if (result != vk::Result::eSuccess)
{
    device.waitIdle(); // C API: vkDeviceWaitIdle
    std::cerr << "Fence wait did not complete successfully.\n";
    return false;
}
return true;
```

`VK_TRUE` waits for all supplied fences; this example passes only one. `UINT64_MAX` requests an indefinite wait. Read results or reuse buffers only after success is confirmed. The `waitIdle()` in the unsuccessful fence-wait path prevents immediate destruction of resources that may still be in use. Normal execution uses the fence to confirm completion of this submission. See [Fence wait results](https://docs.vulkan.org/refpages/latest/refpages/source/vkWaitForFences.html).

> [!NOTE]
> **Questions to consider**
>
> With fences, pipeline barriers, and semaphores, who waits for what? In particular, how do barriers and semaphores differ?

## Compare with CPU reference values

Add a result verification function at the end of the class.

```cpp
bool verifyOutput(uint32_t round)
{
}
```

At the start of the function, read the output memory into a CPU array.

```cpp
std::array<float, elementCount> results{};
void* mapped = output.memory.mapMemory(0, bufferSize); // C API: vkMapMemory
std::memcpy(results.data(), mapped, static_cast<size_t>(bufferSize));
output.memory.unmapMemory(); // C API: vkUnmapMemory
```

Call this function only after `submitAndWait()` succeeds. `mapMemory()` itself does not wait for GPU work. Because the memory is `HOST_COHERENT`, `invalidateMappedMemoryRanges()` can be omitted before reading.

Next, compare each element with a reference value calculated on the CPU.

```cpp
for (uint32_t i = 0; i < elementCount; ++i)
{
    const float a = static_cast<float>(i + round);
    const float b = 10.0f + static_cast<float>(i + round);
    const float expected = a + b;
    const float tolerance = 1e-5f * (1.0f + std::abs(expected));
    if (!std::isfinite(results[i]) || std::abs(results[i] - expected) > tolerance)
    {
        std::cerr << "Mismatch at " << i << ": expected " << expected
                  << ", got " << results[i] << '\n';
        return false;
    }
}
std::cout << "Round " << round + 1 << ": verified " << elementCount
          << " elements; first=" << results.front() << ", last=" << results.back() << '\n';
return true;
```

Checking only whether the difference exceeds the tolerance can miss NaN, so first use `std::isfinite()` to check for a finite value. The validation layer does not know the intended formula. CPU reference values detect incorrect results caused by mistakes such as wrong indices or input bindings.

## Run the same recording twice

Replace `run()` with the following.

```cpp
bool run()
{
    if (!initVulkan() || !recordCommandBuffer())
    {
        return false;
    }
    for (uint32_t round = 0; round < 2; ++round)
    {
        if (round != 0)
        {
            device.resetFences(*fence); // C API: vkResetFences
        }
        if (!writeInputs(round) || !submitAndWait() || !verifyOutput(round))
        {
            return false;
        }
    }
    return true;
}
```

The first run uses the newly created fence. Before the second run, the previous iteration has confirmed fence completion, so reset it to `unsignaled` and submit again. Do not reset a fence associated with an unfinished submission. See [Fence reset rules](https://docs.vulkan.org/refpages/latest/refpages/source/vkResetFences.html).

The command buffer changes state as follows.

```text
Recording complete: executable
    -> submit: pending
    -> fence wait succeeds: executable
    -> submit the same recording again: pending
    -> fence wait succeeds: executable
```

Lesson 10 did not use `eOneTimeSubmit`, so the completed recording can be submitted again. The pipeline, descriptors, element count, and dispatch group count are unchanged here. Only buffer contents change, so descriptor updates and command recording are unnecessary. See [Command buffer resubmission](https://docs.vulkan.org/spec/latest/chapters/cmdbuffers.html#commandbuffers-lifecycle).

The CPU must not write to buffers that the GPU is using, and a pending command buffer must not be reset or freed. This loop waits for completion before writing the next inputs and also waits for the last submission before returning from `run()`.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), change directories from the repository root.

```sh
cd examples/11-dispatch-synchronization
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target dispatch_synchronization_example
.\build\Debug\main.exe
```

To run the completed example, use the same configuration to build and run it as follows.

```powershell
cmake --build build --config Debug --target dispatch_synchronization_completed
.\build\completed\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target dispatch_synchronization_example
./build/main
```

To run the completed example, use the same configuration to build and run it as follows.

```sh
cmake --build build --target dispatch_synchronization_completed
./build/completed/main
```

### Completion criteria

With the default element count of 8, these messages appear.

```text
Round 1: verified 8 elements; first=10, last=24
Round 2: verified 8 elements; first=12, last=26
Resources destroyed.
```

- Every element in both runs matches the CPU reference within the tolerance.
- The second run with changed inputs also succeeds without validation errors.

Rebuild with `elementCount` set to 1, 64, and 65 to check an input smaller than one group, an input of exactly one group, and an input with extra invocations in the last group. Both result checks must pass for each element count.

Submit a screenshot of the terminal showing the output. The next lesson separates memory for CPU transfers from memory for GPU computation and adds copy commands and transfer-stage synchronization.

## Check for synchronization errors

General validation and synchronization validation in `VK_LAYER_KHRONOS_validation` are separate settings. Enable synchronization validation as shown below to check for incorrect read and write dependencies between commands.

In Windows PowerShell, set the environment variable before running the program.

```powershell
$env:VK_VALIDATION_VALIDATE_SYNC = '1'
.\build\Debug\main.exe
```

On Linux, pass the setting to the process being run.

```sh
VK_VALIDATION_VALIDATE_SYNC=1 ./build/main
```

Correct output values can still hide access-order errors, so check both CPU reference results and synchronization validation. Synchronization validation does not detect every host access issue or race within a shader. See [Khronos Synchronization Validation](https://vulkan.lunarg.com/doc/view/latest/linux/synchronization_usage.html).

## References

This lesson was written for the repository's vector addition example. For an example using the same barrier API, see [Khronos Synchronization Examples: Legacy synchronization APIs](https://github.com/KhronosGroup/Vulkan-Docs/wiki/Synchronization-Examples-%28Legacy-synchronization-APIs%29#cpu-read-back-of-data-written-by-a-compute-shader).
