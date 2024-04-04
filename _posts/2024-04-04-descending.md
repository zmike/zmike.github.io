# Into The Spiral of Madness

I know what you're all thinking: there have not been enough blog posts this year. As always, my highly intelligent readers are right, and as always, you're just gonna have to live with that because I'm not changing the way anything works. SGC happens when it happens.

And today. As it snows in April. SGC. Is. Happening.

Let's begin.

# In The Beginning, A Favor Was Asked
I was sitting at my battlestation doing some very ordinary **REDACTED** work for **REDACTED** and friend of the blog, Samuel "Shader Objects" Pitoiset (he has legally changed his name, please be respectful), came to me with a simple request. He wanted to be able to enable [VK_EXT_shader_object](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_shader_object.html) for the radv-zink jobs in mesa CI as the final part of his year-long bringup for the extension. This meant that all the tests passing without shader objects needed to also pass with shader objects.

This should've been easy; it was over a year ago that the Khronos blog famously and confusingly [announced](https://www.khronos.org/blog/you-can-use-vulkan-without-pipelines-today) that pipelines were dead and nobody should ever use them again (paraphrased).