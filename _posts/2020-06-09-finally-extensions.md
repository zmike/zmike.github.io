---
published: true
---
## What are extensions?

Extensions are non-core parts of a spec which provide additional features. `xfb` is an extension in Vulkan. In order to use it, a number of small changes are required.

## What are features?
Features in Vulkan inform an application (or Zink) what the underlying driver supports. In order to enable and use an extension, both the extension and feature must be enabled.

## Step 1: extension detection
To begin, let's check out the significant parts of `zink_screen.c` that got modified, all in `zink_internal_create_screen()`, which creates a `struct zink_screen*` object, a subclass of `struct pipe_screen*`.

```c
VkExtensionProperties *extensions;
vkEnumerateDeviceExtensionProperties(screen->pdev, NULL,
                                     &num_extensions, extensions);

for (uint32_t  i = 0; i < num_extensions; ++i) {
   if (!strcmp(extensions[i].extensionName,
               VK_EXT_TRANSFORM_FEEDBACK_EXTENSION_NAME))
      have_tf_ext = true;

}
```
There's already some code doing this in Zink, so it's a simple case of plugging in another `strcmp` to check for `VK_EXT_TRANSFORM_FEEDBACK_EXTENSION_NAME`.

## Step 2: feature detection
```c
VkPhysicalDeviceFeatures2 feats = {};
VkPhysicalDeviceTransformFeedbackFeaturesEXT tf_feats = {};

feats.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2;
if (have_tf_ext) {
   tf_feats.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_TRANSFORM_FEEDBACK_FEATURES_EXT;
   tf_feats.pNext = feats.pNext;
   feats.pNext = &tf_feats;
}
vkGetPhysicalDeviceFeatures2(screen->pdev, &feats);
```
Again, there's already some code in Zink to handle feature detection, so this just requires plugging in the `xfb` feature parts.

## Step 3: property checking
In addition to extensions and features, there's also properties, which provide things like device-specific limits for various capabilities that need to be checked in order to avoid requesting resources that the gpu hardware can't provide.

```c
if (have_tf_ext && tf_feats.transformFeedback)
   screen->have_EXT_transform_feedback = true;

VkPhysicalDeviceProperties2 props = {};
props.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2;
if (screen->have_EXT_transform_feedback) {
   screen->tf_props.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_TRANSFORM_FEEDBACK_PROPERTIES_EXT;
   screen->tf_props.pNext = NULL;
   props.pNext = &screen->tf_props;
}
vkGetPhysicalDeviceProperties2(screen->pdev, &props);
```
This was another easy plug-in, since it's just the `xfb` parts that need to be added.

## Finishing touches
That's more or less it. The `xfb` extension name needs to get added into `VkDeviceCreateInfo::ppEnabledExtensionNames` when it's passed to `vkCreateDevice()` a bit later, but `xfb` is now fully activated if the driver supports it.

Lastly, the relevant `enum pipe_cap` members need to be handled in `zink_get_param()` so that gallium recognizes the newly-activated capabilities:

```c
case PIPE_CAP_MAX_STREAM_OUTPUT_BUFFERS:
   return screen->have_EXT_transform_feedback ? screen->tf_props.maxTransformFeedbackBuffers : 0;
case PIPE_CAP_STREAM_OUTPUT_PAUSE_RESUME:
case PIPE_CAP_STREAM_OUTPUT_INTERLEAVE_BUFFERS:
   return 1;
```

## And that's it
Everything worked perfectly on the first try, and there were absolutely no issues whatsoever, because working on mesa is just that easy.
