# 01. Create and destroy an instance

[Previous: Environment settings](../00-env-settings/README.md) | [Next: Validation layers](../02-validation-layers/README.md) | [All tutorials](../../README.md#tutorials)

To use Vulkan, first create an instance. The instance connects the application to the Vulkan library. When creating it, provide information about the program and the API version it will use.

Open [main.cpp](main.cpp) and fill in the code by following the instructions below. The [completed code](../99-completed/01-instance.cpp) is available for comparison.

## Find where initialization starts

In the starter code, `main()` creates a `ComputeApplication` object and calls `app.run()`. The `run()` function in the class's `public` section calls the initialization function.

```cpp
void run()
{
    initVulkan();
}
```

The body of `initVulkan()` is still empty. As each Vulkan object is added, connect its initialization step to this function. Start by preparing to create an instance.

## Add a member to hold the instance

The instance must remain available after the creation function returns. Add these members directly below `private`.

```cpp
vk::raii::Context context;
vk::raii::Instance instance = nullptr;
```

`context` provides access to the functions needed to create an instance through the Vulkan loader. Initialize `instance` to `nullptr` because it has not been created yet.

Next, add an empty instance creation function in the `private` section. Write the following code inside this function in order.

```cpp
void createInstance()
{
}
```

## Set the application information

Tell the driver which program is using Vulkan. Inside `createInstance()`, create a `vk::ApplicationInfo` structure and set the program's name and version.

```cpp
vk::ApplicationInfo appInfo{};
appInfo.pApplicationName = "01-instance";
appInfo.applicationVersion = VK_MAKE_API_VERSION(0, 1, 0, 0);
```

`VK_MAKE_API_VERSION(0, 1, 0, 0)` packs version 1.0.0 with variant 0 into a single integer. This is the version of the program being written.

Continue by adding engine information to the same structure. Set the name to `"No Engine"` because this example does not use a separate engine.

```cpp
appInfo.pEngineName = "No Engine";
appInfo.engineVersion = VK_MAKE_API_VERSION(0, 1, 0, 0);
```

The Vulkan API version must also be set, independently of the program version. This tutorial uses Vulkan 1.3.

```cpp
appInfo.apiVersion = VK_API_VERSION_1_3;
```

Vulkan-Hpp sets `sType`, which identifies the structure type, so it does not need to be set manually.

## Examine the available extensions

Extensions add features to Vulkan. Before creating the instance, query the instance extensions available on the system. Add this code below the `appInfo` settings.

```cpp
const auto extensions = context.enumerateInstanceExtensionProperties(); // C API: vkEnumerateInstanceExtensionProperties
```

Each entry in the returned list contains an extension name. Below the query, loop through the list and print `extensionName`.

```cpp
std::cout << "Available instance extensions:\n";
for (const auto& extension : extensions)
{
    std::cout << "  " << extension.extensionName << '\n';
}
```

This only lists the available extensions. To use an extension, its name must be specified in the instance creation information. This lesson creates an instance without a window, so proceed without requesting extensions.

## Connect the application information to the creation information

Vulkan receives object settings through a creation information structure. Add `vk::InstanceCreateInfo` below the loop over the extensions.

```cpp
vk::InstanceCreateInfo createInfo{};
createInfo.pApplicationInfo = &appInfo;
```

`pApplicationInfo` now points to the `appInfo` structure prepared above. This structure can also specify the names of extensions and layers to enable. Neither is requested yet, so leave both counts at their default of 0. Layers are added in the next lesson.

## Create the instance and connect it to initialization

With the creation information ready, pass `context` and `createInfo` to create the instance. Add this directly below the creation information settings.

```cpp
instance = vk::raii::Instance(context, createInfo); // C API: vkCreateInstance
std::cout << "Instance created.\n";
```

The created object is stored in the `instance` member. If creation succeeds, the next line prints a message. This completes `createInstance()`.

Add a call to the existing `initVulkan()` function so that the creation function runs.

```cpp
void initVulkan()
{
    createInstance();
}
```

When the program starts, execution follows `main()`, `run()`, `initVulkan()`, and `createInstance()` in that order.

## Check when the instance is destroyed

A `vk::raii` object releases the Vulkan resource it manages when its lifetime ends. In the starter code, `main()` already contains an inner block around `app` and a final output statement. At the end of this block, members are destroyed in reverse declaration order. The instance is destroyed before `Instance destroyed.` is printed.

The next lesson adds validation layers to receive warnings and errors from Vulkan usage.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), build the `main.cpp` you have written. Change from the repository root to this example directory.

```sh
cd examples/01-instance
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target instance_example
.\build\Debug\main.exe
```

To run the completed example, run the CMake configuration command above, then build and run it with these commands.

```powershell
cmake --build build --config Debug --target instance_completed
.\build\completed\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target instance_example
./build/main
```

To run the completed example, run the CMake configuration command above, then build and run it with these commands.

```sh
cmake --build build --target instance_completed
./build/completed/main
```

### Expected output

Check that the available instance extensions are printed. Also check that the destruction message appears after the instance creation message.

After completing the code for this lesson, submit a screenshot of the terminal showing the output.

---

This document was translated and adapted from [Khronos Vulkan Tutorial: Base code](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/00_Base_code.html) and [Instance](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/01_Instance.html) for compute exercises. The original material and this document are licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
