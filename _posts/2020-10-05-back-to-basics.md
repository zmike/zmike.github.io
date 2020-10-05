---
published: false
---
## Healthier Blogging

I took some time off to focus on making the numbers go up, but if I only do that sort of junk food style blogging with images, and charts, and *benchmarks* then we might all stop learning things, and certainly I won't be transferring any knowledge between the coding part of my brain and the speaking part, so really we're all losers at that point.

In other words, let's get back to going extra deep into some code and doing some long form patch review.

## Descriptor Management
First: what are descriptors?

Descriptors are, in short, when you feed a buffer or an image (+sampler) into a shader. In OpenGL, this is all handled for the user behind the scenes with e.g., a simple `glGenBuffers()` -> `glBindBuffer()` -> `glBufferData()` for an attached buffer. For a gallium-based driver, this example case will trigger the `pipe_context::set_shader_buffers` hook at draw time to inform the driver that a ssbo has been attached, and then the driver can link it up with the GPU.

Things are a bit different in Vulkan. There's [an entire chapter](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#descriptorsets) of the spec devoted to explaining how descriptors work in great detail, but the important details for the zink case are:
* each descriptor must have created for it a `binding` value which is unique for the given descriptor set
* each descriptor must have a Vulkan descriptor type, as converted from its OpenGL type
* each descriptor must also potentially expand to its full array size (image descriptor types only)

Additionally, while organizing and initializing all these descriptor sets, zink has to track all the resources used and guarantee their lifetimes exceed the lifetimes of the batch they're being submitted with.

To handle this, zink has *an amount* of code. In the current state of the repo, it's [about 40 lines](https://gitlab.freedesktop.org/mesa/mesa/-/blob/08d51e92aee0cddc5ad567dddd432cc4016a4570/src/gallium/drivers/zink/zink_draw.c#L293).

However...

[![meme-update_descriptors.png]({{site.url}}/assets/meme-update_descriptors.png)]({{site.url}}/assets/meme-update_descriptors.png)

In the state of my branch that I'm going to be working off of for the next few blog posts, the function for handling descriptor updates is [304 lines](https://gitlab.freedesktop.org/zmike/mesa/-/blob/c1fc05e5e48b5a259e3d53cbaf002505047933b6/src/gallium/drivers/zink/zink_draw.c#L279). This is the increased amount of code that's required to handle (almost) all GL descriptor types in a way that's reasonably reliable.

Is it a great design decision to have a function that large?

Probably not.

But I decided to write it all out first before I did any refactoring so that I could avoid having to incrementally refactor my first attempt at refactoring, which would waste lots of time.

Also, memes.

## How Does This Work?
The idea behind the latter version of the implementation that I linked is as follows:
* iterate over all the shader stages
* iterate over the bindings for the shader
* for each binding:
  * fill in the Vulkan buffer/image info struct
  * for all resources used by the binding:
    * add tracking for the underlying resource(s) for lifetime management
    * flag a pipeline barrier for the resource(s) based on usage*
* fill in the [VkWriteDescriptorSet](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkWriteDescriptorSet.html) struct for the binding

\*Then merge and deduplicate all the accumulated pipeline barriers and apply only those which induce a layout or access change in the resource to avoid over-applying barriers.

As I mentioned in a previous post, zink then applies these descriptors to a newly-allocated, max-size descriptor set object from an allocator pool located on the batch object. Every time a draw command is triggered, a new [VkDescriptorSet](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDescriptorSet.html) is allocated and updated using these steps.

## First Level Refactoring Target
As I touched on briefly in a previous post, the first change to make here in improving descriptor handling is to move the descriptor pools to the program objects. This lets zink create smaller descriptor sets which are likely going to end up using less memory than these giant ones. Here's the code used for creating descriptor sets prior to refactoring:
```c
#define ZINK_BATCH_DESC_SIZE 1000
VkDescriptorPoolSize sizes[] = {
   {VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,         ZINK_BATCH_DESC_SIZE},
   {VK_DESCRIPTOR_TYPE_UNIFORM_TEXEL_BUFFER,   ZINK_BATCH_DESC_SIZE},
   {VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, ZINK_BATCH_DESC_SIZE},
   {VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER,   ZINK_BATCH_DESC_SIZE},
   {VK_DESCRIPTOR_TYPE_STORAGE_IMAGE,          ZINK_BATCH_DESC_SIZE},
   {VK_DESCRIPTOR_TYPE_STORAGE_BUFFER,         ZINK_BATCH_DESC_SIZE},
};
VkDescriptorPoolCreateInfo dpci = {};
dpci.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
dpci.pPoolSizes = sizes;
dpci.poolSizeCount = ARRAY_SIZE(sizes);
dpci.flags = 0;
dpci.maxSets = ZINK_BATCH_DESC_SIZE;
vkCreateDescriptorPool(screen->dev, &dpci, 0, &batch->descpool);
```
Here, all the used descriptor types allocate `ZINK_BATCH_DESC_SIZE` descriptors in the pool, and there are `ZINK_BATCH_DESC_SIZE` sets in the pool. There's a bug here, which is that really the descriptor types should have `ZINK_BATCH_DESC_SIZE * ZINK_BATCH_DESC_SIZE` descriptors to avoid oom-ing the pool in the event that allocated sets actually use that many descriptors, but ultimately this is irrelevant, as we're only ever allocating 1 set at a time due to zink flushing multiple times per draw anyway.

Ideally, however, it would be better to avoid this. The majority of draw cases use much, much smaller descriptor sets which only have 1-5 descriptors total, so allocating 6 * 1000 for a pool is roughly 6 * 1000 more than is actually needed for every set.

The other downside of this strategy is that by creating these giant, generic descriptor sets, it becomes impossible to know what's actually in a given set without attaching considerable metadata to it, which makes reusing sets without modification (i.e., caching) a fair bit of extra work. Yes, I know I said bucket allocation was faster, but I also said I believe in letting the best idea win, and it doesn't really seem like doing full updates every draw should be faster, does it? But I'll get to that in another post.

## Descriptor Migration
When creating a `struct zink_program` (which is the struct that contains all the shaders), zink creates a [VkDescriptorSetLayout](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDescriptorSetLayout.html) object for describing the descriptor set layouts that will be allocated from the pool. This means zink is allocating giant, generic descriptor set pools and then allocating (almost certainly) very, very small sets, which means the driver ends up with this giant memory balloon of unused descriptors allocated in the pool that can never be used.

A better idea for this would be to create descriptor pools which precisely match the layout for which they'll be allocating descriptor sets, as this means there's no memory ballooning, even if it does end up being more pool objects.

Here's the current function for creating the layout object for a program:
```c
static VkDescriptorSetLayout
create_desc_set_layout(VkDevice dev,
                       struct zink_shader *stages[ZINK_SHADER_COUNT],
                       unsigned *num_descriptors)
{
   VkDescriptorSetLayoutBinding bindings[PIPE_SHADER_TYPES * (PIPE_MAX_CONSTANT_BUFFERS + PIPE_MAX_SHADER_SAMPLER_VIEWS + PIPE_MAX_SHADER_BUFFERS + PIPE_MAX_SHADER_IMAGES)];
   int num_bindings = 0;

   for (int i = 0; i < ZINK_SHADER_COUNT; i++) {
      struct zink_shader *shader = stages[i];
      if (!shader)
         continue;

      VkShaderStageFlagBits stage_flags = zink_shader_stage(pipe_shader_type_from_mesa(shader->nir->info.stage));
```
This function is called for both the graphics and compute pipelines, and for the latter, only a single shader object is passed, meaning that `i` is purely for iterating the maximum number of shaders and not descriptive of the current shader being processed.
```c
      for (int j = 0; j < shader->num_bindings; j++) {
         assert(num_bindings < ARRAY_SIZE(bindings));
         bindings[num_bindings].binding = shader->bindings[j].binding;
         bindings[num_bindings].descriptorType = shader->bindings[j].type;
         bindings[num_bindings].descriptorCount = shader->bindings[j].size;
         bindings[num_bindings].stageFlags = stage_flags;
         bindings[num_bindings].pImmutableSamplers = NULL;
         ++num_bindings;
      }
   }
```
This iterates over the bindings in a given shader, setting up the various values required by the layout creation struct using the values stored to the shader struct using the code in `zink_compiler.c`.
```c
   *num_descriptors = num_bindings;
   if (!num_bindings) return VK_NULL_HANDLE;
```
If this program has no descriptors at all, then this whole thing can become a no-op, and descriptor updating can be skipped for draws which use this program.
```c
   VkDescriptorSetLayoutCreateInfo dcslci = {};
   dcslci.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
   dcslci.pNext = NULL;
   dcslci.flags = 0;
   dcslci.bindingCount = num_bindings;
   dcslci.pBindings = bindings;

   VkDescriptorSetLayout dsl;
   if (vkCreateDescriptorSetLayout(dev, &dcslci, 0, &dsl) != VK_SUCCESS) {
      debug_printf("vkCreateDescriptorSetLayout failed\n");
      return VK_NULL_HANDLE;
   }

   return dsl;
}
```
Then there's just the usual Vulkan semantics of storing values to the struct and passing it to the [Create](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCreateDescriptorSetLayout.html) function.

But back in the context of moving descriptor pool creation to the program, this is actually the perfect place to jam the creation code in since all the information about descriptor types is already here. Here's what that looks like:

```c
VkDescriptorPoolSize sizes[6] = {};
int type_map[12];
unsigned num_types = 0;
memset(type_map, -1, sizeof(type_map));

for (int i = 0; i < ZINK_SHADER_COUNT; i++) {
   struct zink_shader *shader = stages[i];
   if (!shader)
      continue;

   VkShaderStageFlagBits stage_flags = zink_shader_stage(pipe_shader_type_from_mesa(shader->nir->info.stage));
   for (int j = 0; j < shader->num_bindings; j++) {
      assert(num_bindings < ARRAY_SIZE(bindings));
      bindings[num_bindings].binding = shader->bindings[j].binding;
      bindings[num_bindings].descriptorType = shader->bindings[j].type;
      bindings[num_bindings].descriptorCount = shader->bindings[j].size;
      bindings[num_bindings].stageFlags = stage_flags;
      bindings[num_bindings].pImmutableSamplers = NULL;
      if (type_map[shader->bindings[j].type] == -1) {
         type_map[shader->bindings[j].type] = num_types++;
         sizes[type_map[shader->bindings[j].type]].type = shader->bindings[j].type;
      }
      sizes[type_map[shader->bindings[j].type]].descriptorCount++;
      ++num_bindings;
   }
}
```
I've added the `sizes`, `type_map`, and `num_types` variables, which map used Vulkan descriptor types to a zero-based array and associated counter that can be used to fill in the `pPoolSizes` and `PoolSizeCount` values in a [VkDescriptorPoolCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDescriptorPoolCreateInfo.html) struct.

After the layout creation, which remains unchanged, I've then added this block:
```c
for (int i = 0; i < num_types; i++)
   sizes[i].descriptorCount *= ZINK_DEFAULT_MAX_DESCS;

VkDescriptorPoolCreateInfo dpci = {};
dpci.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
dpci.pPoolSizes = sizes;
dpci.poolSizeCount = num_types;
dpci.flags = 0;
dpci.maxSets = ZINK_DEFAULT_MAX_DESCS;
vkCreateDescriptorPool(dev, &dpci, 0, &descpool);
```
Which uses the descriptor types and sizes from above to create a pool that will pre-allocate the exact descriptor counts that are needed for this program.

Managing descriptor sets in this way does have other challenges, however. Like resources, it's crucial that sets not be modified or destroyed while they're submitted to a batch.

## Descriptor Tracking
Previously, any time a draw was completed, the batch object would reset and clear its descriptor pool, wiping out all the allocated sets. If the pool is no longer on the batch, however, it's not possible to perform this reset without adding tracking info for the batch to all the descriptor sets. Also, resetting a descriptor pool like this is wasteful, as it's probable that a program will be used for multiple draws and thus require multiple descriptor sets. What I've done instead is add this function, which is called just after allocating the descriptor set:
```c
bool
zink_batch_add_desc_set(struct zink_batch *batch, struct zink_program *pg, struct zink_descriptor_set *zds)
{
   struct hash_entry *entry = _mesa_hash_table_search(batch->programs, pg);
   assert(entry);
   struct set *desc_sets = (void*)entry->data;
   if (!_mesa_set_search(desc_sets, zds)) {
      pipe_reference(NULL, &zds->reference);
      _mesa_set_add(desc_sets, zds);
      return true;
   }
   return false;
}
```
Similar to all the other batch<->object tracking, this stores the given descriptor set into a `set`, but in this case the set is itself stored as the data in a hash table keyed with the program, which provides both objects for use during batch reset:
```c
hash_table_foreach(batch->programs, entry) {
   struct zink_program *pg = (struct zink_program*)entry->key;
   struct set *desc_sets = (struct set*)entry->data;
   set_foreach(desc_sets, sentry) {
      struct zink_descriptor_set *zds = (void*)sentry->key;
      /* reset descriptor pools when no batch is using this program to avoid
       * having some inactive program hogging a billion descriptors
       */
      pipe_reference(&zds->reference, NULL);
      zink_program_invalidate_desc_set(pg, zds);
   }
   _mesa_set_destroy(desc_sets, NULL);
```
And this function is called:
```c
void
zink_program_invalidate_desc_set(struct zink_program *pg, struct zink_descriptor_set *zds)
{
   uint32_t refcount = p_atomic_read(&zds->reference.count);
   /* refcount > 1 means this is currently in use, so we can't recycle it yet */
   if (refcount == 1)
      util_dynarray_append(&pg->alloc_desc_sets, struct zink_descriptor_set *, zds);
}
```
If a descriptor set has no active batch uses, its refcount will be 1, and then it can be added to the array of allocated descriptor sets for immediate reuse in the next draw. In this iteration of refactoring, descriptor sets can only have one batch use, so this condition is always true when this function is called, but future work will see that change.

Putting it all together is this function:
```c
struct zink_descriptor_set *
zink_program_allocate_desc_set(struct zink_screen *screen,
                               struct zink_batch *batch,
                               struct zink_program *pg)
{
   struct zink_descriptor_set *zds;

   if (util_dynarray_num_elements(&pg->alloc_desc_sets, struct zink_descriptor_set *)) {
      /* grab one off the allocated array */
      zds = util_dynarray_pop(&pg->alloc_desc_sets, struct zink_descriptor_set *);
      goto out;
   }

   VkDescriptorSetAllocateInfo dsai;
   memset((void *)&dsai, 0, sizeof(dsai));
   dsai.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO;
   dsai.pNext = NULL;
   dsai.descriptorPool = pg->descpool;
   dsai.descriptorSetCount = 1;
   dsai.pSetLayouts = &pg->dsl;

   VkDescriptorSet desc_set;
   if (vkAllocateDescriptorSets(screen->dev, &dsai, &desc_set) != VK_SUCCESS) {
      debug_printf("ZINK: %p failed to allocate descriptor set :/\n", pg);
      return VK_NULL_HANDLE;
   }
   zds = ralloc_size(NULL, sizeof(struct zink_descriptor_set));
   assert(zds);
   pipe_reference_init(&zds->reference, 1);
   zds->desc_set = desc_set;
out:
   if (zink_batch_add_desc_set(batch, pg, zds))
      batch->descs_used += pg->num_descriptors;

   return zds;
}
```
If a pre-allocated descriptor set exists, it's popped off the array. Otherwise, a new one is allocated. After that, the set is referenced onto the batch.

## Progress
Now all the sets are allocated on the program using a more specific allocation strategy, which paves the way for a number of improvements that I'll be discussing in various lengths over the coming days:
* bucket allocating
* descriptor caching 2.0
* split descriptor sets
* context-based descriptor states
* (pending implementation/testing/evaluation) context-based descriptor pool+layout caching