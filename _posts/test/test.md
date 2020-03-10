---
layout: post
title:  "第一个OpenGL ES项目"
date:   2017-06-28 16:13:25 +0800
categories: test
---!

![](../../post_images/2020-03-10-12-25-55.png)

### [获取示例代码](https://github.com/SquarePants1991/OpenGLESLearn)，本文代码在分支chapter1中。
***
    
        
最近一段时间在研究AR，为了熟悉3D渲染的相关知识，把OpenGL ES研究了一下。后续会写一些OpenGL ES系列的文章，内容上会由浅入深。

第一篇主要说一下如何在iOS里配置一个基本的OpenGL ES项目。
> 我写这篇文章的时候Xcode里面已经没有OpenGL ES的项目模板了，苹果只留下了`SpriteKit`,`SceneKit`,`Metal`的项目模板，估计是为了大力推广自家的高层和底层的SDK。所以现在只能从`Single View Application`开始。

最方便的开始方法是直接使用我的示例。
编译运行本文的示例项目，如果一切没问题，可以看到一个红色的ViewController。

![](http://upload-images.jianshu.io/upload_images/2949750-642724d5ec425df5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


接下来我们来看看这个项目里面主要有什么。

![](http://upload-images.jianshu.io/upload_images/2949750-4a874ecaccd15109.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中和OpenGL ES相关的有三块
`fragment.glsl`
`vertex.glsl`
`ViewController.h/m`

`fragment.glsl`和`vertex.glsl`是OpenGL ES2之后才有的Shader，用来对渲染流程中的两个重要环节进行编程，这个我们后面详细讲解。先看`ViewController.h/m`中的内容。
首先，`ViewController`继承自`GLKViewController`
```
#import <GLKit/GLKit.h>
@interface ViewController : GLKViewController
@end
```
`GLKViewController`会帮助我们对OpenGLES进行一些基本的配置，比如生成默认的`FrameBufferObject`，这些会在后面文章介绍。

在这个ViewController里，最重要的两个变量是
```
@property (strong, nonatomic) EAGLContext *context;
@property (assign, nonatomic) GLuint shaderProgram;
```
`viewDidLoad`中会初始化这两个变量
```
- (void)viewDidLoad {
    [super viewDidLoad];
    [self setupContext];
    [self setupShader];
}
```
`EAGLContext *context`是GLKit用来处理OpenGL ES相关配置的变量，可以初始化出兼容不同`OpenGL ES`版本的上下文环境。
```
- (void)setupContext {
    // 使用OpenGL ES2, ES2之后都采用Shader来管理渲染管线
    self.context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
    
    if (!self.context) {
        NSLog(@"Failed to create ES context");
    }
    
    GLKView *view = (GLKView *)self.view;
    view.context = self.context;
    view.drawableDepthFormat = GLKViewDrawableDepthFormat24;
    [EAGLContext setCurrentContext:self.context];
}
```
上面的代码初始化的就是OpenGL ES2的上下文环境。

`GLuint shaderProgram`可以理解为Shader编译过后使用它们的一个凭证，会在后面介绍Shader时详述。

`update`和`- (void)glkView:(GLKView *)view drawInRect:(CGRect)rect`是在`GLKViewController`中被循环调用的方法，也被称为Game Loop，一般在`update`中更新变量，`- (void)glkView:(GLKView *)view drawInRect:(CGRect)rect`中进行绘制。
```
- (void)update {
    // 距离上一次调用update过了多长时间，比如一个游戏物体速度是3m/s,那么每一次调用update，
    // 他就会行走3m/s * deltaTime，这样做就可以让游戏物体的行走实际速度与update调用频次无关
    // NSTimeInterval deltaTime = self.timeSinceLastUpdate;
    
}

- (void)glkView:(GLKView *)view drawInRect:(CGRect)rect {
   // 清空之前的绘制
    glClearColor(1, 0.2, 0.2, 1);
    glClear(GL_COLOR_BUFFER_BIT);
    
    // 使用fragment.glsl 和 vertex.glsl中的shader
    glUseProgram(self.shaderProgram);
}
```
> OpenGL ES使用缓冲区来存储像素颜色深度等信息，缓冲区可以理解为一个字节数组。OpenGL ES在绘制时会有两个缓冲区，一个用来绘制当前帧，一个被显示出来，绘制当前帧的缓冲区绘制完后会和被显示出来的进行交换，这个动作不停重复，从而达到更新显示的目的。这些操作也是GLKit帮我们做了。

`glClearColor`设置了清空缓冲区使用的颜色，`glClear(GL_COLOR_BUFFER_BIT)`中的`GL_COLOR_BUFFER_BIT`表示只清空缓冲区的颜色部分。后续用到深度缓冲区时，还得将深度缓存一起Clear掉。

`glUseProgram(self.shaderProgram)`指明了使用什么样的Shader渲染后面的代码。

在下一篇文章中就要在`glUseProgram(self.shaderProgram)`之后添加一些渲染代码，绘制出一个三角形。
