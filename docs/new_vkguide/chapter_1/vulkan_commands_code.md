---
layout: default
title: Setting up Vulkan commands
parent:  "New 1. Initializing Vulkan"
nav_order: 21
---

We will begin by writing our FrameData struct, on the vk_engine.h header. This will hold the structures and commands we will need to draw a given frame, and that we will be double-buffering. 
```cpp
struct FrameData {

	VkCommandPool _commandPool;
	VkCommandBuffer _mainCommandBuffer;
};

constexpr unsigned int FRAME_OVERLAP = 2;
```
We also need to add those into the vulkan engine class, alongside the members we will use to store the queue.

```cpp
class VulkanEngine{
public:
	FrameData _frames[FRAME_OVERLAP];

	FrameData& get_current_frame() { return _frames[_frameNumber % FRAME_OVERLAP]; };

	VkQueue _graphicsQueue;
	uint32_t _graphicsQueueFamily;
}
```

We will not be accessing the _frames array directly outside of initialization logic. So we add a getter that will use the _frameNumber member we use to count the frames to access it. This way it will flip between the 2 structures we have. 


## Grabbing the Queue

We now need to find a valid queue family and create a queue from it. We want to create a queue that can execute all types of commands, so that we can use it for everything in the engine.

Luckily, the VkBootstrap library allow us to get the Queue and Family directly.

Go to the end of the `init_vulkan()` function, where we initialized the core Vulkan structures.

At the end of it, add this code.


```cpp
void VulkanEngine::init_vulkan(){

// ---- other code, initializing vulkan device ----

	// use vkbootstrap to get a Graphics queue
	_graphicsQueue = vkbDevice.get_queue(vkb::QueueType::graphics).value();
	_graphicsQueueFamily = vkbDevice.get_queue_index(vkb::QueueType::graphics).value();
}
```

We begin by requesting both a queue family and a queue of type Graphics from vkbootstrap. 

## Creating the Command structures

For the pool, we start adding code into `init_commands()` unlike before, from now on the VkBootstrap library will not do anything for us, and we will start calling the Vulkan commands directly.

```cpp
void VulkanEngine::init_commands()
{
	//create a command pool for commands submitted to the graphics queue.
	//we also want the pool to allow for resetting of individual command buffers
	VkCommandPoolCreateInfo commandPoolInfo =  {};
	commandPoolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
	commandPoolInfo.pNext = nullptr;
	commandPoolInfo.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;

	for (int i = 0; i < FRAME_OVERLAP; i++) {

		VK_CHECK(vkCreateCommandPool(_device, &commandPoolInfo, nullptr, &_frames[i]._commandPool));

		// allocate the default command buffer that we will use for rendering
		VkCommandBufferAllocateInfo cmdAllocInfo = {};
		cmdAllocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
		cmdAllocInfo.pNext = nullptr;
		cmdAllocInfo.commandPool = _frames[i]._commandPool;
		cmdAllocInfo.commandBufferCount = 1;
		cmdAllocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;

		VK_CHECK(vkAllocateCommandBuffers(_device, &cmdAllocInfo, &_frames[i]._mainCommandBuffer));
	}
}
```
Most Vulkan Info structures, used for  the VkCreateX functions, and a lot of the other Vulkan structures, need sType and pNext set. This is used for extensions, as some extensions will still call the VkCreateX function, but with structs of a different type than the normal one. The sType helps the implementation know what struct is being used in the function.

With Vulkan structures, it is very important that we do this
```cpp
VkCommandPoolCreateInfo commandPoolInfo = {};
```

By doing that ` = {}` thing, we are letting the compiler initialize the entire struct to zero. This is critical, as in general Vulkan structs will have their defaults set in a way that 0 is relatively safe. By doing that, we make sure we don't leave uninitialized data in the struct.

We set queueFamilyIndex to the _graphicsQueueFamily that we grabbed before. This means that the command pool will create commands that are compatible with any queue of that "graphics" family.

We are also setting something in the .flags parameter. A lot of Vulkan structures will have that .flags parameter, for extra options. We are sending VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT , which tells Vulkan that we expect to be able to reset individual command buffers made from that pool. An alternative approach would be to reset the whole Command Pool at once, which resets all command buffers. In that case we would not need that flag.

At the end, we finally call VkCreateCommandPool, giving it our VkDevice, the commandPoolInfo for create parameters, and a pointer to the _commandPool member, which will get overwritten if it succeeds.

To check if the command succeeds, we use the VK_CHECK() macro. It will just immediately abort if something happens.

Now that we have the VkCommandPool created, and stored in the _commandPool member, we can allocate our command buffer from it.```

As with the command pool, we need to fill the sType and pNext parameters, and then continue the rest of the Info struct.

We let Vulkan know that the parent of our command will be the _commandPool we just created, and we want to create only one command buffer.

The .commandBufferCount parameter allows you to allocate multiple buffers at once. Make sure that the pointer you send to VkAllocateCommandBuffer has space for those.

The .level is set to Primary . Command buffers can be Primary of Secondary level.
Primary level are the ones that are sent into a VkQueue, and do all of the work. This is what we will use in the guide.
Secondary level are ones that can act as "subcommands" to a primary buffer. They are most commonly used when you want to record commands for a single pass from multiple cores. We are not going to use them.

## The VkInit module

If you remember the article that explored the project files, we commented that the vk_initializers module will contain abstraction over the initialization of Vulkan structures. Let's go abstract the 2 Info structures into there, for easier readability.


vk_initializers.h
```cpp
namespace vkinit {
VkCommandPoolCreateInfo command_pool_create_info(uint32_t queueFamilyIndex, VkCommandPoolCreateFlags flags = 0);
VkCommandBufferAllocateInfo command_buffer_allocate_info(VkCommandPool pool, uint32_t count = 1);
}
```

2 new functions, `command_pool_create_info()`, and `command_buffer_allocate_info()`. We also use default arguments `flags = 0` to not have to input all arguments for basic stuff.
Now, let's copy the code into the implementation of those 2 functions.


vk_initializers.cpp

<!-- codegen from tag init_cmd on file E:\ProgrammingProjects\vulkan-guide-2\shared/vk_initializers.cpp --> 
```cpp
VkCommandBufferAllocateInfo vkinit::command_buffer_allocate_info(
    VkCommandPool pool, uint32_t count /*= 1*/)
{
    VkCommandBufferAllocateInfo info = {};
    info.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    info.pNext = nullptr;

    info.commandPool = pool;
    info.commandBufferCount = count;
    info.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    return info;
}

VkCommandBufferBeginInfo vkinit::command_buffer_begin_info(VkCommandBufferUsageFlags flags /*= 0*/)
{
    VkCommandBufferBeginInfo info = {};
    info.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    info.pNext = nullptr;

    info.pInheritanceInfo = nullptr;
    info.flags = flags;
    return info;
}
```

We will be hardcoding command buffer level to VK_COMMAND_BUFFER_LEVEL_PRIMARY . As we wont ever be using them, we can just ignore their existence. By abstracting things with defaults that match your engine, you can simplify things a bit.

We have now abstracted the calls, so let's go update VulkanEngine::init_commands() to use this.

<!-- codegen from tag init_cmd on file E:\ProgrammingProjects\vulkan-guide-2\chapter-1/vk_engine.cpp --> 
```cpp
void VulkanEngine::init_commands()
{
	//create a command pool for commands submitted to the graphics queue.
	//we also want the pool to allow for resetting of individual command buffers
	VkCommandPoolCreateInfo commandPoolInfo = vkinit::command_pool_create_info(_graphicsQueueFamily, VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT);

	for (int i = 0; i < FRAME_OVERLAP; i++) {

		VK_CHECK(vkCreateCommandPool(_device, &commandPoolInfo, nullptr, &_frames[i]._commandPool));

		// allocate the default command buffer that we will use for rendering
		VkCommandBufferAllocateInfo cmdAllocInfo = vkinit::command_buffer_allocate_info(_frames[i]._commandPool, 1);

		VK_CHECK(vkAllocateCommandBuffers(_device, &cmdAllocInfo, &_frames[i]._mainCommandBuffer));
	}
}
```

Much better and shorter. Over the guide, the vk_initializers module will keep growing with more and more structs. You will be able to reuse that module in other projects safely given how simple it is. This module will be addition-only over the whole tutorial, so if you want, you can copy the one from the project sourcecode and add the "final" version directly into the code. Here is the link if you want it. [Header](https://github.com/vblanco20-1/vulkan-guide/blob/all-chapters-1.3-wip/shared/vk_initializers.h) [Source](https://github.com/vblanco20-1/vulkan-guide/blob/all-chapters-1.3-wip/shared/vk_initializers.cpp)

## Cleanup

Same as before, what we have created, we have to delete

```cpp
void VulkanEngine::cleanup()
{
	if (_isInitialized) {
		//make sure the gpu has stopped doing its things
		vkDeviceWaitIdle(_device);

		for (int i = 0; i < FRAME_OVERLAP; i++) {
			vkDestroyCommandPool(_device, _frames[i]._commandPool, nullptr);
		}

		// --- rest of code
	}
}
```

As the command pool is the most recent Vulkan object, we need to destroy it before the other objects.
It's not possible to individually destroy VkCommandBuffer, destroying their parent pool will destroy all of the command buffers allocated from it.

VkQueue-s also can't be destroyed, as, like with the VkPhysicalDevice, they aren't really created objects, more like a handle to something that already exists as part of the VkInstance. 

Now that we have the Queue and the CommandBuffer, we are ready to start executing commands. But we still need another piece, which is the syncronization structures to syncronize GPU execution with CPU