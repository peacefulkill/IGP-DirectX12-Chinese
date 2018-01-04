# Chapter 7 DRAWING IN DIRECT3D PART II

这一章节我们将会介绍一些会在本书剩余部分使用的渲染模式。
章节最开始，我们将会介绍一个关于渲染优化的东西，一个叫做`帧资源(Frame Resources)`的东西。具体来说就是我们通过一些方法来避免在每一帧中刷新指令队列，及尽量减少同步`CPU`和`GPU`使得我们能够充分使用他们。之后我们介绍一下关于分项渲染的概念。之后，我们会了解更多关于来源标记的细节，以及学习另外两个参数类型，描述符(`root descriptors`)和常量(`root constants`)。最后，我们将会绘制一些复杂的物体。

> 目标：
- 了解如何修改我们的渲染过程来避免每一帧都刷新指令队列，从而提升我们的程序效率。
- 了解另外两个来源标记的参数类型，描述符和常量。
- 了解如何生成和渲染一些基础的物体，例如网格，球体等。
- 了解如何使用`CPU`去改变顶点数据，然后使用`GPU`更新到我们的缓冲中去。

## 7.1 FRAME RESOURCES

回顾4.2，我们知道`GPU`和`CPU`的运行是互相独立的，即平行的，`CPU`构建和提交指令列表，`GPU`处理指令队列里面的指令。而我们的目的就是保持`CPU`和`GPU`不断的运行，从而能够充分使用硬件资源。在之前我们的例子中，我们会在每一帧结束的时候同步`GPU`和`CPU`，下面有两个这样做的理由。

- 在`GPU`没有执行完所有的指令队列的时候指令分配器是不能够重置的。假设我们不同步他们的话，那么`CPU`可能就在`GPU`还没有处理完第$n$帧的指令的时候就开始处理第$n+1$帧的内容，那么如果在第$n+1$帧的时候，`CPU`重置指令分配器的话，`GPU`还在处理第$n$帧的指令，那么`GPU`还没有处理的指令就会被清空。
- 如果一个常缓冲会在一个渲染指令中使用的话，那么在`GPU`执行完这个指令前是不能够使用`CPU`更新它的。在4.22中我们提到过，假设我们不同步的话，`CPU`可能在第$n+1$帧的时候更新我们的常缓冲，但是`GPU`却还没有执行第$n$帧中使用了这个常缓冲的渲染指令的话，那么当`GPU`执行到这个指令的时候，常缓冲里面的数据就已经不是第$n$帧的数据而是第$n+1$帧的数据了。

因此我们非常有必要在每帧的结束的时候同步我们的`CPU`和`GPU`，但是这样做的话，会浪费我们的效率。

- 在这一帧循环刚开始的时候可能`GPU`可能没有任何任务需要处理，要等到`CPU`提交指令列表后才可以处理。
- 在这一帧循环结束的时候，我们的`CPU`要等待我们的`GPU`执行完这一帧的所有的指令。

简单来说这样做的话在每一帧都会有一段时间`CPU`或者`GPU`处于空闲阶段。

因此我们就有了一个解决的方法，就是创建一个存有每一帧`CPU`要修改的资源环型数组。我们称这样的资源叫做帧资源。通常这个环型数组的大小我们是定位3就好了。当你`CPU`处理到第$n$帧的时候，而`GPU`还没有处理到，`CPU`可以通过我们创建的帧资源数组来提前处理下一帧的资源，以及构建指令列表等。当到底数组末尾的时候，如果`GPU`还没有处理完数组开始所代表的那一帧的内容，就等待他完成，否则就直接将数组开始接到数组末尾，这样不断循环下去。

```C++
///Code
```

这样做的话，我们虽然还是需要进行两个处理器的同步工作，但是相比来说有很大的改善。
因为我们在`GPU`还在处理第$n$帧的时候就帮他构建好了之后的指令列表，因此只要`GPU`和`CPU`的差距不是太大，那么`GPU`应该会不间断的运行，然后`CPU`可能会偶尔跑的太快要等待`GPU`处理完，这个时候你还可以用等待的额外时间去计算另外的东西。

## 7.2 RENDER ITEMS

我们在绘制一个物体的时候通常需要设置很多参数，例如要绑定的顶点和索引缓冲，常缓冲，基础图元类型等。
当我们在绘制多个物体的时候，为了简便，我们可以定义一个轻量的结构体来存储我们要绘制的一个物体的数据。
其实就是定义一个结构体来存储一些关于我们要绘制的物体的信息，使得我们在编写代码上能更加直观的编写。
比如这样的例子。

```C++
struct RenderItem{
    int VertexCount;
    int IndexCount;
    int StartVertexLocation;
    int StartIndexLocation;
    //...
};
```

当然我们还可以在里面加入一些数据，这些都是要看具体情况的，你只需要有这样的一个概念就好了。

## 7.3 PASS CONSTANTS

在渲染一个物体的时候，可能我们会在着色器中用到一些其他的信息，例如摄像机的位置，渲染目标的大小等等，因此这里我们就将这些信息都存放在一个常缓冲中，每次绘制一个物体之前就只需要更新一次缓冲就好了。当然具体情况具体分析，这样做并不代表它有什么优势。只是在不追求效率的情况下稍微方便编写代码一些。

## 7.4 SHAPE GEOMETRY

其实主要想法就是将一个模型的网格信息抽象成一个类，即我们用一个类来保存我们的模型的网格的顶点以及索引信息等。

并且之后的内容也只是教你如何使用代码生成一些简单的物体，例如球体，圆台等物体。如果感兴趣可以去看原文。

## 7.5 SHAPES DEMO

这里就是原文的例子的介绍了。主要是介绍了之前讲的内容的应用。直接去看实例代码就好了。

## 7.6 MORE ON ROOT SIGNATURES

我们之前在6.6.5中介绍了什么是来源标记(`Root Signatures`)。它定义了我们在渲染之前要绑定到渲染管道的资源类型，以及将这些资源类型和着色器的输入寄存器一一映射。具体绑定的资源就要看具体需要什么资源了，这里只是规定了资源的类型(例如`CBV`,`SRV`,`UAV`等)。并且当我们创建一个渲染管道状态的时候，我们会验证这个渲染管道状态中的来源标记和着色器是否能够互相吻合。

### 7.6.1 Root Parameters

我们知道一个来源标记能够有很多个参数，之前我们只是定义了一个参数作为描述符表来使用。这里将介绍来源标记的参数支持的类型。

- `Descriptor Table`: 描述符表，能够支持绑定一个堆里面的一个范围内的资源。
- `Root Descriptor`: 描述符，能够直接绑定资源，也就是说资源并不一定要求在堆里面，但是只支持缓冲类型。也就是说你能够使用`CBV`，但如果你使用`SRV`或者`UAV`的话他们就必须被当作一组缓冲使用，而不能被当作纹理使用。
- `Root Constant`: 直接绑定一个$32bit$大小的常量。

由于性能原因，一个来源标记的大小不能超过`64 DWORD`，而不同类型的参数有不同的大小消耗。

- `Descriptor Table`: 每个消耗`1 DWORD`。
- `Root Descriptor`: 每个消耗`2 DWORD`。
- `Root Constant`: 每个消耗`1 DWORD`。

只要我们创建的来源标记的大小不超过限制，那么如何定义就是随你的意了。

```C++
struct D3D12_ROOT_PARAMETER
{
    D3D12_ROOT_PARAMETER_TYPE ParameterType;
    union
    {
            D3D12_ROOT_DESCRIPTOR_TABLE DescriptorTable;
            D3D12_ROOT_CONSTANTS Constants;
            D3D12_ROOT_DESCRIPTOR Descriptor;
    }
    D3D12_SHADER_VISIBILITY ShaderVisibility;
};
```

- `ParameterType`: 参数类型，支持的参数大致就是描述符表，常量，`CBV`,`SRV`,`UAV`类型的描述符。
- `DescriptorTable/Constants/Descriptor`: 这里如果你不知道什么是`union`请先去学习C++的语法再说。具体你该填充哪个参数就要看你选择的参数类型是什么了。
- `ShaderVisibility`: 指定我们这个参数对着色器的可见性。这里我们默认使用`D3D12_SHADER_VISIBILITY_ALL`，如果你只是在像素着色器中使用这个参数的话，那么你也可以指定`D3D12_SHADER_VISIBILITY_PIXEL`来限制我们只能够在像素着色器中使用这个参数。不过限制在一些着色器中使用的话可以优化一些性能。

```C++
enum D3D12_SHADER_VISIBILITY{
    D3D12_SHADER_VISIBILITY_ALL = 0,
    D3D12_SHADER_VISIBILITY_VERTEX = 1,
    D3D12_SHADER_VISIBILITY_HULL = 2,
    D3D12_SHADER_VISIBILITY_DOMAIN = 3,
    D3D12_SHADER_VISIBILITY_GEOMETRY = 4,
    D3D12_SHADER_VISIBILITY_PIXEL = 5
}
```

> **这里我不确保这些参数可以使用异或或者什么方式组合。**

### 7.6.2 Descriptor Tables

```C++
struct D3D12_ROOT_DESCRIPTOR_TABLE 
{
    UINT NumDescriptorRanges;
    const D3D12_DESCRIPTOR_RANGE *pDescriptorRanges;
};

struct D3D12_DESCRIPTOR_RANGE
{
    D3D12_DESCRIPTOR_RANGE_TYPE RangeType;
    UINT NumDescriptors;
    UINT BaseShaderRegister;
    UINT RegisterSpace;
    UINT OffsetInDescriptorsFromTableStart;
};

enum D3D12_DESCRIPTOR_RANGE_TYPE
{
    D3D12_DESCRIPTOR_RANGE_TYPE_SRV = 0,
    D3D12_DESCRIPTOR_RANGE_TYPE_UAV = 1,
    D3D12_DESCRIPTOR_RANGE_TYPE_CBV = 2,
    D3D12_DESCRIPTOR_RANGE_TYPE_SAMPLER = 3
};
```

- `RangeType`: 包含的一组描述符的类型。
- `NumDescriptors`: 这一组的描述符个数。
- `BaseShaderRegister`: 从哪个着色器寄存器开始以此往后推。假设你的描述符类型是`CBV`，你设置3个描述符并且你的`BaseShaderRegister`是1的话，那么它在着色器中就对应下面的三个常缓冲。
    ```hlsl
        cbuffer cbA : register(b1) {...};
        cbuffer cbB : register(b2) {...};
        cbuffer cbC : register(b3) {...};
    ```
- `RegisterSpace`: 用于进一步区分，如果我们有两组描述符的`BaseShaderRegister`是一样的话，我们可以通过使用不同的`RegisterSpace`来将他们区分开来。例如我有两个纹理他的着色器输入寄存器都是0的话，但是他们的寄存器空间不同。如果不指定寄存器空间的话就默认为0。
    ```hlsl
        Texture2D normalMap1 : register(t0, space1);
        Texture2D normalMap2 : register(t0, space2);
    ```
- `OffsetInDescriptorsFromTableStart`: 这一组描述符距离描述符表的起始位置的偏差值。具体参见样例。
