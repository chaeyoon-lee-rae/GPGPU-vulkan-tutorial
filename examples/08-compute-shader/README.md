# 08. Compute shader

[Previous: Descriptor pools and sets](../07-descriptor-sets/README.md) | [Next: Compute pipeline](../09-compute-pipeline/README.md) | [All tutorials](../../README.md#tutorials)

The previous lesson connected inputs `inputA` and `inputB` and output `output` to a descriptor set. This lesson writes a compute shader that adds corresponding elements and creates a shader module for use in C++.

The shader is written in GLSL and converted to a SPIR-V binary during the build. The C++ program reads this binary and passes it to Vulkan. This lesson ends at module creation; GPU computation runs in later lessons that record and submit commands.

Open [add.comp](add.comp) and [main.cpp](main.cpp). The shader file is provided in its completed form, so first examine its declarations and calculation. `main.cpp` contains the code through lesson 07. The [completed code](../99-completed/08-compute-shader.cpp) is available for comparison.

## Assign work to each shader invocation

On the CPU, array addition can use a loop to calculate `output[i] = inputA[i] + inputB[i]`. Each element's calculation is independent, so the GPU applies the same code to many elements in parallel. Each execution is an invocation, and a group of invocations is a workgroup. These correspond to CUDA threads and blocks, respectively.

Start with the following declarations at the top of `add.comp`.

```glsl
#version 450

layout(local_size_x = 64, local_size_y = 1, local_size_z = 1) in;
```

Each workgroup has 64 invocations along the x axis. Keeping the group size fixed and increasing the group count with the input size lets the same shader process larger arrays. [Lesson 10: Command buffers](../10-command-buffers/README.md) sets the number of groups to execute. When `elementCount = 8`, a bounds check allows only 8 of the group's 64 invocations to perform the calculation. `64` is a choice for this exercise, not an optimal size for every GPU.

> [!NOTE]
> **Questions to consider**
>
> What are invocations, workgroups, and dispatches, and which CUDA concepts do they correspond to?

## Declare buffers to match the descriptor bindings

Below the workgroup declaration, define the first input.

```glsl
layout(set = 0, binding = 0, std430) readonly buffer InputA
{
    float values[];
} inputA;
```

`set = 0` selects the descriptor set number, and `binding = 0` selects the binding within that set. This matches the binding connected to `inputA.buffer` in lesson 07. `buffer` declares a block that accesses a storage buffer. Its `float values[]` member specifies how to interpret the bound bytes as a `float` array. The actual memory was already allocated in C++.

`readonly` makes writes through this variable a compile error, preventing accidental overwrites of the input. It communicates the intended access to the compiler but does not guarantee a performance gain. For the same reason, `writeonly` on the output prohibits reads. See [GLSL memory qualifiers](https://docs.vulkan.org/glsl/latest/chapters/variables.html#memory-qualifiers).

`std430` defines memory layout rules, including alignment and array element stride. Under these rules, a `float` array has a 4-byte stride, matching a contiguous C++ array (`std140` uses 16 bytes). If the CPU and shader interpret the stride differently, the same index accesses different bytes. Alignment and padding must also match when using structures or vectors. See [Shader memory layout](https://docs.vulkan.org/guide/latest/shader_memory_layout.html).

The second input and output below it use the same structure.

```glsl
layout(set = 0, binding = 1, std430) readonly buffer InputB
{
    float values[];
} inputB;

layout(set = 0, binding = 2, std430) writeonly buffer Output
{
    float values[];
} resultBuffer;
```

The three blocks correspond to the previous lesson's C++ code as follows. They are connected by set and binding numbers, not by matching GLSL and C++ variable names.

| Set | Binding | Buffer connected in C++ | Shader access |
|---|---|---|---|
| 0 | 0 | `inputA.buffer` | Read `inputA.values[index]` |
| 0 | 1 | `inputB.buffer` | Read `inputB.values[index]` |
| 0 | 2 | `output.buffer` | Write `resultBuffer.values[index]` |

`values[]` is a GLSL runtime-sized array that can be the last member of a storage block. Since its length is not fixed in the source, changing the buffer size does not require recompiling the shader. In this example, a descriptor covering 32 bytes permits access to only eight `float` values; `[]` does not expand the storage.

## Receive the actual element count

Below the buffer declarations, a push constant receives the element count.

```glsl
layout(push_constant) uniform Parameters
{
    uint count;
} params;
```

Push constants pass small values, such as an element count, to a shader by recording them in a command buffer. They are used here to avoid managing a separate buffer and descriptor for the 4-byte `count`. The C++ side will use the matching 32-bit unsigned integer type, `uint32_t`.

This declaration alone does not set `count` to 8. Lesson 09 defines the size and shader stage access for this value in the pipeline layout, and lesson 10 records its actual value.

## Calculate the index and check bounds

The `main()` function at the end of the file is the shader entry point.

```glsl
void main()
{
    uint index = gl_GlobalInvocationID.x;
    if (index >= params.count)
    {
        return;
    }

    resultBuffer.values[index] = inputA.values[index] + inputB.values[index];
}
```

`gl_LocalInvocationID.x` is the **index within a group**, ranging from 0 to 63 in every group. `gl_GlobalInvocationID.x` is the **index within the entire dispatch**, equal to `gl_WorkGroupID.x * 64 + gl_LocalInvocationID.x`. Local index 0 in the second group therefore has global index 64. Using only the local index for array access would make every group process elements 0 through 63, so this shader uses the global index.

The bounds check runs before any buffer read. When `count = 8`, only indices 0 through 7 perform addition; the rest return. Each invocation writes a different output element and does not read other invocations' results, so synchronization between invocations is not needed within this shader.

## Build GLSL into SPIR-V

SPIR-V is a GPU-vendor-independent intermediate representation (IR). GLSL is converted to a common binary format, and the driver converts it to executable code for the target GPU. This separates source-language processing from device-specific compilation. Even with the same SPIR-V, the device must support its features, and performance can differ. See [What is SPIR-V?](https://docs.vulkan.org/guide/latest/what_is_spirv.html).

The `glslc` installed in lesson 00 converts GLSL to SPIR-V. This lesson's [CMakeLists.txt](CMakeLists.txt) already contains the shader build rule. Building the example checks for changes to `add.comp` and generates `build/add.comp.spv`.

The compiler command uses these options.

```sh
glslc --target-env=vulkan1.3 --target-spv=spv1.3 add.comp -o build/add.comp.spv
```

The CMake build runs this compiler command automatically.

CMake passes the generated binary's absolute path as the C++ compile definition `ADD_SHADER_PATH`. The executable uses this path instead of looking for the shader relative to the terminal's working directory. If the build directory is moved, reconfigure CMake and rebuild at the new location.

## Create a shader module in C++

A shader module is a Vulkan object containing SPIR-V code and its entry points. The next lesson selects an entry point from this module and connects the resource interface to create a pipeline. The module itself does not contain buffer bindings or execution commands. See [VkShaderModule](https://docs.vulkan.org/refpages/latest/refpages/source/VkShaderModule.html).

Return to [main.cpp](main.cpp). Add the file input header to the headers at the top. `<vector>` and `<stdint.h>` are already included in the preceding lesson's code.

```cpp
#include <fstream>
```

In the `private` section, add a member to hold the shader module directly below `descriptorSets`.

```cpp
vk::raii::ShaderModule shaderModule = nullptr;
```

Add this empty function at the end of the class's `private` section.

```cpp
bool createShaderModule()
{
}
```

Inside the function, open the file specified by CMake in binary mode. `std::ios::ate` moves the read position to the end after opening the file, allowing its size to be determined.

```cpp
std::ifstream file(ADD_SHADER_PATH, std::ios::ate | std::ios::binary);
```

If opening fails, print the path used and stop initialization. Add this directly below the file opening code.

```cpp
if (!file)
{
    std::cerr << "Cannot open shader: " << ADD_SHADER_PATH << '\n';
    return false;
}
```

Next, obtain the byte size from the position at the end of the file and check it.

```cpp
const std::streamsize byteCount = file.tellg();
if (byteCount <= 0 || byteCount % sizeof(uint32_t) != 0)
{
    std::cerr << "Invalid SPIR-V byte count.\n";
    return false;
}
```

SPIR-V consists of 32-bit words, so the size must be a positive multiple of 4 bytes. This check rejects empty files and invalid lengths; it does not validate the entire file as SPIR-V.

## Read the binary into a 32-bit array

In the same function, add an array to hold the binary. `byteCount` is in bytes, so divide by `sizeof(uint32_t)` to obtain the element count.

```cpp
std::vector<uint32_t> code(static_cast<size_t>(byteCount) / sizeof(uint32_t));
file.seekg(0);
```

Using `std::vector<uint32_t>` provides the pointer type and alignment required by Vulkan. Since the size was read at the end of the file, return to the beginning with `seekg(0)`.

Next, read all the bytes.

```cpp
if (!file.read(reinterpret_cast<char*>(code.data()), byteCount))
{
    std::cerr << "Cannot read SPIR-V file.\n";
    return false;
}
```

`read()` requires a `char*` destination, so convert the pointer only for this call. The storage remains an array of 32-bit integers. If reading fails, incomplete data is not passed to Vulkan.

eva's [`SpvBlob::readFrom()`](https://github.com/vaistrata/eva/blob/2a31c83cc83eac7d4b74f4d9c9d4f6f0752c6f99/src/eva-runtime.cpp#L71-L85) also reads the file into `uint32_t[]` and stores the byte size separately. This example uses the same approach with `std::vector` and a C++ file stream, adding checks for opening, size, and read failures.

## Pass SPIR-V to the module

After reading the binary, add the module creation information.

```cpp
vk::ShaderModuleCreateInfo moduleInfo{};
moduleInfo.codeSize = static_cast<size_t>(byteCount);
moduleInfo.pCode = code.data();
```

`pCode` points to an array of 32-bit words, but `codeSize` is measured in bytes. Using `code.size()` directly would make the size four times too small.

Next, create the module and print the result.

```cpp
shaderModule = vk::raii::ShaderModule(device, moduleInfo); // C API: vkCreateShaderModule
std::cout << "Shader module created: " << moduleInfo.codeSize << " bytes.\n";
return true;
```

The local `code` array can be destroyed after module creation. The module remains in the member for pipeline creation in the next lesson.

## Connect the initialization steps

Replace the final `&& createDescriptorSets();` line in `initVulkan()` with these two lines.

```cpp
    && createDescriptorSets()
    && createShaderModule();
```

After preparing the descriptor set, read the SPIR-V and create the module. The next lesson combines this module with the shader's resource structure to create a compute pipeline.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), change from the repository root to the example directory.

```sh
cd examples/08-compute-shader
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target compute_shader_example
.\build\Debug\main.exe
```

Build and run the completed example with these commands in the same configuration.

```powershell
cmake --build build --config Debug --target compute_shader_completed
.\build\completed\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target compute_shader_example
./build/main
```

Build and run the completed example with these commands in the same configuration.

```sh
cmake --build build --target compute_shader_completed
./build/completed/main
```

### Expected output

Check that the shader module creation message and SPIR-V byte size appear without validation layer errors. The binary size may vary with the compiler. Also check that `build/add.comp.spv` exists and has a size greater than zero.

Do not check the addition result in the output buffer yet. Only the module has been created; the GPU has not run the shader. After completing the code for this lesson, submit a screenshot of the terminal showing the output.

---

References: [GLSL variables, types, and memory layout](https://docs.vulkan.org/glsl/latest/chapters/variables.html), [Vulkan Guide: Push Constants](https://docs.vulkan.org/guide/latest/push_constants.html), [VkShaderModuleCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkShaderModuleCreateInfo.html), [glslc options](https://github.com/google/shaderc/blob/main/glslc/README.asciidoc).
