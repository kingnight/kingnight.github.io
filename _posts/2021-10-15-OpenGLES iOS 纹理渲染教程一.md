---
title: "OpenGLES iOS 纹理渲染教程一"
description: "OpenGLES是 OpenGL 的子集，针对手机、PDA 和游戏主机等嵌入式设备而设计。本系列文章以iOS平台进行纹理渲染为实践目标，在查找了大量资料后进行整理而得，本篇是系列的第一篇。"
category: programming
tags: iOS,Swift,OpenGLES,Texture
---

OpenGLES是 OpenGL 的子集，针对手机、PDA 和游戏主机等嵌入式设备而设计。本系列文章以iOS平台进行纹理渲染为实践目标，在查找了大量资料后进行整理而得，本篇是系列的第一篇。

# 0.前言
OpenGL（Open Graphics Library）是 Khronos Group （一个图形软硬件行业协会，该协会主要关注图形和多媒体方面的开放标准）开发维护的一个规范，它是硬件无关的。它主要为我们定义了用来操作图形和图片的一系列函数的 API，OpenGL 本身并非 API。

OpenGL ES（OpenGL for Embedded Systems）是 OpenGL 的子集，针对手机、PDA 和游戏主机等嵌入式设备而设计。该规范也是由 Khronos Group 开发维护。

OpenGL ES 去除了四边形（GL_QUADS）、多边形（GL_POLYGONS）等复杂图元，以及许多非绝对必要的特性，剩下最核心有用的部分。可以理解成是一个在移动平台上能够支持 OpenGL 最基本功能的精简规范。

目前 iOS 平台支持的有 OpenGL ES 1.0，2.0，3.0。OpenGL ES 3.0 加入了一些新的特性，但是它除了需要 iOS 7.0 以上之外，还需要 iPhone 5S 之后的设备才能支持。

>注：下文中的 OpenGL ES 均指代 OpenGL ES 2.0。


# 1.基本概念


## 渲染(Render)

图形处理单元（GPU）就是能够结合几何、颜色、灯光和其他数据而产生一个屏幕图像的硬件组件。屏幕只有2维，因此显示 3D 数据的技巧就在于产生能够迷惑眼睛使其看到丢失的第3维的一个图像。

**用 3D数据生成一个2D 图像的过程就叫渲染。** 在计算机上显示的图片是由矩形的颜色点组成的，这些矩形的颜色点叫做像素。像素由三个颜色组成，红、绿、蓝；

## 上下文(Context)

OpenGL ES 是一个**状态机**，相关的配置信息会被保存在一个上下文（Context）中，这个些值会被一直保存，直到被修改。但我们可以配置多个上下文，通过调用 [EAGLContext setCurrentContext:context] 来切换。

## 缓存(Buffer)
OpenGL ES 部分运行在 CPU 上，部分运行在 GPU 上，CPU 和 GPU 都有独自控制的内存区域，为了协调这两部分的数据交换，定义了缓存（Buffers）的概念。缓存实际上就是指一块连续的 RAM 。

FBO = Frame Buffer object
## OpenGL ES 中的图元(Primitive)
图元（Primitive）是指 OpenGL ES 中支持渲染的基本图形。OpenGL ES 只支持三种图元，分别是顶点、线段、三角形。复杂的图形得通过渲染多个三角形来实现。

## 顶点(Vertex)
顶点数据也称作顶点属性，指定每个顶点的数据。这种逐顶点数据可以为每个顶点指定，也可以用于所有顶点的常量。例娅，如果你想要绘制固定颜色的三角形（在这个例子中，假定颜色为黑色)，可以指定一个常量值，用于三角形的全部了个顶点。但是，组成三角形的3个项点的位置不同，所以我们必须指定一个顶点数组来存储三个位置值。

VBO = Vertex Buffer Object
## 纹理(Texture)
纹理是一个用来保存图像颜色的元素值的缓存，渲染是指将数据生成图像的过程。纹理渲染则是将保存在内存中的颜色值等数据，生成图像的过程。

## 坐标系(Coordinate system)
### OpenGL ES 坐标系

![d5b55c801c7f42a7499337f6119d67b5](/assets/images/render-texture-image-1.jpeg)

OpenGL ES 坐标系的范围是 -1 ~ 1，是一个三维的坐标系，通常用 X、Y、Z 来表示。Z 轴的正方向指向屏幕外。在不考虑 Z 轴的情况下，左下角为 (-1, -1, 0)，右上角为 (1, 1, 0)。

### 纹理坐标系

![7f6a07c67e4c70321972a018ba353585](/assets/images/render-texture-image-2.jpeg)


纹理坐标系的范围是 0 ~ 1，是一个二维坐标系，横轴称为 S 轴，纵轴称为 T 轴。在坐标系中，点的横坐标一般用 U 表示，点的纵坐标一般用 V 表示。左下角为 (0, 0)，右上角为 (1, 1)。

注： UIKit 坐标系的 (0, 0) 点在左上角，其纵轴的方向和纹理坐标系纵轴的方向刚好相反。

## 着色器(Shaders)
顶点着色器和片段着色器是可编程的部分，着色器（Shader）是一个小程序，它们运行在 GPU 上，在主程序运行的时候进行动态编译，而不用写死在代码里面。编写着色器用的语言是 GLSL（OpenGL Shading Language）


## GLKit
GLKit是iOS基于openGL实现的一个框架，目的在于简化OpenGL的操作，其提供了GLKView和GLKViewController，OpenGL相关的操作需要在这两个类中的子类中完成。

## 小结

![d62e4916e3b4a98d5e24e4d106df95f7](/assets/images/示意图.png)


# 渲染的流程

![ab86079dc068818575bd268f8c5f1459](/assets/images/render-texture-image-3.jpeg)

以渲染三角形为例

1、顶点数据

为了渲染一个三角形，我们需要传入一个包含 3 个三维顶点坐标的数组，每个顶点都有对应的顶点属性，顶点属性中可以包含任何我们想用的数据。在上图的例子里，我们的每个顶点包含了一个颜色值。

并且，为了让 OpenGL ES 知道我们是要绘制三角形，而不是点或者线段，我们在调用绘制指令的时候，都会把图元信息传递给 OpenGL ES 。

![c62a184fc456afcd8aa591a26a4010e2](/assets/images/截屏2021-09-16 17.42.37.png)


2、顶点着色器

顶点着色器会对每个顶点执行一次运算，它可以使用顶点数据来计算该顶点的坐标、颜色、光照、纹理坐标等。

顶点着色器的一个重要任务是进行坐标转换，例如将模型的原始坐标系（一般是指其 3D 建模工具中的坐标）转换到屏幕坐标系。

![e2750d4204fc81a2d621444fefb126fa](/assets/images/step2.png)


3、图元装配

在顶点着色器程序输出顶点坐标之后，各个顶点按照绘制命令中的图元类型参数，以及顶点索引数组被组装成一个个图元。

通过这一步，模型中 3D 的图元已经被转化为屏幕上 2D 的图元。

![d356b02307f2b241e085e5e3be8fc27b](/assets/images/step3.png)


~~4、几何着色器~~

~~在「OpenGL」的版本中，顶点着色器和片段着色器之间有一个可选的着色器，叫做几何着色器（Geometry Shader）。几何着色器把图元形式的一系列顶点的集合作为输入，它可以通过产生新顶点构造出新的图元来生成其他形状。OpenGL ES 目前还不支持几何着色器，这个部分我们可以先不关注。~~


5、光栅化

在光栅化阶段，基本图元被转换为供片段着色器使用的片段。**片段表示可以被渲染到屏幕上的像素**，它包含位置、颜色、纹理坐标等信息，这些值是由图元的顶点信息进行插值计算得到的。

在片段着色器运行之前会执行裁切，处于视图以外的所有像素会被裁切掉，用来提升执行效率。

![896f06702b4f27deba8b92eb9b7f6b5e](/assets/images/step4.png)


6、片段着色器

**片段着色器的主要作用是计算每一个片段最终的颜色值（或者丢弃该片段）。片段着色器决定了最终屏幕上每一个像素点的颜色值。**

![abdac3f030975e7081169dd159e000e0](/assets/images/step5.png)

7、测试与混合

在这一步，OpenGL ES 会根据片段是否被遮挡、视图上是否已存在绘制好的片段等情况，对片段进行丢弃或着混合，最终被保留下来的片段会被写入**帧缓存**中，最终呈现在设备屏幕上。

![c84625fb5cf0fbcfaaa7424a9104dc9c](/assets/images/step6.png)


# 缓存

## CPU到GPU缓存数据交换
程序从 CPU 的内存复制数据到 OpenGL ES的缓存。在GPU 取得一个缓存的所有权以后，运行在 CPU 中的程序理想情况下将不再接触这个缓存。通过控制独占的缓存，GPU 就能够尽可能以最有效的方式读写内存。

在实际应用中，我们需要使用各种各样的缓存。比如在纹理渲染之前，需要生成一块保存了图像数据的纹理缓存。下面介绍一下缓存管理的一般步骤：

使用缓存的过程可以分为 7 步：

1. 生成（Generate）：生成缓存标识符 glGenBuffers()
2. 绑定（Bind）：对接下来的操作，绑定一个缓存 glBindBuffer()
3. 缓存数据（Buffer Data）：从CPU的内存复制数据到缓存的内存 glBufferData() / glBufferSubData()
4. 启用（Enable）或者禁止（Disable）：设置在接下来的渲染中是否要使用缓存的数据 glEnableVertexAttribArray() / glDisableVertexAttribArray()
5. 设置指针（Set Pointers）：告知缓存的数据类型，及相应数据的偏移量 glVertexAttribPointer()
6. 绘图（Draw）：使用缓存的数据进行绘制 glDrawArrays() / glDrawElements()
7. 删除（Delete）：删除缓存，释放资源 glDeleteBuffers()

## 帧缓存
GPU 需要知道应该在内存中的哪个位置存储渲染出来的2D图像像素数据。就像为 GPU 提供数据的缓存一样，**接收渲染结果的缓冲区叫做帧缓存 (frame buffer)。** 程序会像任何其他种类的缓存一样生成、绑定、删除帧绶存。

可以同时存在很多帧缓存，并且可以通过 OpenGL ES 让 GPU 把渲染结果存储到任意数量的帧缓存中。但是，屏幕显示像素要受到**保存在前帧缓存 (front frame buffer)的特定帧缓存中的像素颜色元素**的控制。程序和操作系统很少会直接渲染到前帧缓存中，因为那样会让用户看到正在渲染中的还没渲染完成的图像。相反，程序和操作系统会把渲染结果保存到包括**后帧缓存 (back frame buffer）在内的其他帧绥存**中。**当渲染后的后帧缓存包含一个完成的图像时，前帧缓存与后帧缓存几乎会瞬间切换。后帧缓存会变成新的前帧缓存，同时旧日的前帧缓存会变成后帧缓存。** 下图展示了屏幕显示像素、前帧缓存及后帧缓存三者之间的关系。

![abc6f804bc5b4f629dac5e28b7f7d469](/assets/images/截屏2021-10-09 14.48.09.png)

# 上下文(Context)

用于配置 OpenGL ES 的保存在特定平台的软件数据结构中的信息会被封装到一个OpenGL ES 上下文 （context） 中。**OpenGL ES 是一个状态机器，这意味着在一个程序中设置了一个配置值后，这个值会一直保持，直到程序修改了这个值。** 上下文中的信息可能会被保存在CPU 所控制的内存中，也可能会被保存在GPU 所控制的内存中。OpenGL ES 会按需在两个内存区城之间复制信息。


```swift
private(set) var context: EAGLContext?

self.context = EAGLContext.init(api: .openGLES2)
let result = EAGLContext.setCurrent(self.context)
if !result {
   print("Failed to set current OpenGL context")
   return
}
```

**OpenGL ES 上下文会跟踪用于渲染的帧缓存**。上下文还会跟踪用于几何数据颜色等的缓存。上下文会决定是否使用某些功能，比如纹理和灯光.


# 使用OpenGL ES绘制一个Core Animation层

前面介绍了OpenGL ES 的帧缓存。iOS操作系统不会让应用直接向前帧缓存或者后帧缓存绘图，也不会让应用直接控制前帧缓存和后帧缓存之间的切换。操作系统为自己保留了这些操作，以便它可以随时使用 Core Animation 合成器来控制显示的最终外观。

Core Animation 包含层的概念。同一时刻可以有任意数量的层。Core Animation 合成器会联合这些层并在后帧缓存中产生最终的像素颜色，然后切换缓存。下图显示的是合并两个层来产生后帧缓存中的颜色数据的过程。

![d5b1b256bd28561872ebb8d0fc254818](/assets/images/截屏2021-10-09 15.45.28.png)

帧缓存会保存OpenGL ES 的渲染结果，因此为了缓存到一个 Core Animation 层上，程序需要一个连接到某个层的帧缓存。简言之，每个程序用足够的内存配置一个层来保存像素颜色数据，之后创建一个使用层的内存来保存渲染的图像的帧缓存。下图介绍了 OpenGL ES 的帧缓存与层之间的关系。

![7c4873441d930904bc57d69d4cae8f90](/assets/images/截屏2021-10-09 15.50.39.png)

上图显示了一个像素颜色渲染缓存 (pixcl color render buffer）和另外两个标识为其他渲染缓存（other render buffer）的缓存。除了像素颜色数据，OpenGL ES 和 GPU有时会以這染的副产品的形式产生一些有用的数据。帧缓存可以配置多个叫做渲染缓存的缓存米接收多种类型的输出。**与层分享数据的帧缓存必须要有一个像素颜色渲染缓存，其他的渲染缓存是可选的。**



# 绘制到其他渲染目的地
以下部分为官方文档翻译节选，[原文链接](https://developer.apple.com/library/archive/documentation/3DDrawing/Conceptual/OpenGLES_ProgrammingGuide/WorkingwithEAGLContexts/WorkingwithEAGLContexts.html#//apple_ref/doc/uid/TP40008793-CH103-SW6)。

Framebuffer对象是渲染命令的目标。当您创建一个framebuffer对象时，**您可以精确控制其存储的颜色，深度和模板数据。** 您可以通过将图像附加到帧缓冲区来提供此存储，如图4-1所示。最常见的图像附件是一个renderbuffer对象。您还可以将OpenGL ES纹理附加到帧缓冲区的颜色附加点，这意味着任何绘图命令都将呈现到纹理中。之后，纹理可以作为未来渲染命令的输入。您还可以在单个渲染上下文中创建多个帧缓冲区对象。您可以这样做，以便您可以在多个帧缓冲区之间共享相同的渲染管道和OpenGL ES资源。

![35df7960bbda6726f241dca53f9f8237](/assets/images/E79DB89A-2ED1-48FE-976C-B57E8745EDB0.png)

所有这些方法都需要手动创建framebuffer和renderbuffer对象来存储来自OpenGL ES上下文的渲染结果，以及编写附加代码以将其内容显示在屏幕上，如果需要，运行动画循环


## 创建一个 Framebuffer Object

根据您的应用程序要执行的任务，您的应用程序会配置不同的对象以附加到framebuffer对象。在大多数情况下，配置帧缓冲区的区别在于什么对象附加到framebuffer对象的颜色附着点

* 要使用帧缓冲区进行屏幕外图像处理，请附加一个renderbuffer。请参阅创建Offscreen Framebuffer对象。
* 要使用帧缓冲图像作为后续渲染步骤的输入，请附加纹理。请参阅使用Framebuffer对象渲染到纹理。
* 要在Core Animation图层组合中使用framebuffer，请使用特殊的Core Animation感知renderbuffer。请参阅渲染到核心动画层。

## 创建 Offscreen Framebuffer Objects
用于屏幕外渲染的帧缓冲区将其所有附件分配为OpenGL ES渲染缓冲区。以下代码分配带有颜色和深度附件的framebuffer对象。

1. 创建帧缓冲区并绑定它。
```swift
GLuint framebuffer;
glGenFramebuffers(1, &framebuffer);
glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
```

2. 创建一个彩色渲染缓冲区，为其分配存储空间，并将其附加到framebuffer的颜色附着点。

```swift
GLuint colorRenderbuffer;
glGenRenderbuffers(1, &colorRenderbuffer);
glBindRenderbuffer(GL_RENDERBUFFER, colorRenderbuffer);
glRenderbufferStorage(GL_RENDERBUFFER, GL_RGBA8, width, height);
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_RENDERBUFFER, colorRenderbuffer);
```

3. 创建一个深度或深度/模板renderbuffer，为其分配存储，并将其附加到framebuffer的深度附件点。

```swift
GLuint depthRenderbuffer;
glGenRenderbuffers(1, &depthRenderbuffer);
glBindRenderbuffer(GL_RENDERBUFFER, depthRenderbuffer);
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT16, width, height);
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, depthRenderbuffer);
```

4. 测试framebuffer的完整性。只有当帧缓冲区的配置更改时，才需要执行此测试。

```swift
GLenum status = glCheckFramebufferStatus(GL_FRAMEBUFFER) ;
if(status != GL_FRAMEBUFFER_COMPLETE) {
    NSLog(@"failed to make complete framebuffer object %x", status);
}
```

绘制到屏幕外的renderbuffer后，可以将其内容返回给CPU，以便使用glReadPixels函数进一步处理


## 将Framebuffer对象的内存渲染到纹理中 
创建此帧缓冲区的代码与屏幕外的示例几乎相同，但是现在将分配纹理并附加到颜色附加点

1. 创建framebuffer对象（使用与创建Offscreen Framebuffer对象相同的过程）。
2. 创建目标纹理，并将其附加到framebuffer的颜色附件点。
```swift
// create the texture
GLuint texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8,  width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, NULL);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, texture, 0);
```
3. 分配并附加深度缓冲区（如前所述）。
4. 测试framebuffer的完整性（如前所述）。


## 渲染到核心动画层

Core Animation是iOS上图形渲染和动画的核心基础设施。你可以使用不同的iOS子系统(如UIKit、Quartz 2D和OpenGL ES)承载内容的层来组成应用程序的用户界面或其他可视化显示。**OpenGL ES通过CAEAGLLayer类连接到Core Animation, CAEAGLLayer是一种特殊类型的Core Animation层，它的内容来自OpenGL ES渲染缓冲区。** Core Animation将渲染缓冲区的内容与其他图层混合，并将结果图像显示在屏幕上。

![e9f810573dea9b843bb996f1dc7d17c2](/assets/images/7AA33872-9E7D-4342-8986-E78E923BE8A4.png)

CAEAGLLayer通过提供两个关键功能向OpenGL ES提供此支持。首先，它为renderbuffer分配共享存储。其次，它将渲染缓冲区呈现给Core Animation，将该图层的以前内容替换为renderbuffer中的数据。该模型的优点在于，只有当渲染缓冲区的内容发生变化时, Core Animation layer 才需要进行绘制，Core Animation layer核心动画层的内容不需要在每个帧中绘制。


> **注意：GLKView类会自动执行以下步骤，因此当您需要将OpenGL ES的内容绘制到包含layer的视图上时，您应该使用GLKView。**
>
> **注解：当需要将OpenGL ES的内容绘制到iOS的UIView上时，需要使用GLKView类或CAEAGLLayer类来实现将OpenGL ES的内容绘制到iOS的UIView上。**


## 为OpenGL ES渲染使用Core Animation层

1. 创建CAEAGLLayer对象并配置其属性。
为获得最佳性能，请将图层的不透明属性的值设置为YES。看到注意核心动画合成性能。可选地，通过为CAEAGLLayer对象的drawableProperties属性分配一个新的值字典来配置渲染表面的表面属性。您可以指定renderbuffer的像素格式，并指定在将它们发送到Core Animation之后，renderbuffer的内容是否被丢弃。有关允许密钥的列表，请参阅EAGLDrawable Protocol Reference。

2. 分配OpenGL ES上下文并使其成为当前上下文。请参阅配置OpenGL ES上下文。
3. 创建framebuffer对象（如上面的创建Offscreen Framebuffer对象）。
4. 创建一个颜色renderbuffer，通过调用上下文的renderbufferStorage：fromDrawable：method分配其存储，并传递层对象作为参数。宽度，高度和像素格式取自层，用于为renderbuffer分配存储空间

```swift
GLuint colorRenderbuffer;
glGenRenderbuffers(1, &colorRenderbuffer);
glBindRenderbuffer(GL_RENDERBUFFER, colorRenderbuffer);
[myContext renderbufferStorage:GL_RENDERBUFFER fromDrawable:myEAGLLayer];
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_RENDERBUFFER, colorRenderbuffer);
```

**注意：当核心动画层的界限或属性更改时，应用程序应重新分配renderbuffer的存储空间。如果不重新分配renderbuffers，renderbuffer大小将不匹配图层的大小;在这种情况下，Core Animation可以缩放图像的内容以适应图层。**

5. 检索颜色renderbuffer的高度和宽度。
```swift
GLint width;
GLint height;
glGetRenderbufferParameteriv(GL_RENDERBUFFER, GL_RENDERBUFFER_WIDTH, &width);
glGetRenderbufferParameteriv(GL_RENDERBUFFER, GL_RENDERBUFFER_HEIGHT, &height);
```
在前面的例子中，renderbuffers的宽度和高度被明确地提供给缓冲区的分配存储。这里，代码在分配存储后从颜色renderbuffer中检索宽度和高度。您的应用程序执行此操作是因为颜色renderbuffer的实际尺寸是根据图层的边界和比例因子计算的。附加到帧缓冲区的其他渲染缓冲区必须具有相同的尺寸。除了使用高度和宽度来分配深度缓冲区之外，还可以使用它们来分配OpenGL ES视口，并帮助确定应用程序纹理和模型所需的详细程度。请参阅支持高分辨率显示器。

6. 分配并附加深度缓冲区（如前所述）。
7. 测试framebuffer的完整性（如前所述）。
8. 将CAEAGLLayer对象添加到Core Animation层次结构，将其传递给可见层的addSublayer：方法。


# 参考

* http://www.lymanli.com/2019/02/17/ios-opengles-render-texture/
* [官方文档](https://developer.apple.com/library/archive/documentation/3DDrawing/Conceptual/OpenGLES_ProgrammingGuide/WorkingwithEAGLContexts/WorkingwithEAGLContexts.html#//apple_ref/doc/uid/TP40008793-CH103-SW6)
* OpenGL ES应用开发实践 指南 iOS卷
* OPENGL ES 3.0编程指南
* [Beginning OpenGL for iOS [youtube video Series]](https://www.youtube.com/watch?v=VN_qGY43A1Y&list=PL23Revp-82LL_XoQEiTT6zsgHHrpjr1D9)  









