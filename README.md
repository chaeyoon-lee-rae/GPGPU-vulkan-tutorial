# GPGPU Vulkan Tutorial

A step-by-step introduction to general-purpose GPU computing (GPGPU) with Vulkan, C++, and GLSL. The exercises build a vector addition program, then add staging buffers and multiple dispatches while checking results against CPU reference values.

## Tutorials

The examples use Vulkan-Hpp's `vk::raii`. Comments name C API functions only where a call has a one-to-one correspondence with them.
Starting with lesson 01, open each lesson's `main.cpp` and fill it in by following the code blocks in its README. Available completed examples are in [examples/99-completed/](examples/99-completed/).
Each example builds independently with CMake from its own directory. Follow the build commands in the corresponding lesson.

1. [00. Environment settings](examples/00-env-settings/README.md): Install development tools, query Vulkan devices, and check the C++ build and shader compilation.
2. [01. Create and destroy an instance](examples/01-instance/README.md): Select the API version, query instance extensions, and use RAII for creation and destruction.
3. [02. Validation layers](examples/02-validation-layers/README.md): Enable validation layers and use a debug messenger to receive diagnostic messages.
4. [03. Physical devices and queue families](examples/03-physical-devices/README.md): Query device features and limits, then select a GPU and compute queue family.
5. [04. Logical device and queues](examples/04-logical-device/README.md): Create a logical device, specify the features to use, and prepare a compute queue.
6. [05. Buffers and memory](examples/05-buffers-memory/README.md): Select memory types, allocate, bind, and map memory, and write data from the CPU.
7. [06. Descriptor set layout](examples/06-descriptor-layout/README.md): Specify resource types, counts, and shader stages for each binding, then create the layout.
8. [07. Descriptor pools and sets](examples/07-descriptor-sets/README.md): Create a pool, allocate and update sets, and connect input and output buffers.
9. [08. Compute shader](examples/08-compute-shader/README.md): Examine GLSL vector addition, workgroups, and bounds checks, then load SPIR-V and create a shader module.
10. [09. Compute pipeline](examples/09-compute-pipeline/README.md): Define a pipeline layout and push constant range, then create a compute pipeline.
11. [10. Command buffers](examples/10-command-buffers/README.md): Allocate a command buffer, bind the pipeline and descriptors, set push constants, and record a dispatch.
12. [11. Dispatch and synchronization](examples/11-dispatch-synchronization/README.md): Submit to a queue, use barriers and fences, verify against CPU reference values, and run again with different inputs.
13. [12. Staging buffers](examples/12-staging-buffers/README.md): Separate transfer and compute buffers, copy data, and synchronize transfers with computation.
14. [13. Multiple dispatches](examples/13-multiple-dispatches/README.md): Use an intermediate buffer and two descriptor sets, synchronize compute operations, and verify consecutive calculations.

By the end of lesson 11, the first GPU vector addition is complete. Lesson 12 separates data transfers, and lesson 13 connects an intermediate result kept on the GPU to the next operation.

## Acknowledgement

This project is developed as part of Pseudo-Lab's Open Research Initiative. Special thanks to our contributors and the open source community for their valuable insights and contributions.

### About Pseudo Lab

Pseudo-Lab is a non-profit organization focused on advancing machine learning and AI technologies. Our core values of Sharing, Motivation, and Collaborative Joy drive us to create impactful open-source projects. With over 5k+ researchers, we are committed to advancing machine learning and AI technologies.

## License

[MIT](LICENSE)

Documents are licensed under CC BY-SA 4.0.
