# 09. Compute pipeline

[Previous: Compute shader](../08-compute-shader/README.md) | [Next: Command buffers](../10-command-buffers/README.md) | [All tutorials](../../README.md#tutorials)

The previous lesson compiled the vector addition shader to SPIR-V and created a shader module. This lesson creates a compute pipeline by specifying the shader's resource structure and entry point.

A pipeline layout defines descriptor set layouts and push constant ranges. A compute pipeline combines this layout with a shader entry point to prepare an operation for GPU execution. Creation may involve device-specific compilation and optimization, so a pipeline is usually reused across multiple dispatches. See [Pipelines and compilation](https://docs.vulkan.org/guide/latest/pipeline_cache.html).

[main.cpp](main.cpp) contains the code through lesson 08. Add the two objects and their creation functions in the order below. Once published, the completed code is available in `examples/99-completed/09-compute-pipeline.cpp`. [add.comp](add.comp) and the shader build rules are unchanged from the previous lesson.

## Distinguish the two layouts

The descriptor set layout in lesson 06 defined the bindings within one set. This lesson's pipeline layout specifies which shader set number that layout belongs to and which push constant ranges are used.

| Object | What it defines in this example |
|---|---|
| `descriptorSetLayout` | One storage buffer at each of bindings 0, 1, and 2 |
| `pipelineLayout` | The layout for set 0 and a 4-byte push constant read by the compute stage |
| `computePipeline` | The shader module's `main` entry point using the layout above |

The descriptor set contains actual buffer references, while the pipeline layout contains only their structure. This separation allows input and output buffers to change without recreating the pipeline, as long as the binding structure stays the same. Lesson 10 binds the pipeline and descriptor set separately when recording commands.

## Add members to hold the pipeline objects

In the `private` section, declare these two members directly below `shaderModule`.

```cpp
vk::raii::PipelineLayout pipelineLayout = nullptr;
vk::raii::Pipeline computePipeline = nullptr;
```

Start with empty RAII objects and store the actual objects during initialization. Add an empty pipeline layout creation function at the end of the class's `private` section.

```cpp
bool createPipelineLayout()
{
}
```

## Define the push constant range

The shader has a push constant block containing one `uint count`. This value is the number of elements to process, not the buffer size in bytes. Declare the range used by C++ inside the function.

```cpp
vk::PushConstantRange pushConstantRange{};
pushConstantRange.stageFlags = vk::ShaderStageFlagBits::eCompute;
pushConstantRange.offset = 0;
pushConstantRange.size = sizeof(uint32_t);
```

`stageFlags` identifies the shader stage that reads the value, `offset` is the starting byte in push constant storage, and `size` is the byte count. GLSL `uint` and C++ `uint32_t` are both 4 bytes here, so expose 4 bytes from offset 0 to the compute stage.

Defining a range and supplying its value are separate steps. This code declares storage for `count`; lesson 10 records the actual value of `elementCount` in the command buffer. Push constant `offset` and `size` must be multiples of 4 bytes, and the range must fit within the device's `maxPushConstantsSize`. This example uses only one `uint`.

## Connect the layout for set 0

In the same function, retrieve the handle from the preceding lesson's RAII descriptor set layout and connect it to the creation information.

```cpp
const vk::DescriptorSetLayout layout = *descriptorSetLayout;
vk::PipelineLayoutCreateInfo layoutInfo{};
layoutInfo.setLayoutCount = 1;
layoutInfo.pSetLayouts = &layout;
```

`pSetLayouts` is an array of layouts ordered by set number. Passing one element assigns that layout to set 0. This must match `layout(set = 0, binding = ...)` in `add.comp`.

Next, connect the push constant range prepared above.

```cpp
layoutInfo.pushConstantRangeCount = 1;
layoutInfo.pPushConstantRanges = &pushConstantRange;
```

`setLayoutCount` is the number of set layouts. Do not set it to 3 because there are three bindings. Similarly, `pushConstantRangeCount` counts range descriptions, not bytes.

## Create the pipeline layout

With the creation information complete, create the layout on the logical device.

```cpp
pipelineLayout = vk::raii::PipelineLayout(device, layoutInfo); // C API: vkCreatePipelineLayout
```

After this call, the local `layout` and `pushConstantRange` variables do not need to be retained. Vulkan uses the structure supplied during creation.

Return success at the end of the function.

```cpp
return true;
```

## Select the compute shader to run

Below the function just written, add an empty compute pipeline creation function.

```cpp
bool createComputePipeline()
{
}
```

Inside the function, fill in the shader stage information.

```cpp
vk::PipelineShaderStageCreateInfo stageInfo{};
stageInfo.stage = vk::ShaderStageFlagBits::eCompute;
stageInfo.module = *shaderModule;
stageInfo.pName = "main";
```

`stage` selects the compute stage, and `module` is the handle of the shader module created in the previous lesson. `pName` names an entry point in SPIR-V, corresponding to `void main()` in `add.comp`. It does not specify the shader filename or C++ `main()`.

Next, combine the shader stage and pipeline layout in one creation information structure.

```cpp
vk::ComputePipelineCreateInfo pipelineInfo{};
pipelineInfo.stage = stageInfo;
pipelineInfo.layout = *pipelineLayout;
```

A compute pipeline specifies one compute shader stage. Since it directly performs array computation, it does not need a graphics pipeline's vertex input or rasterization settings. The storage buffer bindings and push constants used here must all match the structure described by `pipelineInfo.layout`.

Pass the logical device and creation information to the RAII constructor.

```cpp
computePipeline = vk::raii::Pipeline(device, nullptr, pipelineInfo); // C API: vkCreateComputePipelines
```

The second argument is a pipeline cache. This example uses no cache object, so pass `nullptr`. A cache reuses results from pipeline creation; it does not execute or synchronize commands.

At the end of the function, print the creation result and return success.

```cpp
std::cout << "Compute pipeline created.\n";
return true;
```

The shader module is used during pipeline creation. It can be released afterward, but this example keeps the preceding lesson's RAII member structure and releases it when the program exits.

## Connect the initialization steps

Replace the final `&& createShaderModule();` line in `initVulkan()` with these three lines.

```cpp
    && createShaderModule()
    && createPipelineLayout()
    && createComputePipeline();
```

Create the pipeline after the shader module and pipeline layout are ready. So far, the executable program and resource structure have been defined. The next lesson records this pipeline, the descriptor set, and the element count in a command buffer.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), change from the repository root to the example directory.

```sh
cd examples/09-compute-pipeline
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target compute_pipeline_example
.\build\Debug\main.exe
```

If the completed example is available, build and run it with these commands in the same configuration.

```powershell
cmake --build build --config Debug --target compute_pipeline_completed
.\build\completed\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target compute_pipeline_example
./build/main
```

If the completed example is available, build and run it with these commands in the same configuration.

```sh
cmake --build build --target compute_pipeline_completed
./build/completed/main
```

### Expected output

Check that `Compute pipeline created.` appears after `Shader module created: ... bytes.`. If the validation layer reports a shader binding or push constant range mismatch, compare lesson 06's bindings and this lesson's layout with the declarations in `add.comp`.

Pipeline creation is separate from executing the computation. This lesson submits no commands to the queue, so the output buffer retains its initial values. The next lesson records commands using this pipeline.

After completing the code for this lesson, submit a screenshot of the terminal showing the output.

---

References: [VkPipelineLayoutCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineLayoutCreateInfo.html), [VkPushConstantRange](https://docs.vulkan.org/refpages/latest/refpages/source/VkPushConstantRange.html), [Vulkan Guide: Push Constants](https://docs.vulkan.org/guide/latest/push_constants.html), [VkComputePipelineCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkComputePipelineCreateInfo.html).
