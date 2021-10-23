---
title: "OpenGLES iOS 纹理渲染教程二"
description: "OpenGLES是 OpenGL 的子集，针对手机、PDA 和游戏主机等嵌入式设备而设计。本系列文章以iOS平台进行纹理渲染为实践目标，在查找了大量资料后进行整理而得，本篇是系列的第二篇。"
category: programming
tags: iOS,Swift,OpenGLES,Texture
---

# 多边形渲染

## 怎么渲染多变形

由于 OpenGL ES 只能渲染三角形，因此**多边形需要由多个三角形来组成**。

![9baaf207dde01db29143d13fb6a1f66c](/assets/images/370D9B50-4E2A-4403-AB33-E3864AD45B8D.jpg)

如图所示，一个五边形，我们可以把它拆分成 3 个三角形来渲染。

渲染一个三角形，我们需要一个保存 3 个顶点的数组。这意味着我们渲染一个五边形，需要用 9 个顶点。而且我们可以看到，其中 V0 、 V2 、V3 都是重复的顶点，显得有点冗余。

那么有没有更简单的方式，可以让我们复用之前的顶点呢？答案是肯定的。

**在 OpenGL ES 中，对于三角形有 3 种绘制模式。在给定的顶点数组相同的情况下，可以指定我们想要的连接方式。** 如下图所示：

![8acf606112d866436aa0d4710c1db6b9](/assets/images/7077E79D-A5AB-4418-9C2D-A181AAC77AFC.jpg)

### 1. GL_TRIANGLES

GL_TRIANGLES 就是我们一开始说的方式，**没有复用顶点**，以每三个顶点绘制一个三角形。第一个三角形使用 V0 、 V1 、V2 ，第二个使用 V3 、 V4 、V5 ，以此类推。如果顶点的个数不是 3 的倍数，那么最后的 1 个或者 2 个顶点会被舍弃。

### 2. GL_TRIANGLE_STRIP

GL_TRIANGLE_STRIP 在绘制三角形的时候，**会复用前两个顶点**。第一个三角形依然使用 V0 、 V1 、V2 ，第二个则会使用 V1 、 V2 、V3，以此类推。第 n 个会使用 V(n-1) 、 V(n) 、V(n+1) 。

### 3. GL_TRIANGLE_FAN

GL_TRIANGLE_FAN 在绘制三角形的时候，**会复用第一个顶点和前一个顶点**。第一个三角形依然使用 V0 、 V1 、V2 ，第二个则会使用 V0 、 V2 、V3，以此类推。第 n 个会使用 V0 、 V(n) 、V(n+1) 。这种方式看上去像是在绕着 V0 画扇形。


# 顶点绘制（VAO & VBO & EBO）

* VBO - Vertex Buffer Object 顶点缓存对象，管理一段顶点数据

* VAO - Vertex Array Object 顶点数组对象，管理着众多的 VBO。 **仅 OpenGL ES 3.0 之后支持**

* EBO - Element Buffer Object 索引（index）缓存对象，可以对顶点进行复用，提高性能

## VBO(Vertex Buffer Object)

```swift
static func createVBO(_ target:GLenum, _ usage: Int, _ datSize: Int, data:UnsafeRawPointer!) ->GLuint {
var vbo:GLuint = 0

glGenBuffers(1, &vbo) //第一参数指定生成缓存标识符的数量，第二个参数是一个指针，指向生成标识符的内存保存地址，当前情况下，一个标识符被生成，并保存在vbo实例变量中
glBindBuffer(target, vbo) //绑定指定标识符的缓存到当前缓存

//glBufferData函数复制应用的顶点数据到当前上下文所绑定的顶点缓存中
glBufferData(target, //Initialize buffer contents
            datSize, //number of bytes to copy 
            data,    //addressof bytes to copy
            GLenum(usage)) //hint:chache in GPU memory

return vbo
}

let vertices: [GLfloat] = [
            1,  1, 1.0, 0.0,   // 右上
            1, -1, 1.0, 1.0,   // 右下
            -1, -1, 0.0, 1.0,  // 左下
            -1, -1, 0.0, 1.0,  // 左下
            -1,  1, 0.0, 0.0,  // 左上
            1,  1, 1.0, 0.0,   // 右上
        ]
        
vertexCount = vertices.count
vbo = BaseGLKView.createVBO(GLenum(GL_ARRAY_BUFFER), 
                    Int(GL_STATIC_DRAW), 
                    MemoryLayout<GLfloat>.size * vertices.count, 
                    data: vertices)
```

OpenGL ES保存不同类型的缓存标识符到当前 OpenGL ES 上下文的不同部位。但是，**在任意时刻每种类型只能绑定一个缓存。如果在这个例子中使用了两个项点属性数组缓存，那么在同一时刻它们不能都被绑定。**

`glBindBuffer()`的第一个参数是一个常量，用于指定要绑定哪一种类型的缓存。OpenGL ES 2.0对于`glBindBuffer()`的实现只支持两种类型的缓存，`GL_ARRAY_BUFFER` 和 `GL_ELEMENT_ARRAY_BUFFER`。`GL_ARRAY_BUFFER` 类型用于指定一个顶点属性数组，`glBindBuffer()`的第二个参数是要绑定的缓存的标识符。

`glBufferData`函数复制应用的顶点数据到当前上下文所绑定的顶点缓存中。`glBufferData`的第一个参数用于指定要更新当前上下文中所绑定的是哪一个缓存。第二个参数指定要复制进这个缓存的字节的数量。第三个参数是要复制的字节的地址。最后，第4个参数提示了缓存在未来的运算中可能将会被怎样使用。`GL_STATIC_DRAW `提示会告诉上下文，缓存中的内容适合复制到 GPU 控制的内存，因为很少对其进行修改。这个信息可以帮助 OpenGL ES 优化内存使用。使用 `GL_DYNAMIC_DRAW`作为提示会告诉上下文，缓存内的数据会频繁改变，同时提示OpenGL ES以不同的方式来处理缓存的存储。

## 绘制过程


>使用缓存的过程可以分为 7 步：
>
>1. 生成（Generate）：生成缓存标识符 glGenBuffers()
>2. 绑定（Bind）：对接下来的操作，绑定一个缓存 glBindBuffer()
>3. 缓存数据（Buffer Data）：从CPU的内存复制数据到缓存的内存 glBufferData() / glBufferSubData()
>4. 启用（Enable）或者禁止（Disable）：设置在接下来的渲染中是否要使用缓存的数据 glEnableVertexAttribArray() / glDisableVertexAttribArray()
>5. 设置指针（Set Pointers）：告知缓存的数据类型，及相应数据的偏移量 glVertexAttribPointer()
>6. 绘图（Draw）：使用缓存的数据进行绘制 glDrawArrays() / glDrawElements()
>7. 删除（Delete）：删除缓存，释放资源 glDeleteBuffers()

4-7是涉及具体的绘制过程

```swift
var positionSlot: GLuint = 0

glEnableVertexAttribArray(positionSlot)
glVertexAttribPointer(positionSlot, 
                    2, 
                    GLenum(GL_FLOAT), 
                    GLboolean(GL_FALSE), 
                    GLsizei(MemoryLayout<GLfloat>.size * 4), 
                    UnsafeRawPointer(bitPattern: 0))
```

在第4步中，通过调用`glEnableVertexAttribArray()`来启动顶点缓存渲染操作。OpenGL ES 所支持的每一个谊染操作都可以单独地使用保存在当前 OpenGL ES 上下文中的设置来开启或关闭。

在第5步中，`glVertextAttribPointer()`函数会告诉 OpenGL ES 顶点数据在哪里，以及怎么解释为每个顶点保存的数据。

* 第一个参数指示当前绑定的缓存包含每个顶点的位置信息。
* 第二个参数指示每个位置有几个部分。
* 第三个参数告诉 OpenGL ES 每个部分都保存为一个浮点类型的值。
* 第四个参数告诉OpenGL ES 小数点固定数据是否可以被改变。例子中会使用小数点固定的数据，因此这个参数值是 GL FALSE。
* 第五个参数叫做“步幅”，它指定了每个顶点的保存需要主少个字节。换句话说，步幅指定了 GPU 从一个顶点的内存开始位置转到下一个项点的内存开始位置需要跳过多小字节。
* 最后一个参数是 NULL，这告诉OpenGL ES 可以从当前绑定的顶点缓存的开始位置访问顶点数据。

```swift
glDrawArrays(GLenum(GL_TRIANGLES), 0, vertexCount);
```

在第6步中，通过调用 `gIDrawArrays()`来执行绘图。`gIDrawArrays()`的第一个参数会告诉 GPU 怎么处理在鄉定的顶点缓存内的项点数据。这个例子会指示 OpenGL ES 去谊染三角形。`gIDrawArrays()`的第二个参数和第三个参数分别指定缓存内的需要渲染的第一个顶点的位置和需要渲染的顶点的数量。

请记住 GPU 运算与CPU 运算是异步的。在这个例子中的所有代码都是运行在CPU上的，然后在需要进一步处理的时候向 GPU 发送命令。GPU 可能也会处理发送自iOS 的 Core Animation 的命令，因此在任何给定的时刻GPU 总共要执行多少处理并不一定。



# 向量与矩阵变换


## 向量

向量有一个方向(Direction)和大小(Magnitude，也叫做强度或长度)。你可以把向量想像成一个藏宝图上的指示：“向左走10步，向北走3步，然后向右走5步”；“左”就是方向，“10步”就是向量的长度。

**那么这个藏宝图的指示一共有3个向量。向量可以在任意维度(Dimension)上，但是我们通常只使用2至4维。如果一个向量有2个维度，它表示一个平面的方向(想象一下2D的图像)，当它有3个维度的时候它可以表达一个3D世界的方向。**

数学家喜欢在字母上面加一横表示向量，比如说v¯。当用在公式中时它们通常是这样的：

```math
\bar{v} = \begin{pmatrix} \color{red}x \\ \color{green}y \\ \color{blue}z \end{pmatrix}
```


### 向量与标量运算

标量(Scalar)只是一个数字（或者说是仅有一个分量的向量）。当把一个向量加/减/乘/除一个标量，我们可以简单的把向量的每个分量分别进行该运算。对于加法来说会像这样:

```math
\begin{pmatrix} \color{red}1 \\ \color{green}2 \\ \color{blue}3 \end{pmatrix} + x = \begin{pmatrix} \color{red}1 + x \\ \color{green}2 + x \\ \color{blue}3 + x \end{pmatrix}
```

其中的+可以是+，-，·或÷，其中·是乘号。注意－和÷运算时不能颠倒（标量-/÷向量），因为颠倒的运算是没有定义的。

### 向量取反

对一个向量取反(Negate)会将其方向逆转。一个指向东北的向量取反后就指向西南方向了。我们在一个向量的每个分量前加负号就可以实现取反了（或者说用-1数乘该向量）:

```math
-\bar{v} = -\begin{pmatrix} \color{red}{v_x} \\ \color{blue}{v_y} \\ \color{green}{v_z} \end{pmatrix} = \begin{pmatrix} -\color{red}{v_x} \\ -\color{blue}{v_y} \\ -\color{green}{v_z} \end{pmatrix}
```

### 向量加减

向量的加法可以被定义为是分量的(Component-wise)相加，即将一个向量中的每一个分量加上另一个向量的对应分量：

```math
\bar{v} = \begin{pmatrix} \color{red}1 \\ \color{green}2 \\ \color{blue}3 \end{pmatrix}, \bar{k} = \begin{pmatrix} \color{red}4 \\ \color{green}5 \\ \color{blue}6 \end{pmatrix} \rightarrow \bar{v} + \bar{k} = \begin{pmatrix} \color{red}1 + \color{red}4 \\ \color{green}2 + \color{green}5 \\ \color{blue}3 + \color{blue}6 \end{pmatrix} = \begin{pmatrix} \color{red}5 \\ \color{green}7 \\ \color{blue}9 \end{pmatrix}

```
向量v = (4, 2)和k = (1, 2)可以直观地表示为：

![2cb09322acc1c06a9475a3698b624bd7](/assets/images/07844A13-0D99-4284-817F-580D65C1B19D.png)

就像普通数字的加减一样，向量的减法等于加上第二个向量的相反向量：

```math
\bar{v} = \begin{pmatrix} \color{red}1 \\ \color{green}2 \\ \color{blue}3 \end{pmatrix}, \bar{k} = \begin{pmatrix} \color{red}4 \\ \color{green}5 \\ \color{blue}6 \end{pmatrix} \rightarrow \bar{v} + -\bar{k} = \begin{pmatrix} \color{red}1 + (-\color{red}{4}) \\ \color{green}2 + (-\color{green}{5}) \\ \color{blue}3 + (-\color{blue}{6}) \end{pmatrix} = \begin{pmatrix} -\color{red}{3} \\ -\color{green}{3} \\ -\color{blue}{3} \end{pmatrix}

```
两个向量的相减会得到这两个向量指向位置的差。这在我们想要获取两点的差会非常有用。

![bc042b6aee810f87ca1c22c25d389928](/assets/images/FAF01C43-A79B-4685-B179-9F9BF57E4F9A.png)

### 长度
我们使用勾股定理(Pythagoras Theorem)来获取向量的长度(Length)/大小(Magnitude)。如果你把向量的x与y分量画出来，该向量会和x与y分量为边形成一个三角形:

![f532bfab06da6d4d73095db58f7c1dce](/assets/images/DBEA8E19-1E7C-4246-8996-77A6B990B392.png)

因为两条边（x和y）是已知的，如果希望知道斜边v¯的长度，我们可以直接通过勾股定理来计算：

```math
||\color{red}{\bar{v}}|| = \sqrt{\color{green}x^2 + \color{blue}y^2}
```

||v¯|| 表示向量v¯的长度，我们也可以加上z2把这个公式拓展到三维空间。

例子中向量(4, 2)的长度等于：

```math
||\color{red}{\bar{v}}|| = \sqrt{\color{green}4^2 + \color{blue}2^2} = \sqrt{\color{green}16 + \color{blue}4} = \sqrt{20} = 4.47

```
结果是4.47。

有一个特殊类型的向量叫做单位向量(Unit Vector)。单位向量有一个特别的性质——它的长度是1。我们可以用任意向量的每个分量除以向量的长度得到它的单位向量n̂ ：

```math
\hat{n} = \frac{\bar{v}}{||\bar{v}||}
```

我们把这种方法叫做一个向量的标准化(Normalizing)。单位向量头上有一个^样子的记号。通常单位向量会变得很有用，特别是在我们只关心方向不关心长度的时候（如果改变向量的长度，它的方向并不会改变）。


### 向量相乘

两个向量相乘是一种很奇怪的情况。普通的乘法在向量上是没有定义的，因为它在视觉上是没有意义的。但是在相乘的时候我们有两种特定情况可以选择：一个是点乘(Dot Product)，记作v¯⋅k¯，另一个是叉乘(Cross Product)，记作v¯×k¯。

#### 点乘

两个向量的点乘等于它们的数乘结果乘以两个向量之间夹角的余弦值。可能听起来有点费解，我们来看一下公式：

```math
\bar{v} \cdot \bar{k} = ||\bar{v}|| \cdot ||\bar{k}|| \cdot \cos \theta
```

它们之间的夹角记作θ。为什么这很有用？想象如果v¯和k¯都是单位向量，它们的长度会等于1。这样公式会有效简化成：

```math
\bar{v} \cdot \bar{k} = 1 \cdot 1 \cdot \cos \theta = \cos \theta
```
现在点积只定义了两个向量的夹角。你也许记得90度的余弦值是0，0度的余弦值是1。使用点乘可以很容易测试两个向量是否正交(Orthogonal)或平行（正交意味着两个向量互为直角）。

所以，我们该如何计算点乘呢？点乘是通过将对应分量逐个相乘，然后再把所得积相加来计算的。两个单位向量的（你可以验证它们的长度都为1）点乘会像是这样：

```math
\begin{pmatrix} \color{red}{0.6} \\ -\color{green}{0.8} \\ \color{blue}0 \end{pmatrix} \cdot \begin{pmatrix} \color{red}0 \\ \color{green}1 \\ \color{blue}0 \end{pmatrix} = (\color{red}{0.6} * \color{red}0) + (-\color{green}{0.8} * \color{green}1) + (\color{blue}0 * \color{blue}0) = -0.8
```
要计算两个单位向量间的夹角，我们可以使用反余弦函数cos−1 ，可得结果是143.1度。现在我们很快就计算出了这两个向量的夹角。点乘会在计算光照的时候非常有用。


## 矩阵

矩阵就是一个矩形的数字、符号或表达式数组。矩阵中每一项叫做矩阵的元素(Element)。下面是一个2×3矩阵的例子：

```math
\begin{bmatrix} 1 & 2 & 3 \\ 4 & 5 & 6 \end{bmatrix}
```

矩阵可以通过(i, j)进行索引，i是行，j是列，这就是上面的矩阵叫做2×3矩阵的原因（3列2行，也叫做矩阵的维度(Dimension)）。这与你在索引2D图像时的(x, y)相反，获取4的索引是(2, 1)（第二行，第一列）（译注：如果是图像索引应该是(1, 2)，先算列，再算行）。


### 矩阵的加减

矩阵与标量之间的加减定义如下：

```math
\begin{bmatrix} 1 & 2 \\ 3 & 4 \end{bmatrix} + \color{green}3 = \begin{bmatrix} 1 + \color{green}3 & 2 + \color{green}3 \\ 3 + \color{green}3 & 4 + \color{green}3 \end{bmatrix} = \begin{bmatrix} 4 & 5 \\ 6 & 7 \end{bmatrix}

```


### 矩阵相乘

```math
\begin{bmatrix} \color{red}1 & \color{red}2 \\ \color{green}3 & \color{green}4 \end{bmatrix} \cdot \begin{bmatrix} \color{blue}5 & \color{purple}6 \\ \color{blue}7 & \color{purple}8 \end{bmatrix} = \begin{bmatrix} \color{red}1 \cdot \color{blue}5 + \color{red}2 \cdot \color{blue}7 & \color{red}1 \cdot \color{purple}6 + \color{red}2 \cdot \color{purple}8 \\ \color{green}3 \cdot \color{blue}5 + \color{green}4 \cdot \color{blue}7 & \color{green}3 \cdot \color{purple}6 + \color{green}4 \cdot \color{purple}8 \end{bmatrix} = \begin{bmatrix} 19 & 22 \\ 43 & 50 \end{bmatrix}
```

![ac7e2ff6334141f104926f7f01dc064a](/assets/images/EA9EF954-8A75-46C5-8CF0-C429DFD3FDB7.png)


相乘还有一些限制：

1. 只有当左侧矩阵的列数与右侧矩阵的行数相等，两个矩阵才能相乘。
2. 矩阵相乘不遵守交换律(Commutative)，也就是说A⋅B≠B⋅A。


# GLKit

## 组成
![33fd416b3565ea2f5486bc1bf243f989](/assets/images/1.png)

## 核心概念
![e8f72696850ab79e366ea6a469a9ad65](/assets/images/1-4.png)

## GLKBaseEffect

GLKBaseEffect旨在简化当今许多OpenGL应用程序常见的视觉效果。 对于iOS，GLKBaseEffect至少需要OpenGL ES 2.0，对于OS X，GLKBaseEffect至少需要一个OpenGL Core Profile。 在实例化和初始化GLKBaseEffect对象之前，必须创建一个合适的上下文并使其成为最新的上下文。

GLKBaseEffect旨在与自定义的OpenGL代码自由地互操作。 它还旨在最小程度上影响OpenGL状态设置。 在初始化GLKBaseEffect对象及其属性时，会保留OpenGL状态设置。

**GLKBaseEffect是基于程序的，通过其底层GLSL程序的绑定**，当 - [GLKBaseEffect prepareToDraw]被调用时，它会隐式地修改GL_CURRENT_PROGRAM状态设置。 出于性能原因，GL_CURRENT_PROGRAM不会被GLKBaseEffect保存和恢复，
出于性能原因，GL_CURRENT_PROGRAM不会被GLKBaseEffect保存和恢复，因此该类的客户端必须在调用[GLKBaseEffect prepareToDraw]之前或之后设置/保存/恢复GL_CURRENT_PROGRAM，因为它们适用于其应用程序。

如果已经指定了纹理属性并被启用，GLKBaseEffect也将修改OpenGL状态元素GL_TEXTURE_BINDING_2D。 此状态也必须以上述GL_CURRENT_PROGRAM描述的方式进行处理。

GLKBaseEffect使用命名顶点属性，因此当配置/启用/绑定要与GLKBaseEffect一起使用的顶点属性数据时，客户端应用程序可以引用以下顶点属性名称：

```swift
  GLKVertexAttribPosition      0
  GLKVertexAttribNormal        1
  GLKVertexAttribColor         2
  GLKVertexAttribTexCoord0     3
  GLKVertexAttribTexCoord1     4

 // 请注意，GLKVertexAttribNormal的法线总是标准化。
```

使用GLKBaseEffect的4个规范步骤是：

```swift
  (1) 分配并初始化GLKBaseEffect的一个实例
  directionalLightEffect = [[GLKBaseEffect alloc] init];

  (2) 在效果上设置所需的属性
  // 配置灯效果
  directionalLightEffect.light0.position = lightPosition;
  directionalLightEffect.light0.diffuseColor = diffuseColor;
  directionalLightEffect.light0.ambientColor = ambientColor;

  // 配置材料
  directionalLightEffect.material.diffuseColor = materialDiffuseColor;
  directionalLightEffect.material.ambientColor = materialAmbientColor;
  directionalLightEffect.material.specularColor = materialSpecularColor;
  directionalLightEffect.material.shininess = 10.0;

  (3) 使用要绘制的模型或场景的顶点数组对象初始化顶点属性/顶点数组状态.

  glGenVertexArraysOES(1, &vaoName);
  glBindVertexArrayOES(vaoName);

  // 为每个顶点属性创建和初始化VBO
  // 下面的例子显示了一个建立position vertex属性的例子。
  // 对每个附加的所需属性重复以下步骤：normal，color，texCoord0，texCoord1。
  
  glGenBuffers(1, &positionVBO);
  glBindBuffer(GL_ARRAY_BUFFER, positionVBO);
  glBufferData(GL_ARRAY_BUFFER, vboSize, dataBufPtr, GL_STATIC_DRAW);
  glVertexAttribPointer(GLKVertexAttribPosition, size, type, normalize, stride, NULL);
  glEnableVertexAttribArray(GLKVertexAttribPosition);

  ... 对其他所需的顶点属性重复上述步骤

  glBindVertexArrayOES(0);   // unbind the VAO we created above

  (4) 对于绘制的每个框架：更新每帧更改的属性。 通过调用 - [GLKBaseEffect prepareToDraw]同步更改的效果状态。 用效果绘制模型

  directionalLightEffect.transform.modelviewMatrix = modelviewMatrix;
  [directionalLightEffect prepareToDraw];
  glBindVertexArrayOES(vaoName);
  glDrawArrays(GL_TRIANGLE_STRIP, 0, vertCt);
```
# 着色器Shader


## 整体流程

![331b2378fbf7091f559074edde02ad6e](/assets/images/截屏2021-10-12 16.00.46.png)

## 着色器编写

首先，我们需要自己编写着色器，包括顶点着色器和片段着色器，使用的语言是 GLSL 。

新建一个文件，一般顶点着色器用后缀 .vsh ，片段着色器用后缀 .fsh 

顶点着色器的代码如下：

```c
attribute vec4 Position;
attribute vec2 TextureCoords;
varying vec2 TextureCoordsVarying;

void main (void) {
    gl_Position = Position;
    TextureCoordsVarying = TextureCoords;
}
```

片段着色器的代码如下：

```c
precision mediump float;

uniform sampler2D Texture;
varying vec2 TextureCoordsVarying;

void main (void) {
    vec4 mask = texture2D(Texture, TextureCoordsVarying);
    gl_FragColor = vec4(mask.rgb, 1.0);
}
```

GLSL 是类 C 语言写成，如果学习过 C 语言，上手是很快的。下面对这两个着色器的代码做一下简单的解释。

attribute 修饰符只存在于顶点着色器中，用于储存每个顶点信息的输入，比如这里定义了 Position 和 TextureCoords ，用于接收顶点的位置和纹理信息。

vec4 和 vec2 是数据类型，分别指四维向量和二维向量。

varying 修饰符指顶点着色器的输出，同时也是片段着色器的输入，要求顶点着色器和片段着色器中都同时声明，并完全一致，则在片段着色器中可以获取到顶点着色器中的数据。

gl_Position 和 gl_FragColor 是内置变量，对这两个变量赋值，可以理解为向屏幕输出片段的位置信息和颜色信息。

precision 可以为数据类型指定默认精度，precision mediump float 这一句的意思是将 float 类型的默认精度设置为 mediump 。

uniform 用来保存传递进来的只读值，该值在顶点着色器和片段着色器中都不会被修改。顶点着色器和片段着色器共享了 uniform 变量的命名空间，uniform 变量在全局区声明，同个 uniform 变量在顶点着色器和片段着色器中都能访问到。

**sampler2D 是纹理句柄类型，保存传递进来的纹理。**

**texture2D() 方法可以根据纹理坐标，获取对应的颜色信息。**

那么这两段代码的含义就很明确了，顶点着色器将输入的顶点坐标信息直接输出，并将纹理坐标信息传递给片段着色器；片段着色器根据纹理坐标，获取到每个片段的颜色信息，输出到屏幕。

## 着色器的编译链接

对于写好的着色器，需要我们在程序运行的时候，动态地去编译链接。编译一个着色器的代码也比较固定，这里通过后缀名来区分着色器类型，直接看代码：

```swift
- (GLuint)compileShaderWithName:(NSString *)name type:(GLenum)shaderType {
    // 查找 shader 文件
    NSString *shaderPath = [[NSBundle mainBundle] pathForResource:name ofType:shaderType == GL_VERTEX_SHADER ? @"vsh" : @"fsh"]; // 根据不同的类型确定后缀名
    NSError *error;
    NSString *shaderString = [NSString stringWithContentsOfFile:shaderPath encoding:NSUTF8StringEncoding error:&error];
    if (!shaderString) {
        NSAssert(NO, @"读取shader失败");
        exit(1);
    }
    
    // 创建一个 shader 对象
    GLuint shader = glCreateShader(shaderType);
    
    // 获取 shader 的内容
    const char *shaderStringUTF8 = [shaderString UTF8String];
    int shaderStringLength = (int)[shaderString length];
    glShaderSource(shader, 1, &shaderStringUTF8, &shaderStringLength);
    
    // 编译shader
    glCompileShader(shader);
    
    // 查询 shader 是否编译成功
    GLint compileSuccess;
    glGetShaderiv(shader, GL_COMPILE_STATUS, &compileSuccess);
    if (compileSuccess == GL_FALSE) {
        GLchar messages[256];
        glGetShaderInfoLog(shader, sizeof(messages), 0, &messages[0]);
        NSString *messageString = [NSString stringWithUTF8String:messages];
        NSAssert(NO, @"shader编译失败：%@", messageString);
        exit(1);
    }
    
    return shader;
}
```

顶点着色器和片段着色器同样都需要经过这个编译的过程，编译完成后，还需要生成一个着色器程序，将这两个着色器链接起来，代码如下：


```swift
- (GLuint)programWithShaderName:(NSString *)shaderName {
    // 编译两个着色器
    GLuint vertexShader = [self compileShaderWithName:shaderName type:GL_VERTEX_SHADER];
    GLuint fragmentShader = [self compileShaderWithName:shaderName type:GL_FRAGMENT_SHADER];
    
    // 挂载 shader 到 program 上
    GLuint program = glCreateProgram();
    glAttachShader(program, vertexShader);
    glAttachShader(program, fragmentShader);
    
    // 链接 program
    glLinkProgram(program);
    
    // 检查链接是否成功
    GLint linkSuccess;
    glGetProgramiv(program, GL_LINK_STATUS, &linkSuccess);
    if (linkSuccess == GL_FALSE) {
        GLchar messages[256];
        glGetProgramInfoLog(program, sizeof(messages), 0, &messages[0]);
        NSString *messageString = [NSString stringWithUTF8String:messages];
        NSAssert(NO, @"program链接失败：%@", messageString);
        exit(1);
    }
    return program;
}
```

这样，我们只要将两个着色器命名统一，按照规范添加后缀名。然后将着色器名称传入这个方法，就可以获得一个编译链接好的着色器程序。


## 传入参数

有了着色器程序后，我们就需要往程序中传入数据，首先要获取着色器中定义的变量，具体操作如下：

注：不同类型的变量获取方式不同。

```c
GLuint positionSlot = glGetAttribLocation(program, "Position");
GLuint textureSlot = glGetUniformLocation(program, "Texture");
GLuint textureCoordsSlot = glGetAttribLocation(program, "TextureCoords");
```

其中
```
GLint glGetAttribLocation（GLuint program,const GLchar *name）;
```
program指定查询的程序对象，name指定要查询位置的属性变量的名称，返回的就是属性变量的位置。


```
GLint glGetUniformLocation( GLuint program,const GLchar *name);
```
查询uniform变量的位置值。我们为查询函数提供着色器程序和uniform的名字。如果glGetUniformLocation返回-1就代表没有找到这个位置值。
# 参考

* [矩阵变换](https://learnopengl-cn.github.io/01%20Getting%20started/07%20Transformations/#_14)
* https://developer.apple.com/documentation/glkit/glkbaseeffect
* https://www.jianshu.com/p/33ee72f9967b
* http://www.lymanli.com/2019/02/17/ios-opengles-render-texture/

