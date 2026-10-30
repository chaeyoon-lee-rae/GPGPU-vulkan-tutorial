# 06. Descriptor set layout

[Previous: Buffers and memory](../05-buffers-memory/README.md) | [Next: Descriptor pools and sets](../07-descriptor-sets/README.md) | [All tutorials](../../README.md#tutorials)

The previous lesson created buffers for two inputs and one output. How does the shader access them? In Vulkan, descriptors connect resources such as buffers and images to shaders.

Before connecting the buffers, specify the types and numbers of resources the shader will use. A descriptor set layout defines this structure. This lesson creates a layout for two inputs and one output.

Open [main.cpp](main.cpp) and add the layout after the buffers from the previous lesson. This lesson does not include a completed source file.

## Add a member to hold the layout

The layout is also used to allocate a descriptor set in the next lesson. To keep it after the function returns, add a member directly below `BufferResource output;` in the `private` section.

```cpp
vk::raii::DescriptorSetLayout descriptorSetLayout = nullptr;
```

Start with an empty RAII object, then store the actual layout after filling in the creation information. Add an empty creation function at the end of the class's `private` section.

```cpp
bool createDescriptorSetLayout()
{
}
```

## Assign binding numbers

A binding number identifies a resource binding within a descriptor set. Input `inputA` will use binding 0, input `inputB` binding 1, and output `output` binding 2. The shader will refer to each buffer with the same numbers.

First, create an array inside the function to hold the binding information. Each `vk::DescriptorSetLayoutBinding` describes one binding, so the array length is 3. `<array>` and `<stdint.h>` were already included in the previous lesson.

```cpp
std::array<vk::DescriptorSetLayoutBinding, 3> bindings{};
```

All three bindings use one storage buffer each and differ only in their binding numbers. Add a loop below the array to avoid repeating the same settings.

```cpp
for (uint32_t i = 0; i < bindings.size(); ++i)
{
    // Define the resource for this binding number.
}
```

Replace the comment inside the loop with this code. Use the array index `i` as the binding number, and select the descriptor type to match the buffers created in the previous lesson.

```cpp
bindings[i].binding = i;
bindings[i].descriptorType = vk::DescriptorType::eStorageBuffer;
```

Each binding now specifies a storage buffer. Continue inside the same loop by adding the descriptor count per binding.

```cpp
bindings[i].descriptorCount = 1;
```

`descriptorCount` is not the number of elements in the buffer. Even though the buffer holds eight `float` values, each binding refers to one buffer, so the count is 1. The three buffers use separate bindings; do not set this to 3.

Finally, specify which shader stage can access this binding. The buffers will be used by a compute shader, so add this line directly below the count setting.

```cpp
bindings[i].stageFlags = vk::ShaderStageFlagBits::eCompute;
```

The loop now fills bindings 0, 1, and 2 with the same type, count, and shader stage. Both inputs and outputs use storage buffer descriptors.

## Combine the bindings into a layout

With the binding array ready, create the information to pass to Vulkan. Add this code after the loop, before the function ends.

```cpp
vk::DescriptorSetLayoutCreateInfo layoutInfo{};
layoutInfo.bindingCount = static_cast<uint32_t>(bindings.size());
layoutInfo.pBindings = bindings.data();
```

`bindingCount` is the number of binding descriptions in the array. `pBindings` points to the start of that array. This counts something different from the `descriptorCount` set for each binding.

Pass the logical device and creation information to the RAII constructor. Store the created layout in the member declared earlier.

```cpp
descriptorSetLayout = vk::raii::DescriptorSetLayout(device, layoutInfo); // C API: vkCreateDescriptorSetLayout
```

The binding information is passed when the layout is created, so the local `bindings` and `layoutInfo` variables do not need to be retained. At the end of the function, use `bindings.size()` to print the actual binding count and return success.

```cpp
std::cout << "Descriptor set layout created: " << bindings.size()
          << " bindings.\n";
return true;
```

No actual buffer handles such as `inputA.buffer` have appeared in this code. The layout defines the resource type and count at each binding number. The next lesson allocates a descriptor set with this structure and connects the actual buffers to the bindings.

## Connect the initialization steps

Call the completed function during initialization. Replace the final `&& createStorageBuffers();` line in `initVulkan()` with these two lines. Keep the earlier instance and device initialization calls.

```cpp
    && createStorageBuffers()
    && createDescriptorSetLayout();
```

Layout creation runs after buffer creation succeeds. Because the calls are joined with `&&`, returning `false` from an earlier step skips the following calls.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), change from the repository root to the example directory.

```sh
cd examples/06-descriptor-layout
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target descriptor_layout_example
.\build\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target descriptor_layout_example
./build/main
```

### Expected output

Compare the binding count in the layout creation output with the size of your `bindings` array. Also check the resource type specified for each binding.

After completing the code for this lesson, submit a screenshot of the terminal showing the output.

---

This document was translated and adapted from [Khronos Vulkan Tutorial: Descriptor layout and buffer](https://docs.vulkan.org/tutorial/latest/05_Uniform_buffers/00_Descriptor_set_layout_and_buffer.html) for compute exercises. The original material and this document are licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
