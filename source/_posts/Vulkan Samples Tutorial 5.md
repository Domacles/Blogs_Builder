---
title: Vulkan Samples Tutorial 5
date: 2017-04-25 00:00:00
tags: Vulkan
---

<!-- TOC -->

- [Create a Render Pass](#create-a-render-pass)
    - [Image Layout Transition](#image-layout-transition)
    - [Create the Render Pass](#create-the-render-pass)

<!-- /TOC -->

## Create a Render Pass

本章节代码在`10-init_render_pass.cpp`文件中。

**渲染通道(Render Pass)**指的是：指定的附件集合(collection of attachments)、子通道(subpasses)、渲染过程用到的依赖(dependencies)这一些概念的渲染操作范畴。1个渲染通道至少包含1个子通道。传递给驱动的信息，可以使驱动了解程序希望什么时候开始渲染以及渲染什么东西，还可以让驱动提供渲染操作的硬件优化急速。

我们通过使用`vkCreateRenderPass()`来定义渲染通道，然后使用`vkCmdBeginRenderPass()`和`vkCmdEndRenderPass()`向命令缓冲区插入一个渲染通道实例。

在本章节内容中，我们只是定义并创建渲染通道，但直到本章节代码结束都没有将它插入到命令缓冲区中。

在本章节样例中，渲染通道附件(render pass attachments)用到了颜色附件(color attachment)，就是从Swapchian获得的图像，还用到了深度/模版附件(depth/stencil attachment)，附件使用的深度缓冲区是在之前创建的。

图像附件必须在使用时准备好，这样它们才能够被添加到渲染通道实例上用于在命令缓冲区中执行。准备的过程包括图像布局(Image Layouts)从初始未定义阶段转换到在渲染通道优化阶段的过程。我们将会在本章节了解在创建渲染通道之前，图像布局的转换过程。

### Image Layout Transition

The Need for Alternate Memory Access Patterns

图像的布局(Image Layouts)指的是图像像素如何从二维坐标系映射到图像内存偏移的方法。比较有代表性的布局是，图像数据使用的线性映射：2D图像按照一行接着一行的方法存储到连续的内存中：
```
offset = rowCoord * pitch + colCoord
```
`pitch`表示一行的大小。该变量通常是和图像的宽度相同，但通常会填充一些字节，使得图像数据的起始位置满足GPU内存地址的对齐要求。

线性的布局很适合沿着行来读写连续的像素，即按照`colCoord`变化的方向，但是，多数图形操作包含一些读取相邻行像素的操作，即按照`rowCoord`的方向。如果图像非常宽，那采用线性映射的方法会导致每次存取相邻行内存地址的跨度很大。在多级缓存内存系统中，由于内存地址转换跨度过大造成TLB未命中或者缓存未命中太多，导致性能低下问题的出现。

为了减少这些低效现象的出现，许多GPU硬件实现支持优化optimal/tiled的内存存取方案。在一种优化的布局设计中，图像中间的矩形形状的像素被存储到一段连续的内存中，其他的矩形形状的像素也按照中心矩阵的存放方法存到内存中。举个例子，由左上角[16, 32]到右下角[31, 47]建立的像素矩阵，可以圈出一块16 x 16的像素块，该像素块存到一块连续的内存上。行与行之间的间隔就变得不长了。

如果GPU想要填充上面所讲的像素块，比如可以使用先沟通的颜色，这种方法就可以用一个相对开销较小的内存操作方法来填充这256个像素组成的像素块。

下面是一个简单的2 x 2的平铺图，注意，蓝色的像素在线性内存布局中两部分想个比较远，而在平铺布局中相隔就比较近：

![ImageMemoryLayout](Vulkan Samples Tutorial 5/ImageMemoryLayout.png)

大部分图像内存布局的实现，采用更复杂的平铺方法，且平铺的大小大于2 x 2(这让我想起了用**希尔伯特曲线**填充平面空间的方法)。

Vulkan Control Over the Layout

由于上面所解释的那样，GPU会为了提升渲染效率才支持优化的布局方案。优化的布局方案通常是**不透明的**(opaque)，这表示优化布局方案格式的具体细节并不是公开的或者不会让别人知道如何读写图像数据。

举个例子，假如你想让GPU使用一种优化的布局方案来渲染图像，但是如果你希望让CPU读取和理解最终需要渲染的图像的数据，你需要从优化的布局方案改成一般的布局方案。

在Vulkan中，从一种布局转换到另外一种布局叫做**布局转换**(layout transition)。下面三种途径的任何一种，都能够让我们实现布局转换：
1. 内存栅格化命令(Memory Barrier Command)，通过`vkCmdPipelineBarrier`来控制
2. 了解渲染通道最终布局规范(Render Pass final layout specification)
3. 了解渲染通道子通道布局规范(Render Pass subpass layout specification)

内存栅格化命令(Memory Barrier Command)，是一个可以在命令缓冲区中显式调用的布局转换命令。应用举例，比如我们可以通过这个命令来同步多个复杂场景的内存访问。上面的方法一旦采用另外两种方法来实现布局转换，那就不能用内存栅格化命令来实现布局转换了。

通常，在渲染开始之前和渲染完成之后，需要进行图形布局转换以适应图像渲染。渲染开始前的第一次转换，会为GPU准备好适合的图像格式用于渲染；渲染完成后的最后一次转换，用于将图像呈现给显示设备。在这些情况下，我们需要将布局转换方式指定为渲染通道定义的一部分。在本章节我们将看到是如何具体做的。

一条布局转换命令，可能会触发也可能不会触发事实上GPU布局转换操作。比如，如果旧的布局格式未定义，且新的布局格式是优化的，那GPU就得什么都不做，除非程序已经安排GPU硬件采用优化的模式访问内存。这是因为图像内容处于未定义状态下，不能够通过转换它的格式来进行另存储。另外，如果旧的数据布局格式是一般的，朴素的，且有迹象表明需要图像数据进行另保存，那转换成优化布局时，可能会涉及到GPU的一些重新调整像素位置的工作，即GPU会进行布局转换操作。

即使你知道或者认为一条布局转换操作不会实际发生，但最好的实践方式还是在需要的地方进行添加布局转换操作，这是因为这样可以给驱动程序更多的信息，来帮助驱动软件确保你的应用程序能够在更多平台上执行正确。

Image Layout Transitions in the Samples

下面的样例采用的子通道定义(subpass definitions)和渲染通道定义(render pass definitions)来指定需要的图像格式的布局转换，而不是采用内存栅格化命令。

![RenderPassLayouts](Vulkan Samples Tutorial 5/RenderPassLayouts.png)

初始的渲染通道布局设置成了未定义，这表示我们不必关心初始的渲染通道布局格式，因为当渲染通道开始时，我们不用考虑图像缓存区中已经有什么样的数据且他们已经处于需要显示的状态了。在渲染通道开始时，我们只需要告诉驱动什么样的布局格式是有效的。在这种情况下，驱动不会进行任何转换，直到子通道开始或者渲染通道结束。

子通道布局格式被设置成对颜色缓冲区的优化布局，这就告诉驱动子通道渲染过程中，需要对子通道布局格式进行转换成优化的布局格式。对深度缓冲区的优化布局的设置过程是相似的。

最终渲染通道布局会告诉驱动要将布局转换成最优的布局格式用来显示。

### Create the Render Pass

现在我们应该知道了如何在某些场景中使用合适的图像布局格式，接下来就可以处理剩下的渲染通道。

Attachments

在本样例中，有两个附件(Attachment)，一个是颜色的，另一个是深度缓冲的：
```
VkAttachmentDescription attachments[2];
attachments[0].format = info.format;
attachments[0].samples = NUM_SAMPLES;
attachments[0].loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
attachments[0].storeOp = VK_ATTACHMENT_STORE_OP_STORE;
attachments[0].stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
attachments[0].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
attachments[0].initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
attachments[0].finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
attachments[0].flags = 0;

attachments[1].format = info.depth.format;
attachments[1].samples = NUM_SAMPLES;
attachments[1].loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
attachments[1].storeOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
attachments[1].stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
attachments[1].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
attachments[1].initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
attachments[1].finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
attachments[1].flags = 0;
```
在两个附件中，`loadOp`成员属性都设置为 **CLEAR** ，说明我们希望缓冲区能够在渲染通道实例( render pass instance)开始时被清空。

在颜色附件中，`storeOp`成员属性被设置为 **STORE** ，表示我们希望将渲染结构留在缓冲区中，这样才能够将图像呈献显示设备上。

在深度附件中，`storeOp`成员属性被设置为 **DONT_CARE** ，表示在渲染通道实例完成时不再需要缓冲区的内容，即缓冲区内容可清空可扔掉。

对于图像布局格式，我们需要将两个附件的初始格式设置为**未定义布局格式**(undefined layouts)，原因如同上面所说。

渲染子通道，即发生在初始化布局和最终布局期间，会将颜色附件转换成`VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`状态，将深度附件转换成`VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL`。

对于颜色附件，我们需要指定最终布局为`VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`格式，该格式是一种用于渲染完成后的呈现操作的适合的格式。而对于深度附件，由于深度缓冲区在呈现操作时不会被用到，我们可以让深度缓冲区布局格式和渲染子通道相同。

Subpass

如果需要设置多个子通道，子通道的定义直接而又有趣。假定要实现环境光遮蔽或一些其它的特效，则可能需要对图形数据做一些预处理或后处理，那我们应当关注一下使用多个子通道。在这里，子通道的设定被用于在渲染过程中指定哪些附件有效，和指定渲染时需要用到的布局。
```
VkAttachmentReference color_reference = {};
color_reference.attachment = 0;
color_reference.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

VkAttachmentReference depth_reference = {};
depth_reference.attachment = 1;
depth_reference.layout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```
`attachment`成员属性表示附件数组中对象附件索引，用于渲染渲染通道中。
```
VkSubpassDescription subpass = {};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
subpass.flags = 0;
subpass.inputAttachmentCount = 0;
subpass.pInputAttachments = NULL;
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &color_reference;
subpass.pResolveAttachments = NULL;
subpass.pDepthStencilAttachment = &depth_reference;
subpass.preserveAttachmentCount = 0;
subpass.pPreserveAttachments = NULL;
```
`pipelineBindPoint`成员属性表明是一个图形子通道还是一个计算子通道。当前属性值表示，只有图形子通道有效。

Render Pass

现在，我们已经拥有所有渲染通道需要定义的内容，可以创建渲染通道了：
```
VkRenderPassCreateInfo rp_info = {};
rp_info.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
rp_info.pNext = NULL;
rp_info.attachmentCount = 2;
rp_info.pAttachments = attachments;
rp_info.subpassCount = 1;
rp_info.pSubpasses = &subpass;
rp_info.dependencyCount = 0;
rp_info.pDependencies = NULL;
res = vkCreateRenderPass(info.device, &rp_info, NULL, &info.render_pass);
```
在接下来的样例中，我们将使用渲染通道。

---

本篇**Vulkan Samples Tutorial**原文是**LUNAEXCHANGE**中的[Vulkan Tutorial](https://vulkan.lunarg.com/doc/sdk/1.0.42.1/windows/tutorial/html/index.html)的译文。
并非逐字逐句翻译，如有错误之处请告知。O(∩_∩)O谢谢~~



































