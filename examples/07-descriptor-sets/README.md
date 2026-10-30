# 07. Descriptor pools and sets

[Previous: Descriptor set layout](../06-descriptor-layout/README.md) | [Next: Compute shader](../08-compute-shader/README.md) | [All tutorials](../../README.md#tutorials)

The previous lesson defined a layout with one storage buffer at each of bindings 0, 1, and 2. Now allocate a descriptor set with that structure and connect the input and output buffers to its bindings.

Descriptor sets are allocated from a descriptor pool. First provide the pool with enough capacity, then allocate the set and write the buffer information into it.

Open [main.cpp](main.cpp) and add these steps after the previous lesson's layout creation. Once published, the completed code is available in `examples/99-completed/07-descriptor-sets.cpp`.

## Add members to hold the pool and sets

Use `std::vector` to hold the allocated sets. Add this line to the headers at the top of the file.

```cpp
#include <vector>
```

In the `private` section, directly below `descriptorSetLayout`, declare the pool and sets in this order.

```cpp
vk::raii::DescriptorPool descriptorPool = nullptr;
std::vector<vk::raii::DescriptorSet> descriptorSets;
```

RAII members are destroyed in reverse declaration order. Declaring the sets after the pool keeps the pool alive while the sets free themselves. This lesson allocates one set and accesses it through `descriptorSets[0]`.

## Set the pool capacity

The pool needs capacity for the sets and descriptors to allocate. First, add an empty pool creation function at the end of the class's `private` section.

```cpp
bool createDescriptorPool()
{
}
```

Inside the function, specify the descriptor types and counts to provide. Prepare three storage buffer descriptors for two inputs and one output.

```cpp
vk::DescriptorPoolSize poolSize{};
poolSize.type = vk::DescriptorType::eStorageBuffer;
poolSize.descriptorCount = 3;
```

This is the storage buffer descriptor capacity of the entire pool. The layout set `descriptorCount` to 1 per binding, but the pool must accommodate all three bindings.

Next, create the pool creation information. RAII sets free themselves individually on destruction, so set the flag that permits this.

```cpp
vk::DescriptorPoolCreateInfo poolInfo{};
poolInfo.flags = vk::DescriptorPoolCreateFlagBits::eFreeDescriptorSet;
```

A pool with `eFreeDescriptorSet` allows allocated sets to be freed individually. Next, set the maximum number of sets that can be allocated from the pool.

```cpp
poolInfo.maxSets = 1;
```

One set is enough because it will contain all three descriptors. `maxSets` limits the number of sets, while `poolSize.descriptorCount` limits the number of descriptors of a particular type.

Connect the descriptor capacity information prepared above to the creation information.

```cpp
poolInfo.poolSizeCount = 1;
poolInfo.pPoolSizes = &poolSize;
```

`poolSizeCount` is the number of capacity entries passed through `pPoolSizes`. Only one type, storage buffer, is used, so pass the address of one entry.

## Create the pool

With the creation information ready, create the pool on the logical device. Continue below the preceding code in the same function.

```cpp
descriptorPool = vk::raii::DescriptorPool(device, poolInfo); // C API: vkCreateDescriptorPool
```

The created pool is stored in the member. Other functions can now use it to allocate sets. At the end of the function, print the result and return success.

```cpp
std::cout << "Descriptor pool created: max sets " << poolInfo.maxSets << '\n';
```

Print the descriptor capacity for each type by iterating over the entries connected to the creation information. Changes to the capacity settings will also appear in the output.

```cpp
for (uint32_t i = 0; i < poolInfo.poolSizeCount; ++i)
{
    const auto& capacity = poolInfo.pPoolSizes[i];
    std::cout << "  " << vk::to_string(capacity.type)
              << ": " << capacity.descriptorCount << " descriptors\n";
}
return true;
```

## Allocate a set using the layout

Allocate a set from the pool. Add this empty function below the one just written, before the end of the class.

```cpp
bool createDescriptorSets()
{
}
```

Allocation requires the layout for each set. First, retrieve the Vulkan handle from the RAII layout created in the previous lesson.

```cpp
const vk::DescriptorSetLayout layout = *descriptorSetLayout;
```

The `*` operator on an RAII object retrieves its underlying handle. This handle identifies the layout of the three storage buffer bindings defined earlier.

Create the allocation information and specify the pool to use. Pass the underlying pool handle from the RAII object here as well.

```cpp
vk::DescriptorSetAllocateInfo allocInfo{};
allocInfo.descriptorPool = *descriptorPool;
```

Next, specify the number of sets to allocate and their layouts. There is one set, so pass the address of one layout handle.

```cpp
allocInfo.descriptorSetCount = 1;
allocInfo.pSetLayouts = &layout;
```

`pSetLayouts` points to an array of layout handles, one for each set. Vulkan uses this information to consume the required descriptor capacity from the pool.

Call the allocation function and store the returned RAII sets in the member.

```cpp
descriptorSets = device.allocateDescriptorSets(allocInfo); // C API: vkAllocateDescriptorSets
```

Allocation is complete, but the set does not yet contain actual buffer information. Next, specify which buffer region each of bindings 0, 1, and 2 refers to.

## Specify the buffers and regions

A descriptor that refers to a buffer needs the buffer handle, starting offset, and size of the region to use. Add an array of `vk::DescriptorBufferInfo` below the allocation code to hold this information.

```cpp
std::array<vk::DescriptorBufferInfo, 3> bufferInfos{};
bufferInfos[0].buffer = *inputA.buffer;
bufferInfos[1].buffer = *inputB.buffer;
bufferInfos[2].buffer = *output.buffer;
```

The array index will be used as the binding number, so entry 0 contains `inputA`, entry 1 `inputB`, and entry 2 `output`. This only fills a local array; it has not updated the set yet.

Use all the data from the start of each buffer. Add this loop below the array initialization.

```cpp
for (auto& bufferInfo : bufferInfos)
{
    bufferInfo.offset = 0;
    bufferInfo.range = bufferSize;
}
```

`offset` is the start of the region within the buffer, and `range` is its size in bytes. The previous lesson defined `bufferSize` as `sizeof(float) * elementCount`, so the region includes every element in each buffer.

## Specify the destination bindings

With the buffer information ready, specify the destination set and binding numbers. Each `vk::WriteDescriptorSet` describes information to write to descriptors. Add an array and another loop below the loop that filled the buffer regions.

```cpp
std::array<vk::WriteDescriptorSet, 3> writes{};
for (uint32_t i = 0; i < writes.size(); ++i)
{
    // Specify the descriptor information to write to this binding.
}
```

First, replace the comment inside the loop with this code. All three entries update the same set; only the binding number changes with `i`.

```cpp
writes[i].dstSet = *descriptorSets[0];
writes[i].dstBinding = i;
writes[i].dstArrayElement = 0;
```

`dstSet` is the set to update, and `dstBinding` is the binding number within it. A binding can contain an array of descriptors. Each binding here has only one, so use `dstArrayElement = 0` to select its first element.

Continue inside the same loop by setting the descriptor type and count. These values must match the structure defined by the layout in the previous lesson.

```cpp
writes[i].descriptorType = vk::DescriptorType::eStorageBuffer;
writes[i].descriptorCount = 1;
```

Write one storage buffer descriptor to each binding. Finally, connect the buffer information prepared earlier.

```cpp
writes[i].pBufferInfo = &bufferInfos[i];
```

`writes[0]` now specifies the `inputA` region for binding 0, `writes[1]` the `inputB` region for binding 1, and `writes[2]` the `output` region for binding 2.

## Apply the updates to the set

So far, the loop has only prepared the updates in local arrays. Call this function after the loop to apply them to the actual descriptor set.

```cpp
device.updateDescriptorSets(writes, {}); // C API: vkUpdateDescriptorSets
```

The first argument contains the update entries just filled in. The second can specify descriptors to copy from other descriptors. No copies are needed here, so pass an empty list, `{}`.

This call configures the set to refer to the buffer regions; it does not copy the data inside the buffers. The previous lesson already wrote that data to memory. At the end of the function, read the allocated set count from the container and print it.

```cpp
std::cout << "Descriptor sets updated: " << descriptorSets.size() << '\n';
```

Also check what was connected to each binding. Iterate over the `writes` used for the update and print the actual binding numbers, types, and counts.

```cpp
for (const auto& write : writes)
{
    std::cout << "Binding " << write.dstBinding << ": "
              << vk::to_string(write.descriptorType) << ", "
              << write.descriptorCount << " descriptors\n";
    // Print the buffer regions referenced by this binding.
}
```

Replace the comment inside the loop with this code. Buffer handles identify resources and may differ between runs. The offsets and ranges are the values set in the descriptors above.

```cpp
for (uint32_t i = 0; i < write.descriptorCount; ++i)
{
    const auto& info = write.pBufferInfo[i];
    std::cout << "  Buffer " << static_cast<VkBuffer>(info.buffer)
              << ", offset " << info.offset << ", range " << info.range << " bytes\n";
}
```

Return success after the outer loop ends.

```cpp
return true;
```

> [!NOTE]
> **Questions to consider**
>
> Assume that the GPU has finished its previous use of the resources and commands for the next execution have not yet been recorded. In each case below, which buffers, descriptor sets, pools, and layouts can be reused, and which must be updated or created?
>
> - Write new numbers into the same regions of the existing A and B buffers and store the result in C. The element count and data type stay the same.
> - Replace A with buffer A' of the same size and data type. A' is already prepared, and the existing reference to A does not need to be retained.
> - Alternate between `C₀ = A₀ + B₀` and `C₁ = A₁ + B₁`. All six buffers are ready, and the shader and binding structure are the same. Only the first group's set exists, and the pool has spare capacity. What is needed to retain both groups of buffer references without rewriting descriptors on each run?
> - A pool has `maxSets = 3` and capacity for 6 storage buffer descriptors. Two sets with 3 descriptors each have been allocated. One more set of the same structure is needed while retaining the existing sets, and its buffers are ready. Does the pool have enough capacity? If not, what must be prepared, and must the layout also be recreated?
> - In addition to the existing storage buffer bindings 0 through 2, a new shader requires a uniform buffer at binding 3 for settings. The settings buffer is ready, and the pool has enough set capacity and capacity for each descriptor type. Is adding another write to the existing set enough? Leave pipeline configuration out of this comparison.

## Connect the initialization steps

The functions for creating the pool and allocating and updating sets are complete. Replace the final `&& createDescriptorSetLayout();` line in `initVulkan()` with these three lines. Keep the earlier initialization calls.

```cpp
    && createDescriptorSetLayout()
    && createDescriptorPool()
    && createDescriptorSets();
```

After the layout is created, create the pool, allocate a set from it, and fill in the buffer information. If an earlier step returns `false`, the following calls are skipped.

The descriptor set now refers to the buffers. To compute with this set, later lessons must prepare the pipeline and command buffer, bind the set, and submit compute commands.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), change from the repository root to the example directory.

```sh
cd examples/07-descriptor-sets
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target descriptor_sets_example
.\build\Debug\main.exe
```

If the completed example is available, run the CMake configuration command above, then build and run it with these commands.

```powershell
cmake --build build --config Debug --target descriptor_sets_completed
.\build\completed\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target descriptor_sets_example
./build/main
```

If the completed example is available, run the CMake configuration command above, then build and run it with these commands.

```sh
cmake --build build --target descriptor_sets_completed
./build/completed/main
```

### Expected output

Check the set and descriptor capacity provided by the pool. Compare the update output with the code to verify that each binding refers to the intended input or output buffer.

After completing the code for this lesson, submit a screenshot of the terminal showing the output.

---

This document was translated and adapted from [Khronos Vulkan Tutorial: Descriptor pool and sets](https://docs.vulkan.org/tutorial/latest/05_Uniform_buffers/01_Descriptor_pool_and_sets.html) for compute exercises. The original material and this document are licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
