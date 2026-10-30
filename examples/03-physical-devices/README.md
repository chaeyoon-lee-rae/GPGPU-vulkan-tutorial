# 03. Physical devices and queue families

[Previous: Validation layers](../02-validation-layers/README.md) | [Next: Logical device and queues](../04-logical-device/README.md) | [All tutorials](../../README.md#tutorials)

After initializing the Vulkan library through an instance, select a physical device for computation. A computer may have multiple GPUs, and their supported features may differ. This tutorial uses the first device that supports Vulkan 1.3 and can process compute commands.

Open [main.cpp](main.cpp) and work through the steps, starting with retrieving the device list. Once published, the completed code is available in `examples/99-completed/03-physical-devices.cpp`.

## Prepare to select a device

The `submitDebugMessage()` function in lesson 02 checked that the callback was called. It is no longer used from this lesson onward. If you are extending your lesson 02 code directly, remove that function and its call in `run()`. Keep the validation layer and debug messenger to receive warnings and errors from Vulkan usage.

The selected device must remain available after the function returns. In the `private` section, directly below `debugMessenger`, add members to hold the device and queue family index.

```cpp
vk::raii::PhysicalDevice physicalDevice = nullptr;
uint32_t computeQueueFamilyIndex = 0;
```

`physicalDevice` stores the selected device. `computeQueueFamilyIndex` is the index of the queue family to use on that device; it will be set after examining the queue families.

Also add the headers needed for this lesson at the top of the file. `<stdint.h>` provides the integer type for the index, `<optional>` is used for the queue search result, and `<utility>` is used to move the device into the member.

```cpp
#include <stdint.h>
#include <optional>
#include <utility>
```

## Retrieve the physical device list

First, add an empty device selection function in the `private` section. It will return `true` if a device is selected and `false` otherwise.

```cpp
bool pickPhysicalDevice()
{
}
```

Inside the function, call `enumeratePhysicalDevices()` to retrieve the physical devices accessible through this instance.

```cpp
auto physicalDevices = instance.enumeratePhysicalDevices(); // C API: vkEnumeratePhysicalDevices
```

Selection cannot proceed if no devices are listed. Directly below the query, check whether the list is empty, print an error, and return failure.

```cpp
if (physicalDevices.empty())
{
    std::cerr << "No Vulkan physical devices found.\n";
    return false;
}
```

Even if devices are available, they may not all meet the requirements. Add a loop below the `if` statement to check each device. `candidate` is the device being examined.

```cpp
for (auto& candidate : physicalDevices)
{
    // Check this device's API version and queue families.
    // Store the selected device if it meets the requirements.
}
```

The function will return success inside the loop when it finds a suitable device. Reaching the end of the loop therefore means that no suitable device was found. Add this code below the loop's closing brace, before the end of `pickPhysicalDevice()`.

```cpp
std::cerr << "No Vulkan 1.3 physical device with a compute queue family found.\n";
return false;
```

## Check the API version

Now fill in the loop body. First, retrieve the device properties. The information from `getProperties()` includes the device name, device type, and supported API version.

```cpp
const auto properties = candidate.getProperties(); // C API: vkGetPhysicalDeviceProperties
```

This tutorial uses Vulkan 1.3, so skip devices that only support an earlier version. Add this condition directly below the property query.

```cpp
if (properties.apiVersion < VK_API_VERSION_1_3)
{
    continue;
}
```

`continue` skips the remaining checks for this device and moves to the next one. If the version check passes, the next step is to find a queue family that can process compute commands.

## Find a compute queue family

In Vulkan, commands for the GPU are submitted to a queue. A queue family is a group of queues that support the same types of commands. Some families support both graphics and compute commands, while others may handle transfer operations. Find a family that can process compute commands.

Outside `pickPhysicalDevice()`, add a queue family search function in the same `private` section. This function takes a device to examine and returns the index of a suitable family. The search may find no match, so use `std::optional` to express whether a value is present.

```cpp
static std::optional<uint32_t> findComputeQueueFamily(
    const vk::raii::PhysicalDevice& candidate)
{
}
```

First, retrieve the device's queue families inside the function.

```cpp
const auto queueFamilies = candidate.getQueueFamilyProperties(); // C API: vkGetPhysicalDeviceQueueFamilyProperties
```

Each entry contains the family's queue count and supported command types. The array index is needed when a suitable family is found, so iterate by index. Add this loop below the query.

```cpp
for (uint32_t index = 0; index < queueFamilies.size(); ++index)
{
    const auto& family = queueFamilies[index];
    // Check whether this family can process compute commands.
}
```

`queueCount` is the number of queues in this family, and `queueFlags` describes the supported command types. A family is suitable if it has at least one queue and supports compute commands. Replace the comment inside the loop with this code.

```cpp
if (family.queueCount > 0 && (family.queueFlags & vk::QueueFlagBits::eCompute))
{
    return index;
}
```

`queueFlags & vk::QueueFlagBits::eCompute` checks whether the support flags include compute. Returning the index of the first suitable family ends the search.

If no suitable family is found, return `std::nullopt` to indicate that there is no result. Add this line after the loop, at the end of the function.

```cpp
return std::nullopt;
```

> [!NOTE]
> **Questions to consider**
>
> What are queues and queue families, and how are they related? Use Vulkan Hardware Capability Viewer to examine the supported operations and queue count of each queue family on your GPU.

## Store a suitable device

With the queue search function complete, return to the device loop in `pickPhysicalDevice()`. Call the new function below the `if` statement that checks the API version.

```cpp
const auto queueFamily = findComputeQueueFamily(candidate);
if (!queueFamily)
{
    continue;
}
```

If there is no search result, `if (!queueFamily)` advances to the next device. A result means that this device passes both the version and queue requirements.

Next, store the selected device and queue family index in the members. Add this code after the new `if` statement, inside the same loop.

```cpp
physicalDevice = std::move(candidate);
computeQueueFamilyIndex = *queueFamily;
return true;
```

`std::move(candidate)` moves the RAII device object from the list into the member. `*queueFamily` retrieves the index from the search result. After storing it, return success. The function can now select a suitable device or report that none was found.

## Examine the selected device

Print information about the selected device. Add an empty output function in the `private` section.

```cpp
void printDeviceInfo()
{
}
```

First, retrieve the selected device's properties and print its name and type. Selection examined `candidate`; this function uses the `physicalDevice` stored in the member.

```cpp
const auto properties = physicalDevice.getProperties(); // C API: vkGetPhysicalDeviceProperties
std::cout << "Selected physical device: " << properties.deviceName << '\n'
          << "Device type: " << vk::to_string(properties.deviceType) << '\n';
```

Below that, print the supported API version. `apiVersion` packs the version into one integer, so use the macros to extract the major, minor, and patch values.

```cpp
std::cout << "Device API version: "
          << VK_API_VERSION_MAJOR(properties.apiVersion) << '.'
          << VK_API_VERSION_MINOR(properties.apiVersion) << '.'
          << VK_API_VERSION_PATCH(properties.apiVersion) << '\n';
```

Also check the selected queue family index and its queue count. Use the saved `computeQueueFamilyIndex` to find the entry in the queue family list. Add this code below the version output.

```cpp
const auto queueFamilies = physicalDevice.getQueueFamilyProperties(); // C API: vkGetPhysicalDeviceQueueFamilyProperties
std::cout << "Compute queue family index: " << computeQueueFamilyIndex << '\n'
          << "Queues in selected family: " << queueFamilies[computeQueueFamilyIndex].queueCount << '\n';
```

Use `getFeatures()` to examine optional device features. For example, `shaderFloat64` indicates support for 64-bit floating-point operations in shaders. Add this code to print whether it is supported.

```cpp
const auto features = physicalDevice.getFeatures(); // C API: vkGetPhysicalDeviceFeatures
std::cout << "shaderFloat64 supported (not enabled): " << (features.shaderFloat64 ? "yes" : "no") << '\n';
```

This only queries support. To use the feature, request it when creating the logical device in the next lesson.

Finally, examine the size limits for computation. Compute shader invocations are grouped into workgroups. Both the group count and the size within a group are specified along the x, y, and z axes. The device's `limits` contains the maximum allowed size along each axis.

```cpp
const auto& limits = properties.limits;
std::cout << "maxComputeWorkGroupCount: " << limits.maxComputeWorkGroupCount[0] << ", "
          << limits.maxComputeWorkGroupCount[1] << ", " << limits.maxComputeWorkGroupCount[2] << '\n'
          << "maxComputeWorkGroupSize: " << limits.maxComputeWorkGroupSize[0] << ", "
          << limits.maxComputeWorkGroupSize[1] << ", " << limits.maxComputeWorkGroupSize[2] << '\n';
```

`maxComputeWorkGroupCount` limits the number of groups along each axis in one dispatch. `maxComputeWorkGroupSize` limits the number of invocations along each axis within one group.

Even when each dimension meets its limit, a separate limit applies to the total number of invocations in one group. Print that limit, `maxComputeWorkGroupInvocations`, followed by `maxStorageBufferRange`, the maximum byte range that a storage buffer descriptor can specify.

```cpp
std::cout << "maxComputeWorkGroupInvocations: " << limits.maxComputeWorkGroupInvocations << '\n'
          << "maxStorageBufferRange: " << limits.maxStorageBufferRange << " bytes\n";
```

These values are used later to choose buffer sizes and shader execution dimensions. See [VkPhysicalDeviceLimits](https://docs.vulkan.org/refpages/latest/refpages/source/VkPhysicalDeviceLimits.html) for detailed definitions.

The output function is complete. Return to `pickPhysicalDevice()` and add a call to `printDeviceInfo()` immediately after storing the device and index, before `return true;`. The end of the loop now looks like this.

```cpp
physicalDevice = std::move(candidate);
computeQueueFamilyIndex = *queueFamily;
printDeviceInfo();
return true;
```

## Connect the initialization steps

Connect device selection to the initialization flow. Replace the existing `initVulkan()` with the following to select a device after creating the instance and debug messenger.

```cpp
bool initVulkan()
{
    return createInstance() && setupDebugMessenger() && pickPhysicalDevice();
}
```

If an earlier step fails, the function after `&&` is not called. If device selection also succeeds, the function returns `true`, and the selected device remains in the member for the next step.

The physical device and queue family have now been selected. The next lesson uses this information to create a logical device and retrieve the queue handle.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), change from the repository root to this example directory.

```sh
cd examples/03-physical-devices
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target physical_devices
.\build\Debug\main.exe
```

If the completed example is available, run the CMake configuration command above, then build and run it with these commands.

```powershell
cmake --build build --config Debug --target physical_devices_completed
.\build\completed\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target physical_devices
./build/main
```

If the completed example is available, run the CMake configuration command above, then build and run it with these commands.

```sh
cmake --build build --target physical_devices_completed
./build/completed/main
```

### Expected output

Check the selected device name, API version, and queue family index. Compare the supported features and limits with the results queried from your own device.

After completing the code for this lesson, submit a screenshot of the terminal showing the output.

---

This document was adapted for compute exercises from study notes translated from [Khronos Vulkan Tutorial: Physical devices and queue families](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/03_Physical_devices_and_queue_families.html). The original material and this document are licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
