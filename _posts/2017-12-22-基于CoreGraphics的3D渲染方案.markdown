---
layout: post
title: 基于CoreGraphics的3D渲染方案
date: 2017-12-22 21:42:35 +0800
categories: iOS开发
---

### 前言
今年的首要研究对象OpenGL基本研究的差不多了，突发奇想，想用CoreGraphics根据OpenGL的渲染流水线，渲染出3D图形来。折腾了2天，写出了个demo，效果如下。
![](http://upload-images.jianshu.io/upload_images/2949750-6dccdb7b425a517f.gif)
> 其实这种通过2D渲染引擎渲染3D的技术方案在Flash时代我就听说了，但是当时对于3D技术不是很了解，并没有做深入研究。

### 原理
在OpenGL中，每个顶点通过Vertex Shader的处理，被处理成基本的绘制图形，比如三角形，再通过Fragment Shader处理各个像素点的颜色。最终以透视的效果呈现在屏幕上（当然如果你用了正交矩阵就没透视啦）。根据上面的原理，我在渲染方案中定义了两种基本图形，线和多边形。线由2个顶点组成，多边形由3到多个顶点组成。通过MVP矩阵对顶点进行变换，然后用CoreGraphics绘制顶点变换后的图形。上图中的正方体就是由6个四边形组成，锥体则是4个三角形组成。

### 基本绘制图形
每个基本绘制图形都会实现下面的协议，`material`是图形的样式，包括颜色，线条粗细等，`transform`方法用来对组成图形的顶点进行变换，并返回变换后的图形。`sortZRef`会返回图形在z方向排序的参考值，这个主要用来弥补CoreGraphics中无法进行Depth Test的缺陷。不过目前的参考值计算方案还是有问题的，仅仅是计算了所有顶点变换后z的平均值而已。
```swift
public protocol HT3DElement {
    var material: HT3DMaterial { get set }
    func transform(matrix: GLKMatrix4) -> Self
    func sortZRef() -> Float
}
```
下面我们来看看图形-线的实现。
```swift
public var startPoint: GLKVector3
public var endPoint: GLKVector3
public var material: HT3DMaterial
public func transform(matrix: GLKMatrix4) -> HT3DLineElement {
    let newStartPoint = matrix * GLKVector4.init(vector3: startPoint, w: 1)
    let newEndPoint = matrix * GLKVector4.init(vector3: endPoint, w: 1)
    return HT3DLineElement.init(startPoint: (newStartPoint / newStartPoint.w).xyz, endPoint:
(newEndPoint
            / newEndPoint.w).xyz, material: material)
}
public func sortZRef() -> Float {
    return (startPoint.z + endPoint.z) / 2.0
}
```
在使用矩阵对顶点变换后，要重新把顶点变换到屏幕空间，所以将顶点除以它的w。`xyz`是自定义的扩展，获取4维向量的前3维。
```swift
(newStartPoint / newStartPoint.w).xyz
```
`sortZRef`的实现正如上面所说，求z的平均值。

### 几何体Geometry
Geometry由一组基本图形组成，比如一个正方体。Geometry提供`modelMatrix`对这一组基本图形进行变换。它的存在让我们可以为每一组基本图形提供不同的模型变换。同时它也肩负着管理图形材质的任务。可以通过它的`setMaterialForElementsInRange`方法为每一个基本图形设置不同的样式。
```swift
public var elements: [HT3DElement]?
public var materials: [HT3DMaterial] = []
public var modelMatrix: GLKMatrix4 = GLKMatrix4Identity
public var material: HT3DMaterial? {
    return materials.first
}
public init(elements: [HT3DElement], material: HT3DMaterial) {
    self.elements = elements
    self.materials.append(material)
    self.setMaterialForElementsInRange(range: Range<Int>.init(uncheckedBounds: (0,
elements.count - 1)), materialIndex: 0)
}

public func setMaterialForElementsInRange(range: Range<Int>, materialIndex: Int) {
    if let material = materials[cycle: materialIndex] {
        for index in range.lowerBound...range.upperBound {
            if let element = self.elements?[safe: index] {
                var ele = element
                ele.material = material
                self.elements?[index] = ele
            }
        }
    }
}
```

### CoreGraphics渲染
为了方便其他渲染器的实现，我定义了渲染器的协议。渲染器的主要功能就是渲染基本图形的集合。
```swift
protocol HT3DRenderContext {
    func render(elements: [HT3DElement])
}
```
为了更加方便的调用渲染器代码，为该协议编写了下面的扩展方法。
```swift
extension HT3DRenderContext {
    public func render(vpMatrix: GLKMatrix4, geometries: [HT3DGeometry]) {
        var elements: [HT3DElement] = []
        for geometry in geometries {
            let _ = geometry.elements?.map {
                elements.append($0.transform(matrix: vpMatrix * geometry.modelMatrix))
            }
        }
        elements.sort { $0.sortZRef() > $1.sortZRef() }
        render(elements: elements)
    }
}
```
这样就可以很方便的使用VP（ProjectionMatrix * ViewMatrix）和几何体列表渲染了。在这个方法中，我们将基本图形的顶点使用VP和所属几何体的ModelMatrix进行变换，然后将这些基本图形按照z轴排序，从而模拟DepthTest，最后调用渲染器的渲染方法。这个方法的具体实现取决于你用什么样的渲染器。本文自然采用了CoreGraphics渲染器。渲染器代码在`HT3DCGRenderContext.swift`中。主要就是线和多边形两种基本图形的渲染，非常简单的代码。
```swift
func renderElement(context: CGContext, element: HT3DLineElement) {
    context.setStrokeColor(UIColor.fromVec3(glkVector3: element.material.lineColor).cgColor)
    context.setLineWidth(element.material.lineWidth)
    context.beginPath()
    context.move(to: convertCoordFromGLToCG(element.startPoint.cgPoint()))
    context.addLine(to: convertCoordFromGLToCG(element.endPoint.cgPoint()))
    context.strokePath()
}

func renderElement(context: CGContext, element: HT3DPolygonElement) {
    context.setFillColor(UIColor.fromVec3(glkVector3: element.material.diffuse).cgColor)
    context.setStrokeColor(UIColor.fromVec3(glkVector3: element.material.lineColor).cgColor)
    context.setLineWidth(element.material.lineWidth)

    context.beginPath()

    element.points.first.map { context.move(to: convertCoordFromGLToCG($0.cgPoint())) }
    for index in 1..<element.points.count {
        context.addLine(to: convertCoordFromGLToCG(element.points[index].cgPoint()))
    }
    if element.isClosed {
        context.closePath()
    }

    context.drawPath(using: .fillStroke)
}

func convertCoordFromGLToCG(_ from: CGPoint) -> CGPoint {
    return (from * (1, -1) + 1.0) * 0.5 * (canvasSize.width, canvasSize.height)
}
```
其中`convertCoordFromGLToCG`用于将OpenGL坐标转换成CoreGraphics中的坐标，如果你对OpenGL坐标不了解，可以去看我的[OpenGL系列教程](http://www.gltech.win/categories/%E5%AD%A6%E4%B9%A0OpenGLES%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0.html)。

> 如果你对本文的其他关于OpenGL的概念也不理解，也可以去教程里面找找答案，毕竟本文中很多OpenGL的概念我只是一笔带过。

### 整合
最后将CoreGraphics渲染器用Layer封装起来，代码在`HT3DCGLayer`中。比较简单，有兴趣的同学可以直接[前往GitHub查看](https://github.com/SquarePants1991/HT3DLayer){:target="_blank"}，如果觉得项目还凑合，别忘了Star一下哦～

> 目前这个方案还只是开始阶段，还有很多优化和不足的地方有待改进。比如使用更加精准的z轴排序算法，提供基本光照模型等等。