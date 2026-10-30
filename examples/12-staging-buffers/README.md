# 12. Staging buffers

[Previous: Dispatch and synchronization](../11-dispatch-synchronization/README.md) | [Next: Multiple dispatches](../13-multiple-dispatches/README.md) | [All tutorials](../../README.md#tutorials)

In lesson 11, the compute shader used the same storage buffers that the CPU mapped. This lesson separates the buffers used for CPU transfers from those used for GPU computation. Keep the shader's `A + B` calculation and add input uploads and result downloads to the command buffer.

Write the code in [main.cpp](main.cpp) in the order below. The [completed code](../99-completed/12-staging-buffers.cpp) is available for comparison.

## Separate transfer buffers from compute buffers

Memory that the CPU can map is not always the best memory for GPU computation. On a GPU with separate VRAM, moving input to device memory once can reduce repeated host-memory access during computation. A staging buffer temporarily holds CPU-written values for this transfer. Results move to a readback buffer that the CPU can read.

```text
CPU -> uploadA --copy--> inputA --+
                                +-- dispatch: A + B -> output --copy--> readback -> CPU
CPU -> uploadB --copy--> inputB --+
```

The buffer usages and required memory properties are as follows.

| Buffer | `usage` | Requested memory properties | CPU access |
|---|---|---|---|
| `uploadA`, `uploadB` | `eTransferSrc` | `eHostVisible \| eHostCoherent` | Write inputs |
| `inputA`, `inputB` | `eStorageBuffer \| eTransferDst` | `eDeviceLocal` | No direct access |
| `output` | `eStorageBuffer \| eTransferSrc` | `eDeviceLocal` | No direct access |
| `readback` | `eTransferDst` | `eHostVisible \| eHostCoherent` | Initialize and read results |

`usage` specifies which commands may use the buffer, while memory properties determine where to allocate it. A copy source requires `eTransferSrc`, and a copy destination requires `eTransferDst`. Only the compute buffers `inputA`, `inputB`, and `output` are connected to descriptors. Transfer buffers do not need storage buffer descriptors. See [Copy command requirements](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdCopyBuffer.html).

`eDeviceLocal` identifies memory suitable for device access; `eHostVisible` determines whether the CPU can map it. A type may have both properties. This example does not request `eHostVisible` for compute memory, so it does not map that memory directly. See [Memory property definitions](https://docs.vulkan.org/refpages/latest/refpages/source/VkMemoryPropertyFlagBits.html).

Staging helps only if the savings from subsequent GPU accesses exceed the copy cost. Direct access may be more suitable for a single addition of a small array, as in this example, or on a UMA device where the CPU and GPU share memory. Measure with the actual device, input size, and data reuse count. See [Khronos guidance on staging and UMA](https://docs.vulkan.org/guide/latest/synchronization_examples.html#_upload_data_from_the_cpu_to_a_vertex_buffer).

> [!NOTE]
> **Questions to consider**
>
> Which is more effective: letting the GPU read CPU-accessible memory directly, or copying from a staging buffer to device memory first? Compare one-time reads with repeated reuse, and integrated GPUs with discrete GPUs.

## Dependencies between transfers and computation

Lesson 11's `COMPUTE_SHADER / SHADER_WRITE -> HOST / HOST_READ` dependency allowed the CPU to read the GPU output buffer directly. The CPU now reads a different buffer, so these three dependencies are needed.

| Buffer | Earlier access | Later access |
|---|---|---|
| `inputA`, `inputB` | `TRANSFER / TRANSFER_WRITE` | `COMPUTE_SHADER / SHADER_READ` |
| `output` | `COMPUTE_SHADER / SHADER_WRITE` | `TRANSFER / TRANSFER_READ` |
| `readback` | `TRANSFER / TRANSFER_WRITE` | `HOST / HOST_READ` |

`copyBuffer()` moves buffer data, while barriers specify dependencies that make the results usable by later accesses. The CPU waits for the fence of the submission containing the final barrier before reading `readback`.

Copies and dispatches are submitted to the single compute queue selected earlier. A compute-capable queue also supports these buffer copies, so a separate transfer queue is not needed.

## Pass usage and memory properties to buffer creation

Add the transfer buffers directly below the `output` member.

```cpp
BufferResource uploadA;
BufferResource uploadB;
BufferResource readback;
```

The buffers now have different usages and memory properties. Change the parameter list of `createBuffer()` as follows.

```cpp
bool createBuffer(vk::DeviceSize size, vk::BufferUsageFlags usage,
                  vk::MemoryPropertyFlags properties, BufferResource& resource)
```

Inside the function, assign the supplied value to `bufferInfo.usage` instead of the fixed `eStorageBuffer`.

```cpp
bufferInfo.usage = usage;
```

Remove the two-line local declaration `const auto properties = ...` after the `requirements` query. The existing `findMemoryType()` call now receives `properties` from the function parameter. Keep the memory requirements query, allocation, and buffer binding unchanged.

`findMemoryType()` now searches for properties beyond `HOST_VISIBLE`. Change its final failure message.

```cpp
std::cerr << "No compatible memory type for requested properties.\n";
```

Keep the search condition unchanged. Select a type that `memoryTypeBits` allows and that contains all requested properties. When only `eDeviceLocal` is requested, selecting a type that also has `eHostVisible` is valid.

## Create buffers for each role

Remove the existing body of `createStorageBuffers()` and start with these declarations.

```cpp
const auto hostMemory = vk::MemoryPropertyFlagBits::eHostVisible
                      | vk::MemoryPropertyFlagBits::eHostCoherent;
const auto deviceMemory = vk::MemoryPropertyFlagBits::eDeviceLocal;
const auto inputUsage = vk::BufferUsageFlagBits::eStorageBuffer
                      | vk::BufferUsageFlagBits::eTransferDst;
const auto outputUsage = vk::BufferUsageFlagBits::eStorageBuffer
                       | vk::BufferUsageFlagBits::eTransferSrc;
```

Below them, create the compute and transfer buffers.

```cpp
return createBuffer(bufferSize, inputUsage, deviceMemory, inputA)
    && createBuffer(bufferSize, inputUsage, deviceMemory, inputB)
    && createBuffer(bufferSize, outputUsage, deviceMemory, output)
    && createBuffer(bufferSize, vk::BufferUsageFlagBits::eTransferSrc, hostMemory, uploadA)
    && createBuffer(bufferSize, vk::BufferUsageFlagBits::eTransferSrc, hostMemory, uploadB)
    && createBuffer(bufferSize, vk::BufferUsageFlagBits::eTransferDst, hostMemory, readback);
```

Every buffer uses the same `bufferSize`. `createDescriptorSets()` already connects `inputA`, `inputB`, and `output`, so keep the descriptor code unchanged.

## Redirect CPU writes to staging buffers

Keep the input arrays and initial NaN values in `writeInputs()`. Change only its final three `writeBuffer()` calls as follows.

```cpp
return writeBuffer(uploadA, valuesA)
    && writeBuffer(uploadB, valuesB)
    && writeBuffer(readback, initialOutput);
```

The CPU no longer maps compute memory. The NaN values in `readback` detect a missing result copy; they do not initialize the compute buffer `output`. The shader must write every valid output element.

CPU writes finish before `submitAndWait()`, and the transfer memory is `HOST_COHERENT`, so an explicit flush is unnecessary. Under these conditions, host writes completed before submission are available to the device, so no separate host-to-transfer barrier is added before the input copies. `HOST_COHERENT` does not replace waiting for GPU completion. CPU writes for the next round therefore occur after the previous round's fence wait. See [Host writes and queue submission](https://docs.vulkan.org/spec/latest/chapters/synchronization.html#synchronization-submission-host-writes).

## Record the input copies

In `recordCommandBuffer()`, add the copy region and commands after `cmd.begin(beginInfo)`.

```cpp
vk::BufferCopy copyRegion{};
copyRegion.size = bufferSize;
cmd.copyBuffer(*uploadA.buffer, *inputA.buffer, copyRegion); // C API: vkCmdCopyBuffer
cmd.copyBuffer(*uploadB.buffer, *inputB.buffer, copyRegion); // C API: vkCmdCopyBuffer
```

Initialization with `{}` sets `srcOffset` and `dstOffset` to 0. Copy `bufferSize` bytes from the start of each buffer. These calls record copy commands; the copies execute after submission to the queue.

Record dependencies so the shader reads the inputs after the copies finish. Prepare two buffer memory barriers below the copy commands.

```cpp
std::array<vk::BufferMemoryBarrier, 2> uploadBarriers{};
uploadBarriers[0].buffer = *inputA.buffer;
uploadBarriers[1].buffer = *inputB.buffer;
```

Next, fill in the common properties for both barriers.

```cpp
for (auto& barrier : uploadBarriers)
{
    barrier.srcAccessMask = vk::AccessFlagBits::eTransferWrite;
    barrier.dstAccessMask = vk::AccessFlagBits::eShaderRead;
    barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barrier.offset = 0;
    barrier.size = bufferSize;
}
```

The targets are `inputA` and `inputB`, which the copies wrote, rather than the upload buffers used as copy sources. No queue family change is involved, so set both indices to `VK_QUEUE_FAMILY_IGNORED`.

Record the barriers below the loop.

```cpp
cmd.pipelineBarrier(vk::PipelineStageFlagBits::eTransfer,
                    vk::PipelineStageFlagBits::eComputeShader, {}, {}, uploadBarriers, {}); // C API: vkCmdPipelineBarrier
```

Pipeline stages identify the operation stages, and access masks identify memory accesses within those stages. Here, transfer writes are connected to compute shader reads. The existing pipeline binding, descriptor set binding, push constants, and `dispatch()` follow this code.

## Copy the compute result to the readback buffer

Below `dispatch()`, remove the existing `hostBarrier` declaration through its `cmd.pipelineBarrier()` call. Keep `cmd.end()` and add the output buffer dependency in the removed code's place.

```cpp
vk::BufferMemoryBarrier downloadBarrier{};
downloadBarrier.srcAccessMask = vk::AccessFlagBits::eShaderWrite;
downloadBarrier.dstAccessMask = vk::AccessFlagBits::eTransferRead;
downloadBarrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
downloadBarrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
downloadBarrier.buffer = *output.buffer;
downloadBarrier.offset = 0;
downloadBarrier.size = bufferSize;
```

To let the copy command read the shader's writes to `output`, record a barrier followed by a copy.

```cpp
cmd.pipelineBarrier(vk::PipelineStageFlagBits::eComputeShader,
                    vk::PipelineStageFlagBits::eTransfer, {}, {}, downloadBarrier, {}); // C API: vkCmdPipelineBarrier
cmd.copyBuffer(*output.buffer, *readback.buffer, copyRegion); // C API: vkCmdCopyBuffer
```

The input and output sizes are the same, so reuse the earlier `copyRegion`. GPU results are now copied to a buffer the CPU can read.

## Make the copied results visible to the CPU

After the output copy, add a new `hostBarrier`. Its target is `readback`, and its earlier access is a transfer write.

```cpp
vk::BufferMemoryBarrier hostBarrier{};
hostBarrier.srcAccessMask = vk::AccessFlagBits::eTransferWrite;
hostBarrier.dstAccessMask = vk::AccessFlagBits::eHostRead;
hostBarrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
hostBarrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
hostBarrier.buffer = *readback.buffer;
hostBarrier.offset = 0;
hostBarrier.size = bufferSize;
```

Below it, record the transfer-to-host dependency. Keep the existing `cmd.end()` and the rest of the function after this code.

```cpp
cmd.pipelineBarrier(vk::PipelineStageFlagBits::eTransfer,
                    vk::PipelineStageFlagBits::eHost, {}, {}, hostBarrier, {}); // C API: vkCmdPipelineBarrier
```

The recorded commands follow this order.

```text
Two input copies
-> transfer write / compute read barrier
-> dispatch
-> compute write / transfer read barrier
-> output copy
-> transfer write / host read barrier
```

Each dependency is expressed with `vk::BufferMemoryBarrier` and `pipelineBarrier()`. For buffer memory barrier scopes and the relationship between stages and access masks, see the [Vulkan API reference](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdPipelineBarrier.html).

## Check readback and reuse

At the start of `verifyOutput()`, change the mapping and unmapping calls from `output.memory` to `readback.memory`. Including the array declaration and copy, the code is as follows.

```cpp
std::array<float, elementCount> results{};
void* mapped = readback.memory.mapMemory(0, bufferSize); // C API: vkMapMemory
std::memcpy(results.data(), mapped, static_cast<size_t>(bufferSize));
readback.memory.unmapMemory(); // C API: vkUnmapMemory
```

Keep the loop that calculates CPU reference values and compares results. Since `readback` uses `HOST_COHERENT` memory, an explicit invalidate is unnecessary.

`run()` records the command buffer once and submits it twice with different inputs. The recording contains buffer handles and copy ranges, so the second submission uses the same resources but reads new input values. The preceding fence wait must finish before overwriting the upload buffers and resubmitting the command buffer. Buffers, descriptors, and the pipeline are not recreated each round.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), change from the repository root to the example directory.

```sh
cd examples/12-staging-buffers
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target staging_buffers_example
.\build\Debug\main.exe
```

Build and run the completed example with these commands.

```powershell
cmake --build build --config Debug --target staging_buffers_completed
.\build\completed\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target staging_buffers_example
./build/main
```

Build and run the completed example with these commands.

```sh
cmake --build build --target staging_buffers_completed
./build/completed/main
```

### Expected output

Both runs with different inputs must match the CPU's `A + B` results. Check that the program exits without validation errors, and compare the mapped buffers with the code. The CPU accesses `uploadA`, `uploadB`, and `readback`, while the descriptors refer to `inputA`, `inputB`, and `output`.

With the default `elementCount = 8`, verification prints the following.

```text
Round 1: verified 8 elements; first=10, last=24
Round 2: verified 8 elements; first=12, last=26
Resources destroyed.
```

After completing the code for this lesson, submit a screenshot of the terminal showing the output.
