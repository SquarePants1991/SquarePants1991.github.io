---
layout: post
title: Billboards
date: 2017-10-30 15:17:36 +0800
categories: 学习OpenGLES系列文章
---
### [获取示例代码](https://github.com/SquarePants1991/OpenGLESLearn){:target="_blank"}，本文代码在分支chapter29中。
***

### 前言
本文将介绍3D游戏中一种常见的技术Billboards，又可以称为公告板技术。形象的来说，就好像一个人举着牌子，无论你从哪个方向看向牌子，那个人都会把牌子朝着你的方向旋转，你永远只能看到牌子的正面。在3D游戏中，可以用来制作npc头上的名字或是任务标记，用来制作较远处的树木，路灯也是可以的。本文的例子制作的就是基于公告板的树木。下面是运行效果。不过树木属于比较特殊的公告板，它只能绕着垂直轴旋转，后面会详细介绍这一点。

![](http://upload-images.jianshu.io/upload_images/2949750-094c0627583abb2d.gif?imageMogr2/auto-orient/strip)

### 原理
公告板技术主要利用一个正方形网格，根据当前的摄像机变换信息，调整至面向摄像机的位置。这里我们主要利用摄像机向上和向右的两个向量来调整正方形网格的点。

![](http://upload-images.jianshu.io/upload_images/2949750-523f284e227c826b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
首先我们找到网格中心的位置，设其为`CenterPosition`，类型为三维向量。设我们想要的网格大小为`Size`，类型为二维向量。摄像机向上的向量为`VecUp`，向右的为`VecRight`。我们使用的正方形网格初始顶点数据如下。
```objc
- (GLfloat *)planeData {
    static GLfloat planeData[] = {
        -0.5,   0.5f,  0.0,   0,  0,  1, 0, 0,
        -0.5f,  -0.5f,  0.0,  0,  0,  1, 0, 1,
        0.5f,   -0.5f,  0.0,  0,  0,  1, 1, 1,
        0.5,    -0.5f, 0.0,   0,  0,  1, 1, 1,
        0.5f,  0.5f,  0.0,    0,  0,  1, 1, 0,
        -0.5f,   0.5f,  0.0,  0,  0,  1, 0, 0,
    };
    return planeData;
}
```
对于每个顶点，我们可以使用下面这个公式求解。
`NewPosition = CenterPosition + VecUp * OldPosition.x * Size.x + VecRight * OldPosition.y * Size.y`。这个公式的含义就是在摄像机的xy坐标系内生成一个长宽各与x，y轴平行的四边形。四边形长宽就是`Size`。

### Shader
为了支持Billboards，需要编写新的Vertex Shader。主要增加的下面的代码。完整代码在`vtx_billboard.glsl`中。
```c
vec3 cameraRightInWorldspace = vec3(cameraMatrix[0][0], cameraMatrix[1][0], cameraMatrix[2][0]);
vec3 cameraUpInWorldspace = vec3(0.0, 1.0, 0.0);
if (lockToYAxis == false) {
    cameraUpInWorldspace = vec3(cameraMatrix[0][1], cameraMatrix[1][1], cameraMatrix[2][1]);
}
vec3 vertexPositionInWorldspace = billboardCenterPosition + cameraRightInWorldspace * position.x * billboardSize.x +
cameraUpInWorldspace * position.y * billboardSize.y;
fragPosition = vertexPositionInWorldspace;
gl_Position = vp * vec4(vertexPositionInWorldspace, 1.0);
```
我们可以通过使用`cameraMatrix`的逆矩阵和摄像机初始的up，right向量相乘获得当前的up和right。不过有更简单的方法直接从`cameraMatrix`中获得。
```c
vec3 cameraRightInWorldspace = vec3(cameraMatrix[0][0], cameraMatrix[1][0], cameraMatrix[2][0]);
...
cameraUpInWorldspace = vec3(cameraMatrix[0][1], cameraMatrix[1][1], cameraMatrix[2][1]);
```
如果我们想要网格只围绕Y轴旋转的话，把`cameraUpInWorldspace`设置为`vec3(0.0, 1.0, 0.0);`即可。
最后就是使用上面提到的公式，为每个顶点重新计算位置。
```c
vec3 vertexPositionInWorldspace = billboardCenterPosition + cameraRightInWorldspace * position.x * billboardSize.x +
cameraUpInWorldspace * position.y * billboardSize.y;
```
注意这里我并没有使用modelMatrix，因为变换信息都是通过`billboardCenterPosition`等uniform变量传递或者计算出来的。

我也为Billboard重新编写了Fragment Shader，不过很简单，只是在fragment.glsl的基础上去掉了光照模型而已。最主要的就是增加了AlphaTest，完全透明的像素被直接忽略掉。
```c
void main(void) {
    vec4 diffuseColor = texture2D(diffuseMap, fragUV);
    if (diffuseColor.a == 0.0) {
        discard;
    }
    vec3 finalColor = colorWithFog(diffuseColor.rgb);
    gl_FragColor = vec4(finalColor, 1.0);
}
```

### OC代码
OC代码基本复用平面几何体Plane的代码，在绘制的时候将Billboards特有的几个参数传递进去。完整代码在`Billboard.m`中。
```objc
[glContext setUniform2fv:@"billboardSize" value:self.billboardSize];
[glContext setUniform3fv:@"billboardCenterPosition" value:self.billboardCenterPosition];
[glContext setUniform1i:@"lockToYAxis" value:self.lockToYAxis];
```

### 创建一些树
本文的代码基于雾效果那篇文章的例子编写，在`ViewController.m`中添加下面的代码生成一些树木。因为我没有判断地形的高度，所以有些树会只显示一半，你也可以在生成树的时候获取对应坐标地形的高度进行判断。
```objc
- (void)createTrees {
    NSString *vertexShaderPath = [[NSBundle mainBundle] pathForResource:@"vtx_billboard" ofType:@".glsl"];
    NSString *fragmentShaderPath = [[NSBundle mainBundle] pathForResource:@"frag_billboard" ofType:@".glsl"];
    self.treeGlContext = [GLContext contextWithVertexShaderPath:vertexShaderPath fragmentShaderPath:fragmentShaderPath];
    
    for (int cycleTime = 0; cycleTime < 8; ++cycleTime) {
        for (int angleSampleCount = 0; angleSampleCount < 9; ++angleSampleCount) {
            float angle = rand() / (float)RAND_MAX * M_PI * 2.0;
            float radius = rand() / (float)RAND_MAX * 70 + 40;
            float xloc = cos(angle) * radius;
            float zloc = sin(angle) * radius;
            [self createTree: GLKVector3Make(xloc, 5, zloc)];
        }
    }
}

- (void)createTree:(GLKVector3)position {
    GLKTextureInfo *grass = [GLKTextureLoader textureWithCGImage:[UIImage imageNamed:@"tree.png"].CGImage options:nil error:nil];
    Billboard *tree = [[Billboard alloc] initWithGLContext:self.treeGlContext texture:grass];
    [tree setBillboardCenterPosition:position];
    [tree setBillboardSize:GLKVector2Make(6.0, 10.0)];
    [tree setLockToYAxis:YES];
    [self.objects addObject:tree];
}
```

### 总结
本文主要通过构建基于Billboards的树来介绍这项技术，Billboards除了上面说到的用处之外，还可以用来渲染粒子系统中的粒子。这样只需要两个三角形就可以生成一个360度都可以完整看到的粒子了。
