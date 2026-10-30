# 02. Validation layers

[Previous: Create and destroy an instance](../01-instance/README.md) | [Next: Physical devices and queue families](../03-physical-devices/README.md) | [All tutorials](../../README.md#tutorials)

The Vulkan API itself performs limited error checking. During development, enable a validation layer to receive messages about invalid arguments or incorrect object usage order. The layer checks for problems, and the callback written in this lesson prints its messages to the console.

[main.cpp](main.cpp) starts with the instance creation code from lesson 01. Begin by checking whether the layer is available. Once published, the completed code is available in `examples/99-completed/02-validation-layers.cpp`.

## Define the layer name

This exercise uses `VK_LAYER_KHRONOS_validation`. Add a constant directly below `private` so that the same name is used to find and enable the layer.

```cpp
static constexpr const char* validationLayer = "VK_LAYER_KHRONOS_validation";
```

Use `std::strcmp()` to compare this string with the installed layer names. Add `<cstring>` to the headers at the top of the file.

```cpp
#include <cstring>
```

## Check whether the layer is installed

Knowing the layer's name is not enough; it must be installed on the system. Add an empty check function in the `private` section. It will return `true` if the layer is found and `false` otherwise.

```cpp
bool checkValidationLayerSupport()
{
}
```

First, retrieve the installed layers inside this function. Use `context` because the instance has not been created yet.

```cpp
const auto layers = context.enumerateInstanceLayerProperties(); // C API: vkEnumerateInstanceLayerProperties
```

Below this query, add a loop to examine each entry.

```cpp
for (const auto& layer : layers)
{
    // Check whether the name matches the required layer.
}
```

`layerName` contains each layer's name. Replace the comment inside the loop with the code below. `std::strcmp()` returns 0 when the strings match.

```cpp
if (std::strcmp(layer.layerName, validationLayer) == 0)
{
    return true;
}
```

The function returns as soon as it finds a matching name. If the loop finishes without a match, the required layer is missing. After the loop, print an error and return failure.

```cpp
std::cerr << "Required layer not found: " << validationLayer << '\n';
return false;
```

## Write a callback to receive messages

Prepare a function that Vulkan can call to report problems found by the layer. Add this empty callback in the `private` section.

```cpp
static VKAPI_ATTR VkBool32 VKAPI_CALL debugCallback(
    VkDebugUtilsMessageSeverityFlagBitsEXT severity,
    VkDebugUtilsMessageTypeFlagsEXT type,
    const VkDebugUtilsMessengerCallbackDataEXT* callbackData,
    void*)
{
}
```

`severity` is the message severity, and `type` is the message type. `callbackData` contains the identifier and message text. The last parameter accepts user data and is unused in this example.

This is a `static` member function with no `this` pointer, so it can be passed to Vulkan as an ordinary function pointer. `VKAPI_ATTR` and `VKAPI_CALL` match the calling conventions Vulkan requires.

First, add code to print the severity and type. Converting the C API types to Vulkan-Hpp types allows `vk::to_string()` to print their names.

```cpp
std::cerr << "[debug "
          << vk::to_string(static_cast<vk::DebugUtilsMessageSeverityFlagBitsEXT>(severity))
          << ' ' << vk::to_string(vk::DebugUtilsMessageTypeFlagsEXT(type)) << "] ";
```

Continue by printing the message identifier and text. Use `"unnamed"` if the identifier pointer is null.

```cpp
std::cerr << (callbackData->pMessageIdName ? callbackData->pMessageIdName : "unnamed")
          << ": " << callbackData->pMessage << '\n';
```

This callback lets the Vulkan call continue after printing the message. Return `VK_FALSE` at the end of the function.

```cpp
return VK_FALSE;
```

## Select which messages to receive

With the callback ready, select the message severities and types to receive. Add a function in the `private` section that returns the creation information, so the same settings can be used both during and after instance creation.

```cpp
static vk::DebugUtilsMessengerCreateInfoEXT makeDebugMessengerCreateInfo()
{
}
```

First, create the information structure inside the function and select warnings and errors.

```cpp
vk::DebugUtilsMessengerCreateInfoEXT createInfo{};
createInfo.messageSeverity = vk::DebugUtilsMessageSeverityFlagBitsEXT::eWarning
                           | vk::DebugUtilsMessageSeverityFlagBitsEXT::eError;
```

`|` is the bitwise OR operator. Combining the flags this way selects messages with either Warning or Error severity.

Below that, select the message types. Receive general messages, API validation messages, and performance messages.

```cpp
createInfo.messageType = vk::DebugUtilsMessageTypeFlagBitsEXT::eGeneral
                       | vk::DebugUtilsMessageTypeFlagBitsEXT::eValidation
                       | vk::DebugUtilsMessageTypeFlagBitsEXT::ePerformance;
```

Connect the function that handles messages matching these conditions. Set the previously written `debugCallback` and return the creation information.

```cpp
createInfo.setPfnUserCallback(debugCallback);
return createInfo;
```

## Check support before creating the instance

Use the layer check in the existing `createInstance()` function. Change its return type from `void` to `bool` so it can report failure when the layer is unavailable. Keep the existing body and insert this code at its beginning.

```cpp
if (!checkValidationLayerSupport())
{
    return false;
}
```

If the layer is missing, return without creating an instance. Keep the `appInfo` settings below this code.

Registering the callback and submitting messages also require the `VK_EXT_debug_utils` extension. Directly below the existing extension query and before the loop, add a variable to store the search result.

```cpp
bool debugUtilsAvailable = false;
```

Add this check inside the existing `for (const auto& extension : extensions)` loop.

```cpp
if (std::strcmp(extension.extensionName, VK_EXT_DEBUG_UTILS_EXTENSION_NAME) == 0)
{
    debugUtilsAvailable = true;
}
```

When an extension name matches the required name, mark it as available. If the variable remains `false` after all extensions have been checked, initialization cannot continue. Add this condition after the loop.

```cpp
if (!debugUtilsAvailable)
{
    std::cerr << "Required instance extension not found: VK_EXT_debug_utils\n";
    return false;
}
```

## Enable the layer and extension for the instance

Checking support does not enable a feature. In the same `createInstance()` function, find the `vk::InstanceCreateInfo` settings. After the existing `createInfo.pApplicationInfo = &appInfo;`, add the layer count and name.

```cpp
createInfo.enabledLayerCount = 1;
createInfo.ppEnabledLayerNames = &validationLayer;
```

The extension also needs a name list and count. Add an array immediately before the existing `vk::InstanceCreateInfo createInfo{};` declaration.

```cpp
const char* requiredExtensions[] = {VK_EXT_DEBUG_UTILS_EXTENSION_NAME};
```

Connect this array directly below the layer settings.

```cpp
createInfo.enabledExtensionCount = 1;
createInfo.ppEnabledExtensionNames = requiredExtensions;
```

Messages during instance creation and destruction should also be received. After the `requiredExtensions` array and before the `createInfo` declaration, prepare the callback settings defined earlier.

```cpp
auto debugCreateInfo = makeDebugMessengerCreateInfo();
```

The instance creation information's `pNext` field connects additional settings structures. Add this line below the extension settings to use the callback settings during instance creation and destruction.

```cpp
createInfo.pNext = &debugCreateInfo;
```

Keep the following `instance = vk::raii::Instance(context, createInfo);` line. Directly after creating the instance, print the enabled layer name and return success at the end of the function.

```cpp
std::cout << "Validation layer enabled: " << validationLayer << '\n';
return true;
```

## Receive messages after instance creation

The settings connected through `pNext` apply during instance creation and destruction. To receive messages from Vulkan calls throughout the instance's lifetime, create a separate debug messenger object.

First, add a member to hold the debug messenger directly below the existing `instance` member.

```cpp
vk::raii::DebugUtilsMessengerEXT debugMessenger = nullptr;
```

RAII members are destroyed in reverse declaration order. Placing the member here ensures that the debug messenger is destroyed before the instance.

Add an empty function in the `private` section to create the debug messenger.

```cpp
bool setupDebugMessenger()
{
}
```

Inside the function, create the debug messenger with the settings prepared earlier and store it in the member.

```cpp
debugMessenger = instance.createDebugUtilsMessengerEXT(makeDebugMessengerCreateInfo()); // C API: vkCreateDebugUtilsMessengerEXT
```

Below that, print a message to confirm creation and return success.

```cpp
std::cout << "Debug messenger created.\n";
return true;
```

## Send a test message to the callback

Code without errors may produce no callback output. Submit a message directly to check that the registered callback is called. Add an empty function in the `private` section.

```cpp
void submitDebugMessage()
{
}
```

First, set the identifier and message text inside the function.

```cpp
vk::DebugUtilsMessengerCallbackDataEXT callbackData{};
callbackData.pMessageIdName = "tutorial.callback-check";
callbackData.pMessage = "Application-injected callback check; this is not a validation error.";
```

Below that, submit the message. Set the severity to Warning and the type to General so it passes the filters configured earlier.

```cpp
instance.submitDebugUtilsMessageEXT( // C API: vkSubmitDebugUtilsMessageEXT
    vk::DebugUtilsMessageSeverityFlagBitsEXT::eWarning,
    vk::DebugUtilsMessageTypeFlagBitsEXT::eGeneral,
    callbackData);
```

This is a test message submitted by the application, not an error found by the validation layer. Use it only in this lesson; remove the function and its call from lesson 03 onward.

## Connect initialization and execution

With the required functions written, connect them to the execution flow. Replace the existing `initVulkan()` with the following.

```cpp
bool initVulkan()
{
    return createInstance() && setupDebugMessenger();
}
```

If the left side of `&&` is `false`, the right side is not evaluated. A failed layer or extension check therefore prevents debug messenger creation.

The test message must also be skipped if initialization fails. Change the return type of the existing `run()` function in the `public` section to `bool`, and replace its `initVulkan();` call with this condition.

```cpp
if (!initVulkan())
{
    return false;
}
```

After the condition and before the function ends, submit the message and return success.

```cpp
submitDebugMessage();
return true;
```

Finally, `main()` must handle the execution result. Keep `ComputeApplication app;` in the existing inner block and replace the single `app.run();` line with this condition.

```cpp
if (!app.run())
{
    return 1;
}
```

Failure produces exit code 1. On success, leaving the inner block destroys the debug messenger and then the instance. The starter code already includes the output statement after that block.

Execution now checks layer support, creates the instance, creates the debug messenger, and submits the test message in that order. Starting in the next lesson, use this setup to receive messages from actual Vulkan usage while selecting a physical device.

## Build and run

After completing [Environment settings](../00-env-settings/README.md), change from the repository root to this example directory.

```sh
cd examples/02-validation-layers
```

### Windows

Build and run with the Visual Studio generator.

```powershell
cmake -S . -B build
cmake --build build --config Debug --target validation_layers
.\build\Debug\main.exe
```

If the completed example is available, run the CMake configuration command above, then build and run it with these commands.

```powershell
cmake --build build --config Debug --target validation_layers_completed
.\build\completed\Debug\main.exe
```

### Linux

Build and run with Makefiles or Ninja.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target validation_layers
./build/main
```

If the completed example is available, run the CMake configuration command above, then build and run it with these commands.

```sh
cmake --build build --target validation_layers_completed
./build/completed/main
```

### Expected output

Check the enabled layer name and the debug messenger creation message. If the submitted message reaches the callback, its severity, type, and text should appear together.

After completing the code for this lesson, submit a screenshot of the terminal showing the output.

---

This document was translated and adapted from [Khronos Vulkan Tutorial: Validation layers](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/02_Validation_layers.html) for compute exercises. The original material and this document are licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
