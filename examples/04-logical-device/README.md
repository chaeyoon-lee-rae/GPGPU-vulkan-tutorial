# 04. Logical device and queues

[Previous: Physical devices and queue families](../03-physical-devices/README.md) | [Next: Buffers and memory](../05-buffers-memory/README.md) | [All tutorials](../../README.md#tutorials)

The previous lesson selected a physical device and compute queue family. Now create a logical device to request work on that device. Specify the features and number of queues to use, then retrieve a handle to the created queue.

Open [main.cpp](main.cpp) and follow the steps below. This lesson does not include a completed source file.

## Add members to hold the device and queue

The logical device and queue are used after initialization. In the `private` section, add these members below `computeQueueFamilyIndex`.

```cpp
vk::raii::Device device = nullptr;
vk::raii::Queue computeQueue = nullptr;
```

`device` holds the logical device, and `computeQueue` holds the queue for submitting commands. Initialize both objects to `nullptr` because they have not been created yet.

Add an empty creation function at the end of the class's `private` section. Start by preparing the queue creation information inside it.

```cpp
bool createLogicalDevice()
{
}
```

## Specify the queue to create

The `computeQueueFamilyIndex` saved in lesson 03 identifies the family to use. To use a queue from that family, request it when creating the logical device. Start with one queue.

First, set the priority of the requested queue. Add this as the first line of `createLogicalDevice()`.

```cpp
const float queuePriority = 1.0f;
```

The priority is a value from 0.0 to 1.0 and must be specified even for a single queue.

Next, use `vk::DeviceQueueCreateInfo` to specify the family and the number of queues to request.

```cpp
vk::DeviceQueueCreateInfo queueCreateInfo{};
queueCreateInfo.queueFamilyIndex = computeQueueFamilyIndex;
queueCreateInfo.queueCount = 1;
queueCreateInfo.pQueuePriorities = &queuePriority;
```

`queueFamilyIndex` identifies the family found in the previous lesson, and `queueCount` is the number of queues requested from that family. `pQueuePriorities` needs one priority per requested queue. Since only one queue is requested here, pass the address of `queuePriority`.

## Connect the features and creation information

With the queue specified, select the features to enable on the device. The previous lesson's `getFeatures()` only queried support. Optional features must be requested during device creation, even when the device supports them.

This exercise needs no optional features. Add an empty feature structure below the queue creation information.

```cpp
vk::PhysicalDeviceFeatures enabledFeatures{};
```

Initializing with `{}` disables every feature in this structure. For example, even if the previous lesson reported support for `shaderFloat64`, it is not enabled unless requested here.

Create the logical device creation information and connect the prepared queue information.

```cpp
vk::DeviceCreateInfo createInfo{};
createInfo.queueCreateInfoCount = 1;
createInfo.pQueueCreateInfos = &queueCreateInfo;
```

`queueCreateInfoCount` is the **number of queue creation information structures**, not the number of queues. Set it to 1 because there is one structure for one family, and pass that structure's address to `pQueueCreateInfos`.

Below that, connect the structure specifying which features to enable.

```cpp
createInfo.pEnabledFeatures = &enabledFeatures;
```

This compute exercise does not display images in a window, so no device extensions such as the swapchain extension are added.

## Create the device and retrieve the queue

The creation information is ready. In the same function, pass the selected physical device and creation information to create the logical device.

```cpp
device = vk::raii::Device(physicalDevice, createInfo); // C API: vkCreateDevice
```

This also creates the queue requested through `queueCreateInfo`. The program still needs to retrieve its handle. Store that handle in `computeQueue` next.

```cpp
const uint32_t queueIndex = 0;
computeQueue = vk::raii::Queue(device, computeQueueFamilyIndex, queueIndex); // C API: vkGetDeviceQueue
```

The second argument is the family index, and the third is the **queue index within that family**. Set `queueIndex` to 0 because one queue was requested. Use this value both to retrieve the queue and to print the result. This call retrieves the previously created queue; it does not create another one.

At the end of the function, print the creation result and return success.

```cpp
std::cout << "Logical device created.\n"
          << "Compute queue ready: family " << queueCreateInfo.queueFamilyIndex
          << ", queue " << queueIndex << ".\n";
return true;
```

> [!NOTE]
> **Questions to consider**
>
> If two operations are submitted to the same queue in order, must the first finish completely before the second begins? Compare execution ordering and synchronization in CUDA streams and Vulkan queues.

## Connect the initialization steps

With `createLogicalDevice()` complete, connect it to initialization. The physical device and queue family must be selected first, so replace the existing `initVulkan()` with the following.

```cpp
bool initVulkan()
{
    return createInstance()
        && setupDebugMessenger()
        && pickPhysicalDevice()
        && createLogicalDevice();
}
```

If an earlier step fails, the function after `&&` is not called. The logical device is created only after device selection succeeds.

When `app` is destroyed, its RAII members are cleaned up in reverse declaration order. Queues are released when the logical device is destroyed, so no separate queue destruction function is called.

The device and queue are now ready. The next lesson uses this device to create buffers and memory for computation.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), change from the repository root to this example directory.

```sh
cd examples/04-logical-device
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target logical_device_example
.\build\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target logical_device_example
./build/main
```

### Expected output

Check that the logical device was created and the queue handle was retrieved. Verify that the printed family index matches the index saved during device selection.

After completing the code for this lesson, submit a screenshot of the terminal showing the output.

---

This document was translated and adapted from [Khronos Vulkan Tutorial: Logical device and queues](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/04_Logical_device_and_queues.html) for compute exercises. The original material and this document are licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
