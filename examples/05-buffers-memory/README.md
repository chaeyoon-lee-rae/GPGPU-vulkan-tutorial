# 05. Buffers and memory

[Previous: Logical device and queues](../04-logical-device/README.md) | [Next: Descriptor set layout](../06-descriptor-layout/README.md) | [All tutorials](../../README.md#tutorials)

With the logical device and queue ready, prepare the data for computation. In Vulkan, a buffer is a resource that holds data, but creating a buffer does not allocate storage. Suitable memory must be found, allocated, and bound to the buffer.

Create two input buffers and one output buffer. Write values to each buffer from the CPU, then read them back to verify them. Later lessons use the GPU to compute with these values.

Open [main.cpp](main.cpp) and follow the steps below. The [completed code](../99-completed/05-buffers-memory.cpp) is available for comparison.

## Set the buffer size and add storage members

`<array>` is needed to hold the input and readback values. Add `<stddef.h>` for `size_t`, which specifies the number of bytes to copy. Add these headers at the top of the file.

```cpp
#include <array>
#include <stddef.h>
```

Each buffer will hold eight 4-byte `float` values. In the `private` section, after `validationLayer`, define the element count and buffer size.

```cpp
static_assert(sizeof(float) == 4);
static constexpr uint32_t elementCount = 8;
static constexpr vk::DeviceSize bufferSize = sizeof(float) * elementCount;
```

`bufferSize` is the size in bytes of one element multiplied by the element count. `static_assert` checks the size of `float` at compile time.

Directly below, add a structure to hold each buffer object together with its memory object.

```cpp
struct BufferResource
{
    vk::raii::DeviceMemory memory = nullptr;
    vk::raii::Buffer buffer = nullptr;
};
```

Members are destroyed in reverse declaration order. Declaring `memory` first ensures that `buffer` is destroyed before the memory bound to it.

Below `computeQueue`, add members for the two input buffers and one output buffer.

```cpp
BufferResource inputA;
BufferResource inputB;
BufferResource output;
```

## Create a buffer object

Input and output buffers can be created in the same way. Add an empty function for creating one buffer at the end of the class's `private` section. It takes the byte size in `size` and stores the created objects in `resource`, which is passed by reference.

```cpp
bool createBuffer(vk::DeviceSize size, BufferResource& resource)
{
}
```

First, set the buffer size and usage inside the function. Use `eStorageBuffer` because the buffer will hold data that a compute shader reads and writes.

```cpp
vk::BufferCreateInfo bufferInfo{};
bufferInfo.size = size;
bufferInfo.usage = vk::BufferUsageFlagBits::eStorageBuffer;
bufferInfo.sharingMode = vk::SharingMode::eExclusive;
```

`sharingMode` specifies how the resource is shared between queue families. This example uses only the previously selected family, so set it to `eExclusive`.

With the creation information ready, create the buffer object below it.

```cpp
resource.buffer = vk::raii::Buffer(device, bufferInfo); // C API: vkCreateBuffer
```

Only the buffer object has been created so far. To store data, it must be bound to memory that the buffer can use.

## Check the buffer's memory requirements

Before allocating memory, find the size and memory types required by the driver. Retrieve the requirements on the line after buffer creation.

```cpp
const auto requirements = resource.buffer.getMemoryRequirements(); // C API: vkGetBufferMemoryRequirements
```

`requirements.size` is the size that must actually be allocated and may exceed the requested buffer size. `alignment` is the alignment required for the memory binding offset. `memoryTypeBits` identifies the memory types allowed for this buffer using bits.

Among the types allowed by the buffer, select memory that the CPU can write directly. Specify the required properties below the query.

```cpp
const auto properties = vk::MemoryPropertyFlagBits::eHostVisible
    | vk::MemoryPropertyFlagBits::eHostCoherent;
```

`eHostVisible` means that the memory can be mapped for CPU access. `eHostCoherent` means that explicit cache flush and invalidate calls are not required for CPU access. It does not wait for GPU work to complete.

Find a memory type that is allowed by the buffer and has both properties. Write the search function first, then call it here.

## Find a suitable memory type

Outside `createBuffer()`, add this empty function below it.

```cpp
bool findMemoryType(uint32_t typeFilter, vk::MemoryPropertyFlags properties,
                    uint32_t& typeIndex)
{
}
```

`typeFilter` specifies the types allowed by the buffer, and `properties` specifies the properties the program needs. When a suitable type is found, store its index in `typeIndex` and return `true`.

First, query the physical device's memory types inside the function.

```cpp
const auto memoryProperties = physicalDevice.getMemoryProperties(); // C API: vkGetPhysicalDeviceMemoryProperties
```

Properties may differ between types, so check the list in order. Add this empty loop below the query.

```cpp
for (uint32_t i = 0; i < memoryProperties.memoryTypeCount; ++i)
{
    // Check the buffer requirements and required memory properties.
}
```

Inside the loop, first check whether this buffer allows memory type `i`. Replace the comment with this line.

```cpp
const bool allowed = (typeFilter & (1u << i)) != 0;
```

`1u << i` has only bit `i` set. If the same bit is set in `typeFilter`, that memory type is allowed for this buffer.

An allowed type does not necessarily support CPU access. Directly below, check whether this type has all the requested properties.

```cpp
const bool hasProperties =
    (memoryProperties.memoryTypes[i].propertyFlags & properties) == properties;
```

`properties` will contain both `eHostVisible` and `eHostCoherent`. The bitwise result must equal `properties` for all requested properties to be present.

If both conditions pass, save the type index and finish the search. Add this inside the same loop.

```cpp
if (allowed && hasProperties)
{
    typeIndex = i;
    return true;
}
```

Reaching the end of the loop means that no suitable type was found. Add failure handling outside the loop, at the end of the function.

```cpp
std::cerr << "No compatible HOST_VISIBLE | HOST_COHERENT memory type found.\n";
return false;
```

The search function is complete. Return to `createBuffer()` and call it after declaring `properties`.

```cpp
uint32_t memoryTypeIndex = 0;
if (!findMemoryType(requirements.memoryTypeBits, properties, memoryTypeIndex))
{
    return false;
}
```

If the search fails, stop initializing this buffer. On success, `memoryTypeIndex` holds the selected type's index, so memory allocation can proceed.

> [!NOTE]
> **Questions to consider**
>
> What do `eHostVisible`, `eHostCoherent`, and `eDeviceLocal` each guarantee? Check whether your GPU has a memory type with both `eHostVisible` and `eDeviceLocal`. On a discrete GPU, where is memory allocated when it has only `eDeviceLocal`, only `eHostVisible`, or both properties?

## Allocate memory and bind it to the buffer

In the same function, create the allocation information below the new `if` statement. Use the driver's `requirements.size` for the size and the search result for the type.

```cpp
vk::MemoryAllocateInfo allocationInfo{};
allocationInfo.allocationSize = requirements.size;
allocationInfo.memoryTypeIndex = memoryTypeIndex;
```

Next, allocate the memory and store it in `resource`.

```cpp
resource.memory = vk::raii::DeviceMemory(device, allocationInfo); // C API: vkAllocateMemory
```

With both the buffer and memory created, bind them together.

```cpp
resource.buffer.bindMemory(*resource.memory, 0); // C API: vkBindBufferMemory
```

The second argument is the byte offset at which the buffer starts within the allocation. Each buffer has a separate allocation, so use 0, the start of that allocation. This also satisfies the buffer's alignment requirement.

At the end of the function, print the selected memory information and return success.

```cpp
std::cout << "Buffer memory: size " << requirements.size
          << ", alignment " << requirements.alignment
          << ", type " << memoryTypeIndex << '\n';
return true;
```

> [!NOTE]
> **Questions to consider**
>
> What do buffer creation, memory allocation, and `bindMemory()` each do? Why does Vulkan separate these three steps? Compare them with CUDA's `cudaMalloc()`.

## Write values from the CPU

With the buffer and memory ready, write values to them. Below `findMemoryType()`, add an empty function to write and verify the data.

```cpp
bool writeAndVerifyBuffer(BufferResource& resource,
                          const std::array<float, elementCount>& values,
                          const char* label)
{
}
```

`resource` is the buffer to write, and `values` contains the data. `label` distinguishes buffers such as input A and input B in the output.

Memory must be mapped before the CPU can access it. On the first line of the function, map a range equal to the buffer size.

```cpp
void* mapped = resource.memory.mapMemory(0, bufferSize); // C API: vkMapMemory
```

The CPU can access memory through the address returned by `mapMemory()`. Copy the array's bytes to that address, then unmap the memory when finished.

```cpp
std::memcpy(mapped, values.data(), static_cast<size_t>(bufferSize));
resource.memory.unmapMemory(); // C API: vkUnmapMemory
```

The starter code already includes `<cstring>`, which provides `std::memcpy`. `unmapMemory()` removes the mapping to a CPU address; it does not free the memory or erase the stored values.

## Read the values back

To verify the write, map the same memory again and read it into a separate array. Add this code below the `unmapMemory()` call just written.

```cpp
std::array<float, elementCount> readback{};
mapped = resource.memory.mapMemory(0, bufferSize); // C API: vkMapMemory
std::memcpy(readback.data(), mapped, static_cast<size_t>(bufferSize));
resource.memory.unmapMemory(); // C API: vkUnmapMemory
```

The copy direction is reversed this time. Values move from the buffer's memory into the CPU array `readback`.

Directly below, compare the readback array with the original values. If they differ, print which buffer failed and return failure.

```cpp
if (readback != values)
{
    std::cerr << label << ": CPU data verification failed.\n";
    return false;
}
```

If the values match, print the array for visual inspection. Add this code at the end of the function.

```cpp
std::cout << label << ':';
for (float value : readback)
{
    std::cout << ' ' << value;
}
std::cout << '\n';
return true;
```

Here, the CPU reads back values it wrote itself. No GPU work has been submitted, so queue submission and GPU completion waits are not needed yet.

> [!NOTE]
> **Questions to consider**
>
> What do `mapMemory()`, `memcpy()`, and `unmapMemory()` each do? After unmapping, can the stored values and the returned pointer still be used? If the CPU reads results computed by the GPU instead of values it wrote itself, is `eHostCoherent` alone enough?

## Prepare the input and output buffers

The buffer creation and data writing functions are ready. Use them to create two input buffers and one output buffer. Add this empty function before `createBuffer()`.

```cpp
bool createStorageBuffers()
{
}
```

Inside the function, collect the addresses of the buffers to create. The array size is deduced from its elements. This array will also be used to print the number of created buffers.

```cpp
const std::array buffers = {&inputA, &inputB, &output};
```

Loop over the array and create each buffer and its memory. If any creation fails, stop before further initialization.

```cpp
for (auto* buffer : buffers)
{
    if (!createBuffer(bufferSize, *buffer))
    {
        return false;
    }
}
```

With the buffers created, prepare the values to write. Below the loop, create two input arrays and an array of initial output values.

```cpp
std::array<float, elementCount> valuesA{};
std::array<float, elementCount> valuesB{};
const std::array<float, elementCount> initialOutput{};
```

Elements initialized with `{}` are all zero. Leave the output array as it is, and fill input A with 0 through 7 and input B with 10 through 17.

```cpp
for (uint32_t i = 0; i < elementCount; ++i)
{
    valuesA[i] = static_cast<float>(i);
    valuesB[i] = 10.0f + static_cast<float>(i);
}
```

After preparing the values, write each array to its buffer and verify it below the loop.

```cpp
if (!writeAndVerifyBuffer(inputA, valuesA, "Input A")
    || !writeAndVerifyBuffer(inputB, valuesB, "Input B")
    || !writeAndVerifyBuffer(output, initialOutput, "Output (initial)"))
{
    return false;
}
```

Each call writes data, reads it back, and compares it. If all calls succeed, print a completion message at the end of the function.

```cpp
std::cout << "Host-visible storage buffers ready: " << buffers.size()
          << " x " << bufferSize << " bytes.\n"
          << "CPU write/readback verified. No GPU dispatch yet.\n";
return true;
```

## Connect the initialization steps

With `createStorageBuffers()` complete, connect it to initialization. Buffers and memory are created through the logical device, so add the call at the end of `initVulkan()`.

```cpp
bool initVulkan()
{
    return createInstance()
        && setupDebugMessenger()
        && pickPhysicalDevice()
        && createLogicalDevice()
        && createStorageBuffers();
}
```

The buffer members are declared after `device`, so the buffers and memory are released before the device. The next lesson creates a descriptor set layout to specify how these buffers connect to the shader.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), change from the repository root to this example directory.

```sh
cd examples/05-buffers-memory
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target buffers_memory_example
.\build\Debug\main.exe
```

Build and run the completed example with these commands in the same configuration.

```powershell
cmake --build build --config Debug --target buffers_memory_completed
.\build\completed\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target buffers_memory_example
./build/main
```

Build and run the completed example with these commands in the same configuration.

```sh
cmake --build build --target buffers_memory_completed
./build/completed/main
```

### Expected output

Check that the values read from the input buffers match the arrays prepared in the code. Compare the output buffer's initial values and the byte size per buffer with your own calculations. The memory type and alignment depend on the device.

After completing the code for this lesson, submit a screenshot of the terminal showing the output.

---

This document was translated and adapted from [Khronos Vulkan Tutorial: Vertex buffer creation](https://docs.vulkan.org/tutorial/latest/04_Vertex_buffers/01_Vertex_buffer_creation.html) for exercises in allocating and mapping storage buffers for compute. The original material and this document are licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
