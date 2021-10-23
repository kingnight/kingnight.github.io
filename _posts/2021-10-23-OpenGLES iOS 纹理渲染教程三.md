---
title: "OpenGLES iOS 纹理渲染教程三"
description: "OpenGLES是 OpenGL 的子集，针对手机、PDA 和游戏主机等嵌入式设备而设计。本系列文章以iOS平台进行纹理渲染为实践目标，在查找了大量资料后进行整理而得，本篇是系列的第三篇。"
category: programming
tags: iOS,Swift,OpenGLES,Texture
---

OpenGLES是 OpenGL 的子集，针对手机、PDA 和游戏主机等嵌入式设备而设计。本系列文章以iOS平台进行纹理渲染为实践目标，在查找了大量资料后进行整理而得，本篇是系列的第三篇。

# 纹理

## 概念

回顾整个渲染流程
![ab86079dc068818575bd268f8c5f1459](/assets/images/render-texture-image-3.jpeg)

纹理是一个用来保存图像颜色的元素值的缓存；

### 纹素(texel)

当用一个图像初始化一个纹理缓存之后，在这个图像中的每个像素变成了纹理中的一个纹素 (texel) 。与像素类似，纹素保存颜色数据。像素和纹素之间的差别很微妙：像素通常表示计算机屏幕上的一个实际的颜色点，因此，像素通常被用来作为一个测量单位。说一个图像有256像素宽，256像素高是正确的。纹素存在于一个虚拟的没有尺寸的数学坐标系中。

纹理坐标系有一个命名为S和T的2D轴。在一个纹理中无论有多少个纹素，纹理的尺寸永远是在S轴上从0.0到1.0，在T轴上从0.0到1.0。 从一个1像素高64像素宽的图像初始化来的纹理会沿着整个T轴有1纹素，沿着S轴有64纹素。


### 对齐纹理和几何形状

* 帧缓存中的像素位置叫做(viewport)坐标。

* 光栅化（Rasterizing）：将几何形状数据转换为片段的渲染步骤。

* 片段（Fragment）：视口坐标中的颜色像素。**没有使用纹理时，会使用对象顶点来计算片段的颜色；使用纹理时，会根据纹素来计算。**

* 映射（Mapping）：对齐顶点和纹素的方式。即将顶点坐标 (X, Y, Z) 与 纹理坐标 (U, V) 对应起来。

![51efda9b18ac41acc1a2af709939ce26](/assets/images/截屏2021-10-11 15.09.02.png)


### 纹理的取样模式

取样（Sampling）：在顶点固定后，每个片段根据计算出来的 (U, V) 坐标，去找相应纹素的过程。

OpenGL ES 支持多个不同的取样模式：考虑一下当一个三角形产生的片段少于绑定的纹理内的可用纹素的数量时会发生什么。一个拥有大量纹素的纹理被映射到帧缓存内的一个只覆盖几个像素的三角形中，这种情况会在任何时间发生。相反的情况也会发生。一个包含少量纹素的纹理可能会被映射到一个在帧缓存中产生很多个片段的三角形。**程序会使用如下的gITexParameteri()函数来配置每个绑定的纹理，以便使 OpenGL ES 知道怎么处理可用纹素的数量与需要被着色的片元的数量之间的不匹配。**

```swift
glTexParameteri( GLenum(GL_TEXTURE_2D), GLenum(GL_TEXTURE_MIN_FILTER), GL_LINEAR );
glTexParameteri( GLenum(GL_TEXTURE_2D), GLenum(GL_TEXTURE_MAG_FILTER), GL_LINEAR );
glTexParameteri( GLenum(GL_TEXTURE_2D), GLenum(GL_TEXTURE_WRAP_S), GL_CLAMP_TO_EDGE);
glTexParameteri( GLenum(GL_TEXTURE_2D), GLenum(GL_TEXTURE_WRAP_T), GL_CLAMP_TO_EDGE);
```

使用值 `GL_LINEAR` 来指定参数 `GL_TEXTURE_MIN_FILTER` 会告诉 OpenGL ES无论何时出现多个纹素对应一个片元时，从相配的多个纹素中取样颜色，然后使用线性内插法来混合这些颜色以得到片段的颜色。产生的片段颜色可能最终是一个纹理中不存在的颜色。例如，一个纹理是由交替的黑色和白色纹素组成的，线性取样会混合纹素的颜色，因此片元最终会是灰色的。

使用`GL_NEAREST` 值来指定 `GL_TEXTURE_MIN_FILTER` 参数会产生一个不同的结果。与片段的U、V坐标最接近的纹素的颜色会被取样。如果一个纹理是由交替的黑色和白色纹素组成的，`GL_NEAREST` 取样模式会拾取其中的一个纹素或者另一个，并且最终的片段要么是白色的，要么是黑色的。

`GL_TEXTURE_MAG_FILTER` 参数用于在没有足够的可用纹素来唯一性地映射一个或者多个纹素到每个片元上时配置取样。在这种情况下，`GL_LINEAR` 值会告诉OpenGL ES 混合附近纹素的颜色来计算片元的颜色。`GL_LINEAR`值会有一个放大纹理的效果，并会让它模糊地出现在渲染的三角形上。`GL_TEXTURE_MAG_FILTER` 的 `GL_NEAREST` 值仅仅会拾取与片元的 U、V位置接近的纹素的颜色，并放大纹理，这会使它有点像索化地出现在渲染的三角形上。

### MIP贴图

内存存取是现代图形处理的薄弱环节。当有当个纹素对应一个片段时，线性取样会导致 GPU 仅仅为了计算一个片元的最终颜色而读取多个纹素的颜色值。MIP 贴图是一个为纹理存储多个细节级别的技术。**高细节的纹理会沿着S轴和T轴存储很多的纹素。低细节的纹理沿着每个轴存储很少的纹素。最低细节的纹理只保存一个纹素。** 多个细节级別增加在S、T轴上的纹素和每个片元的 U、V坐标之间有紧密的对应关系的可能性。当存在一个紧密的对应关系时，GPU 会减少取样纹素的数量，进而会减少内存访问的次数。

## 纹理图片的加载

生成纹理的步骤比较固定，以下封装成一个方法：

```swift
    static func createTexture2D(fileName: String) -> GLuint
    {
        var texture: GLuint = 0
        
        // 1获取图片的CGImageRef
        guard let spriteImage = UIImage(named: fileName)?.cgImage else {
            print("Failed to load image \(fileName)")
            return texture
        }
        // 读取图片大小
        let width = spriteImage.width
        let height = spriteImage.height
        
        let spriteData = calloc(width * height * 4, MemoryLayout<GLubyte>.size)
        
        let spriteContext = CGContext(data: spriteData, width: width, height: height, bitsPerComponent: 8, bytesPerRow: width*4, space: spriteImage.colorSpace!, bitmapInfo: CGImageAlphaInfo.premultipliedLast.rawValue)
        
        // 3在CGContextRef上绘图
        spriteContext?.draw(spriteImage, in: CGRect(x: 0, y: 0, width: width, height: height))
    
        // 4绑定纹理到默认的纹理ID（这里只有一张图片，故而相当于默认于片元着色器里面的colorMap，如果有多张图不可以这么做）
        glGenTextures(1, &texture)
        glBindTexture(GLenum(GL_TEXTURE_2D), texture);
        
        //设置如何把纹素映射成像素
        glTexParameteri( GLenum(GL_TEXTURE_2D), GLenum(GL_TEXTURE_MIN_FILTER), GL_LINEAR );
        glTexParameteri( GLenum(GL_TEXTURE_2D), GLenum(GL_TEXTURE_MAG_FILTER), GL_LINEAR );
        glTexParameteri( GLenum(GL_TEXTURE_2D), GLenum(GL_TEXTURE_WRAP_S), GL_CLAMP_TO_EDGE);
        glTexParameteri( GLenum(GL_TEXTURE_2D), GLenum(GL_TEXTURE_WRAP_T), GL_CLAMP_TO_EDGE);
        
        let fw = width
        let fh = height;
        //生成纹理
        glTexImage2D(GLenum(GL_TEXTURE_2D), 0, GL_RGBA, GLsizei(fw), GLsizei(fh), 0, GLenum(GL_RGBA), GLenum(GL_UNSIGNED_BYTE), spriteData);
        glBindTexture(GLenum(GL_TEXTURE_2D), 0);
        //释放内存
        free(spriteData)
        return texture
    }
```

其中
```swift   
glTexImage2D(GLenum(GL_TEXTURE_2D), 0, GL_RGBA, GLsizei(fw), GLsizei(fh), 0, GLenum(GL_RGBA), GLenum(GL_UNSIGNED_BYTE), spriteData);
```

它复制图片像素的颜色数据到绑定的纹理缓存中。

* 第一个参数是用于2D纹理的`GL_TEXTURE_2D`。
* 第二个参数用于指定 MIP 贴图的初始细节级别。如果没有使用 MIP 贴图，第二个参数必须是0。如果开启了 MIP 贴图，使用第二个参数来明确地初始化每个细节级别，但是要小心，因为从全分辨率到只有一纹素的每个级别都必领被指定，否则GPU 将不会接受这个纹理缓存。
* 第三个参数是 iteralFormat，用于指定在纹理缓存内每个纹素需要保存的信息的数量。对于iOS设备来说，纹素信息要么是`GL_RGB`,要么是`GL_RGBA`,`GL_RGB` 为每个纹素保存红、绿、蓝三种颜色元素。`GLL_RGBA` 保存—个额外的用于指定每个纹素透明度的透明度元素。
* 第四个和第五个参数用于指定图像的宽度和高度。高度和宽度需要是2的幂。border 参数一直是用来确定围绕纹理的纹素的-个边界的大小，但是在OpenGL ES 中它总是被设置为0。
* 第七个参数 fomat 用于指定初始化缓存所使用的图像数据中的每个像素所要保存的信息，这个参数应该,总是与 interalFormat 参数相同。其他的OpenGL 版本可能在 format 和 internalFormat 参数不一致时自动执行图像数据格式的转换。
* 倒数第二个参数用于指定缓存中的纹素数据所使用的位编码类型，可以是下面的符号值之一：`GL_UNSIGNED_BYTE、GL_UNSIGNED_SHORT_5_6_5、 GL_UNSIGNED_SHORT_4_4_4_4` 和 `GL_UNSIGNED_SHORT_5_5_5_1`。使用`GL_UNSIGNED_BYTE`会提供最佳色彩质量，但是它每个纹素中每个颜色元素的保存需要一字节的存储空间。


## 激活 & 指定 纹理数据 到 fragment shader 的 uniform 处

首先回顾

### 理解流水线

![eb34dfd131577559d1b85a6fbb32c6a0](/assets/images/5D773A7B-AAC5-41EF-8A2A-F9A190F2123A.jpg)

用户的数据输入，经过 OpenGL 这个盒子，就会输出成像素。用户输入，可以是顶点、纹理、颜色、光照、法线等等。当我们打开 OpenGL 这个盒子，它分成一个个模块。将一些细节砍掉，粗略分为：

```
输入 -> | 顶点处理 -> 裁剪、图元组装 -> 光栅化 -> 图元处理 | -> 像素
```
上一个模块的输出，是下一个部件的模块的输入，构成数据流。


### 理解Shader

OpenGL 的模块向可编程方向发展。顶点处理模块和图元处理模块，最先被抽离出来，变成可编程。OpenGL 可以看成 GPU 的一个抽象，Shader 其实就是跑在 GPU 上的一段小程序。

顶点处理模块，执行 Vertex Shader。 图元处理模块，执行 Fragment Shader。

**两个 Shader 分别在不同的模块中执行，因此需要分开编译。顶点处理的输出，会最终传到图元处理模块作为输入，因此两个 Shader 需要配合起来，也就需要链接。**

Fragment 这个词，这里翻译成图元，另外的文章可能翻译成面片，或者是片段。

3D模型（2D可以看成是3D的特列），不管怎么复杂，也可以一直拆分，最终拆分为一个个三角形，或者是一个个四边形。这种到最后已经不可以再拆分的最基础的形状，就叫图元。OpenGL ES 去掉了四边行，只留下三角形作为最基础的图元。三角形有个优点，是三点永远在同一平面上，可以避免很多特殊情况。

三角形由一个个顶点组成，不同的三角形可能共用相同的顶点。先处理顶点，再将顶点组装起来，变成一个个图元，再处理图元。组装之后图元中每个顶点的信息都有了，之后再通过插值的方式，来处理每个图元中内部的像素点。

**Vertex Shader，用来处理顶点，每个顶点执行一次。 Fragment Shader，用来处理图元，每个图元中的像素执行一次。（这里先忽略像素裁剪）**

**同一个渲染过程中，Vertex Shader 和 Fragment Shader 执行的次数并不是一样的。Fragment Shader会执行更多次。**

Shader 针对图形学设计，基本的数据类型有。

* 整形，int
* 浮点，float
* 向量，vec4，vec3
* 矩阵，mat4
* **纹理单元，sampler2D**


### Shader的修饰符
Shader 每个数据，除了需要指定数据类型，有时候还需要指定修饰符。有三个修饰符：

* attribute
* uniform
* varying

网上可以找到很多资料，告诉你修饰符的作用。比如 attribute 只可以用在 Vertex shader 中，uniform 可以同时用在 Vertex 和 Fragment Shader中，varying 需要分别在 Vertex 和 Frament 中定义。

但是为什么呢？修饰符怎么用可以很容易查到，但是为什么修饰符要这样用呢？

将上面的渲染管线图再简化：
```
输入 -> | 顶点处理 -> XXXXXX -> 图元处理 | -> 像素
```
Vertex Shader对应于顶点处理，Fragment Shader 对应于图元处理，这样变成
```
输入 -> | Vertex Shader -> XXXXXX -> Fragment Shader | -> 像素
```
从 Shader 的角度来看，有三种数据：

* Shader 内部用的数据
* Shader 的输入数据
* Shader 的输出数据

内部数据是自己用的，也就不用加任何修饰符。

**按照管线的设计，用户只能向 Vertex Shader 传数据，不能直接向 Fragment Shader传数据。而 Vertex Shader 可以向 Fragment Shader 传送数据。**

**另外用户也可以向 OpenGL 的状态机传数据，用户传的状态数据在整个渲染过程，是始终如一，不变的。从而 Vertex Shader 和 Fragment Shader 都可以获取。**

Vertex Shader 每个顶点都会被执行一次，因此传向 Vertex Shader 的数据是每个顶点都不同的，用户的输入就很自然表现成一个数组。

经过上面的讨论，再来分析三个修饰符：

* attribute，为每个顶点的属性数据，每个顶点都不一样。顶点属性也只能在Vertex Shader 中定义，**是用户向顶点处理模块传输的数据。**
* uniform，为状态数据，整个渲染过程都是不变的。uniform 这个单词意思就为始终如一。状态数据，两种 Shader 中都可以使用，**但需要声明一致，不然会链接错误。**
* **varying，是 Vertex Shader 向 Fragment Shader 传输的数据，相当于两个shader 之间的数据通道。** 要通信，两个 Shader 就需要有一致的约定，表现成语言，就变成一致的数据声明。Vertex Shader 只管向 varying 写入顶点处理后的结果。Fragment shader 只管从 varying 中读取已经是经过插值后的结果。

纹理、矩阵是整个渲染过程都不变的（一个渲染过程指调用一次glDrawXXX)，通常都是作为 uniform 来传输。

### 纹理跟 sampler2D 的关联

首先，OpenGL ES 所有的资源，都分配了一个数字标示符，之后用标示符来操作资源。比如应用中有 2 个 Shader 程序（一个程序由 Vertex Shader 和 Fragment Shader 链接而成）有不同的标示符。glUseProgram 传进相应的标示符时就可以切换当前的程序。纹理也一样，每个纹理也有一个标示符，使用 glBindTexture 来切换当前的纹理。

但 Shader 程序跟纹理有点不一样。**同一时刻只可以使用一个程序，但同一时刻可以使用多个纹理。纹理单元用来处理纹理，既然可以同时使用多个纹理，就会有多个纹理单元（Texture Unit）。**

**这样处理纹理之前，就需要两个步骤：**

* 选择一个纹理单元，作为当前的纹理单元。对应于接口 glActiveTexture。
* 选择一个纹理，放进当前的纹理单元中。对应于接口 glBindTexture。

**纹理单元也用数字来标示。GLTEXTURE0表示纹理单元0，GLTEXTURE1表示纹理单元1，依次类推。**

```swift
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, dogTextureId);

glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, catTextureId);
```

**上面代码就表示纹理单元0，关联了catTextureId表示的纹理。纹理单元1，关联了catTextureId表示的纹理。**

**Shader中的变量类型sampler2D，并非表示纹理。而是表示纹理单元（Texture Unit）。** 

假如 Shader 中定义了：
```swift
uniform sampler2D CC_Texture0;
uniform sampler2D CC_Texture1;
```
而用户代码调用：
```
GLuint location0 = glGetUniformLocation(programeId, "CC_Texture0");
glUniform1i(location0, 0);

GLuint location1 = glGetUniformLocation(programeId, "CC_Texture1");
glUniform1i(location1, 1);
```

**这样 Shader 中 CCTexture0 赋值为0，表示 CCTexture0 将使用纹理单元0中的纹理。CC_Texture1 赋值为 1，将使用纹理单元1中的纹理。从而CCTexture0 就跟 dogTextureId 的纹理关联起来了。 CCTexture1 就跟 catTextureId 的纹理关联起来了。**

当用户代码修改成
```swift
glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, dogTextureId);
```
这样 CC_Texture1 也就跟 dogTextureId 的纹理关联起来了。


----

纹理单元0是默认的，因此当只需要处理一张纹理的时候不用显式调用 glActiveTexture。很多例子代码写成：
```swift
glBindTexture(GL_TEXTURE_2D, dogTextureId);
GLuint location0 = glGetUniformLocation(programeId, "CC_Texture0");
glUniform1i(location0, 0);
```

从中看不出 CCTexture0 是怎么跟 dogTextureId 关联的。更惨的是 CCTexture0 的值默认也是0。有些例子更是直接写成：

```
glBindTexture(GL_TEXTURE_2D, dogTextureId);
```

之后就可以使用在 Shader 中使用 CC_Texture0 了。这种简化，初学者更加看不明白了。sampler2D 跟纹理的关联，中间多了纹理单元这个间接层。



## 三种常用id标识符

```swift
    //顶点坐标
    var positionSlot: GLuint = 0
    var texture1Slot: GLuint = 0
    var texture2Slot: GLuint = 0
    //纹理单元sampler2D的id
    var tex1_loc: Int32 = 0
    var tex2_loc: Int32 = 0
    //纹理标识符id
    var tex1: GLuint = 0
    var tex2: GLuint = 0
```

slot 插槽


## 混合

两种方式：
* blend混合的优势在于OpenGL标准支持，但是无法支持特定的alpha值；
* shader混合的优势在于可以任意操作颜色值；

```
void glBlendFunc(GLenum sfactor,GLenum dfactor);
```

使用glBlendFunc，需要通过`glEnable`开启blend功能



# 纹理变换

四种常用 2D 纹理变换，其核心思想就是调整纹理坐标和顶点坐标。

理想情况下，我们的纹理、图形、视口（view port）尺寸一致，那把纹理贴到图像上、把图形绘制到视口上这两个过程都不存在任何问题。次理想情况下，这三者的尺寸不一致但宽高比一致，那在这两个过程中使用最基本的缩放操作（即 OpenGL 默认的宽高填充，FIT_XY）也不会导致任何问题。

## 裁剪

如果我们绘制的图形是一个“全屏”的矩形（即顶点的横纵坐标范围都是 `[-1, 1]`），且纹理也是“全屏”的（即纹理的横纵坐标范围都是 `[0, 1]`），那我们就可以把纹理完全贴在图形上（图形会被全屏渲染到屏幕上）。如果我们希望裁掉各个方向上一定比例的像素，那我们可以把相应方向上的纹理坐标向中点（0.5）靠拢。例如把左右各 128 个像素的内容裁掉，我们就能不变形地把 `896*360` 的图像渲染到 `640*360` 的屏幕上。

为什么这样？下面这幅图解释了左右各裁掉 20%（`128 / 640 = 0.2`）的原理：

![a7dd163e2a5d1894bee71001417085e4](/assets/images/2017-10-07-opengl_crop_showcase.jpeg)

限制纹理坐标的取值范围来实现裁剪的效果

```swift
private void resolveCrop(float x, float y, float width, float height) {
    float minX = x;
    float minY = y;
    float maxX = minX + width;
    float maxY = minY + height;

    // left bottom
    textureCoords[0] = minX;
    textureCoords[1] = minY;
    // right bottom
    textureCoords[2] = maxX;
    textureCoords[3] = minY;
    // left top
    textureCoords[4] = minX;
    textureCoords[5] = maxY;
    // right top
    textureCoords[6] = maxX;
    textureCoords[7] = maxY;
}
```

## 翻转

我们继续使用上面提出的场景，如果我们要实现左右翻转（也称水平翻转），那么把纹理的左边两个点和右边两个点对调即可，如果我们要实现上下翻转（也称垂直翻转），把纹理的上边两个点和下边两个点对调即可。

GPUImage/Base/Framebuffer.swft/Rotation
```swift
    func textureCoordinates() -> [GLfloat] {
        switch self {
            case .noRotation: return [0.0, 0.0, 1.0, 0.0, 0.0, 1.0, 1.0, 1.0]
            case .rotateCounterclockwise: return [0.0, 1.0, 0.0, 0.0, 1.0, 1.0, 1.0, 0.0]
            case .rotateClockwise: return [1.0, 0.0, 1.0, 1.0, 0.0, 0.0, 0.0, 1.0]
            case .rotate180: return [1.0, 1.0, 0.0, 1.0, 1.0, 0.0, 0.0, 0.0]
            case .flipHorizontally: return [1.0, 0.0, 0.0, 0.0, 1.0, 1.0, 0.0, 1.0]
            case .flipVertically: return [0.0, 1.0, 1.0, 1.0, 0.0, 0.0, 1.0, 0.0]
            case .rotateClockwiseAndFlipVertically: return [0.0, 0.0, 0.0, 1.0, 1.0, 0.0, 1.0, 1.0]
            case .rotateClockwiseAndFlipHorizontally: return [1.0, 1.0, 1.0, 0.0, 0.0, 1.0, 0.0, 0.0]
        }
    }
```

![5e0ec1cd688d78efb14133e6536e0aee](/assets/images/IMG_4773.jpg)

```swift
private void resolveFlip(int flip) {
    switch (flip) {
        case Transformation.FLIP_HORIZONTAL:
            swap(textureCoords, 0, 2);
            swap(textureCoords, 4, 6);
            break;
        case Transformation.FLIP_VERTICAL:
            swap(textureCoords, 1, 5);
            swap(textureCoords, 3, 7);
            break;
        case Transformation.FLIP_HORIZONTAL_VERTICAL:
            swap(textureCoords, 0, 2);
            swap(textureCoords, 4, 6);

            swap(textureCoords, 1, 5);
            swap(textureCoords, 3, 7);
            break;
        case Transformation.FLIP_NONE:
        default:
            break;
    }
}
```

## 旋转

我们如果把纹理的顶点按左下、右下、左上、右上编号，逆时针旋转 90° 的变化如下图所示：

![0f1fe376cff75bad37b88a92d95b9d2c](/assets/images/2017-10-07-opengl_rotate_showcase.jpeg)

这里我们相当于把位置和编号进行一个错位操作，比如原图的左下 -> 右下 -> 右上 -> 左上是 0132，逆时针旋转 90° 之后则变成了 2013，我们传递给 OpenGL 的纹理坐标是按位置的，传入不同的编号顺序即可达到旋转的效果，但需要是 0123 可以旋转出来的编号顺序，包括 0132、2013、3201、1320。

所以这种思路只能实现 90/180/270 的旋转，任意角度的旋转则需要结合投影矩阵实现，不过对纹理进行非 90/180/270 角度的旋转效果会很诡异，一般用不到。比如逆时针旋转 45° 的效果如下：

![22d7d755b8f3e94e6a924cd501bc5595](/assets/images/2017-10-07-opengl_preview_rotate_315.png)

可以看到左上、右上、右下三个角的画面变成了条纹状。

```swift
private void resolveRotate(int rotation) {
    float x, y;
    switch (rotation) {
        case Transformation.ROTATION_90:
            x = textureCoords[0];
            y = textureCoords[1];
            textureCoords[0] = textureCoords[4];
            textureCoords[1] = textureCoords[5];
            textureCoords[4] = textureCoords[6];
            textureCoords[5] = textureCoords[7];
            textureCoords[6] = textureCoords[2];
            textureCoords[7] = textureCoords[3];
            textureCoords[2] = x;
            textureCoords[3] = y;
            break;
        case Transformation.ROTATION_180:
            swap(textureCoords, 0, 6);
            swap(textureCoords, 1, 7);
            swap(textureCoords, 2, 4);
            swap(textureCoords, 3, 5);
            break;
        case Transformation.ROTATION_270:
            x = textureCoords[0];
            y = textureCoords[1];
            textureCoords[0] = textureCoords[2];
            textureCoords[1] = textureCoords[3];
            textureCoords[2] = textureCoords[6];
            textureCoords[3] = textureCoords[7];
            textureCoords[6] = textureCoords[4];
            textureCoords[7] = textureCoords[5];
            textureCoords[4] = x;
            textureCoords[5] = y;
            break;
        case Transformation.ROTATION_0:
        default:
            break;
    }
}
```


![75cce694fd323b33ff8ec762df626cdf](/assets/images/IMG_4774.jpg)


## 缩放

通常缩放会有三种模式：

* FIT_XY：宽高都填充满，如果宽高比不一致，则会发生变形；
* CENTER_CROP：短边填充满，长边等比例缩放，超出部分两端裁掉；
* CENTER_INSIDE：长边填充满，短边等比例缩放，不足部分两端留黑边；

其中

* `FIT_XY` 是 OpenGL 的默认模式，我们无需做任何操作；

* 对于 `CENTER_CROP/CENTER_INSIDE` 我们则可以通过调整顶点坐标进行实现。`CENTER_CROP` 是让顶点坐标取值范围超过 `[-1, 1]`，超过部分会在被 OpenGL 裁掉；

* `CENTER_INSIDE` 是让顶点坐标取值范围小于 `[-1, 1]`，不足部分则没有内容，如果我们可以通过 `glClearColor` 调用设置无内容区域的颜色。

**裁剪、翻转、旋转这三种操作实际上都没有调整我们绘制的图形（都是“全屏”矩形），只是调整贴纹理的方式，而缩放则不调整贴纹理的方式（“全屏”纹理），它调整的是我们绘制的图形。**

```objc
private void resolveScale(int inputWidth, int inputHeight, int outputWidth, 
        int outputHeight, int scaleType) {
    if (scaleType == Transformation.SCALE_TYPE_FIT_XY) {
        // The default is FIT_XY
        return;
    }

    // Note: scale type need to be implemented by adjusting
    // the vertices (not textureCoords).
    if (inputWidth * outputHeight == inputHeight * outputWidth) {
        // Optional optimization: If input w/h aspect is the same as output's,
        // there is no need to adjust vertices at all.
        return;
    }
    //如果 inputWidth * outputHeight == inputHeight * outputWidth，那三种缩放模式其实效果一样，所以我们无需做事。


    float inputAspect = inputWidth / (float) inputHeight;
    float outputAspect = outputWidth / (float) outputHeight;

    if (scaleType == Transformation.SCALE_TYPE_CENTER_CROP) {
        if (inputAspect < outputAspect) {
            float heightRatio = outputAspect / inputAspect;
            vertices[1] *= heightRatio;
            vertices[3] *= heightRatio;
            vertices[5] *= heightRatio;
            vertices[7] *= heightRatio;
       /*
       CENTER_CROP 时，如果 inputAspect < outputAspect，即输入宽高比小于输出宽高比，则说明（竖屏时）输入宽是短边高是长边，宽填充满即顶点横坐标取值范围充满 [-1, 1]，高按比例放大即各顶点纵坐标值乘以放大系数，系数为 outputAspect / inputAspect
       */     
            
        } else {
            float widthRatio = inputAspect / outputAspect;
            vertices[0] *= widthRatio;
            vertices[2] *= widthRatio;
            vertices[4] *= widthRatio;
            vertices[6] *= widthRatio;
        }
    } else if (scaleType == Transformation.SCALE_TYPE_CENTER_INSIDE) {
        if (inputAspect < outputAspect) {
            float widthRatio = inputAspect / outputAspect;
            vertices[0] *= widthRatio;
            vertices[2] *= widthRatio;
            vertices[4] *= widthRatio;
            vertices[6] *= widthRatio;
        } else {
            float heightRatio = outputAspect / inputAspect;
            vertices[1] *= heightRatio;
            vertices[3] *= heightRatio;
            vertices[5] *= heightRatio;
            vertices[7] *= heightRatio;
        }
    }
}
```

![ce77a7f1072258847fb8f14ba0215f4d](/assets/images/IMG_4778.jpg)


# 参考

* https://zhuanlan.zhihu.com/p/20289660
* https://blog.piasy.com/2017/10/06/Open-gl-es-android-2-part-3/index.html
* OpenGL ES应用开发实践 指南 iOS卷
* https://www.jianshu.com/p/f6f7e5d4403c