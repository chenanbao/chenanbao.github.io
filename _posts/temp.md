###### 学习笔记 UGUI源码

学习目的：了解UGUI源码更方便为游戏性能调优和扩展项目所需特制组件

完整笔记参见博客[https://chenanbao.github.io/](https://chenanbao.github.io/)

##### [顶点辅助类]

存储Mesh所需的数据：顶点，顶点色，UV，法线，切线，和三角面顶点索引。
数据构建并不在VertexHelper中构建，而是在各个组件中构建。Image和Text组合构成。

###### [优势]
数据都用ObjectPool构成的ListPool来维护[【游戏设计模式:对象池模式】](https://gpp.tkchu.me/object-pool.html)，这样可反复使用节约内存。如果是自定义特制形状UI组件就需要借助VertexHelper来组织数据。数据最终通过CanvasRenderer.AddUIVertexStream传入引擎使用，这部分代码未开源。

##### [Image顶点构成]

Type.Simple：二个三角面构成矩形
![image](/img/simple.png)

Type.Sliced：上下左右九宫组织
![image](/img/slice.png)

Type.Tiled：瓷砖一样平铺
![image](/img/tiled.png)

Type.Filled：根据实际形状构成，一般用于进度性质形状。
![image](/img/fill.png)

##### [RawImage顶点构成]

RawImage的顶点构成和Image区别在于scale系数

texelSize的Unity一个暗黑的数字，在文档中查到shader中数据定义[_MainTex_TexelSize](https://docs.unity3d.com/Manual/SL-PropertiesInPrograms.html?_ga=2.154597616.977275482.1600655737-1408819953.1598263206) Vector4(1 / width, 1 / height, width, height)

##### [Text顶点构成]

TextGenerator通过settings（文字大小，颜色，行距，对齐，字体等）排版文字，输出顶点，再通过VertexHelper.AddUIVertexQuad逐个构造文字Mesh数据。


##### [基于顶点效果]

BaseMeshEffect是个抽象基类提供修改Mesh相关数据结构，描边和投影等效果都是基于此实现。

UGUI采用Dirty标记系统[【游戏设计模式:脏标识模式】](https://gpp.tkchu.me/dirty-flag.html)，只要控件被标记为“Dirty状态，就会强制刷新一遍，在改变了顶点相关数据都会触发重绘。

##### [投影顶点构成]

获取原有形状的顶点数据，复制一份做xy偏移后叠加到原有顶点数据中。

##### [描边顶点构成]

![image](/img/outline.png)

根据effectColor和effectDistance调整顶点,描边相当于做了上下左右四次投影，当文字多时做描边效果，顶点将几何倍数暴增。


如果大量文本使用描边和投影效果推荐使用TextMeshPro插件，其文字渲染采用Signed Distance Field(有向距离场)方式，描边投影采用shader实现。

###### [法线效果]

根据坐标设置uv1坐标（法线贴图坐标），为文字等组件添加法线贴图效果

###### [渐变文字-Unity未支持]

计算出文字的上下边界，Color32.Lerp计算线性颜色修改到顶点色。

![image](/img/GradientTextComponent.png)


##### [GraphicRegistry]

GraphicRegistry管理所有Graphic,内部通过IndexedSet维护数据


##### [Graphic]

仅当OnEnable()，OnCanvasHierarchyChanged()，OnTransformParentChanged() 时触发查找第一个激活启用的Canvas，并以此为key将Graphic注册到GraphicRegistry。

当动画改变时触发SetAllDirty。

在组件激活(IsActive())的情况下，SetLayoutDirty触发LayoutRebuilder.MarkLayoutForRebuild(rectTransform)，SetVerticesDirty触发CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this)，SetMaterialDirty触发CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this)。

**当RectTransform/Color/Material/Parent/Animation改变时会触发重建。**

Rebuild主要是几何和材质重建。

##### [UpdateGeometry]

UI组件宽高非0才构建VertexHelper数据，各个组件重写OnPopulateMesh构建VertexHelper数据，遍历IMeshModifier组件（描边，投影等）修改VertexHelper数据，最终VertexHelper数据填充到workerMesh（static）然后交给canvasRenderer去渲染。

##### [UpdateMaterial]

获取材质后，经过挂载的所有IMaterialModifier组件（做一些材质特效）修改后，和mainTexture一起交给canvasRenderer渲染。


##### [CanvasUpdateRegistry]

CanvasUpdateRegistry管理Layout重建队列和Graphic重建队列

Canvas在渲染前会调用willRenderCanvases，执行PerformUpdate


**PerformUpdate执行流程**

1.清除队列中空对象和已销毁的对象

2.Layout队列中元素根据其父节点个数排序

3.Layout队列中元素（ Prelayout = 0,Layout = 1,PostLayout = 2）逐个Rebuild

4.LayoutComplete

5.ClipperRegistry做Cull裁剪

6.Graphic队列中元素（PreRender = 3,LatePreRender = 4,MaxUpdateValue = 5）逐个Rebuild

7.GraphicUpdateComplete


##### [ClipperRegistry]

ClipperRegistry维护裁剪者队列，IClipper是裁剪者，IClippable是可裁剪对象，ClipperRegistry


##### [MaskableGraphic]


MaskableGraphic在Graphic基础上实现了被裁剪与遮罩


Cpu裁剪：RectMask2D实现PerformClipping方法，先通过Clipping.FindCullAndClipWorldRect计算出一个最小裁剪矩形（持续每帧计算子节点的裁剪区域），然后遍历所有IClippable去做裁剪，最后触发canvasRenderer.cull处理。

Gpu裁剪：Mask是通过创建新的材质来裁剪。


##### [Selectable]

Button/Dropdown/InputField/Scrollbar/Slider/Toggle都是继承Selectable实现，Transition实现状态间切换。


##### [EventSystem]

EventSystem管理所有BaseInputModule，系统支持且激活启用的Module被调用Process，PointerEventData和Raycast的结果传给Module去处理，BaseInputModule通知具体的IEventSystemHandler进行逻辑处理（ExecuteEvents）。

EventData：

BaseEventData实现AbstractEventData

PointerEventData：点击触摸相关数据

AxisEventData:移动方向和量

##### [StandaloneInputModule]

StandaloneInputModule->PointerInputModule->BaseInputModule

ProcessMouseEvent:鼠标左中右键处理ProcessMousePress，ProcessMove，ProcessDrag

ProcessTouchEvents:多点触摸处理ProcessTouchPress，ProcessMove，ProcessDrag

均产生PointerEventData符合条件的交予ExecuteEvents执行事件。

##### [Raycasters]

如下图可可知事件处理中，**性能最大消耗点在BaseRaycaster.Raycast**
![image](/img/profile.png)

如下代码可知即使未勾选raycastTarget也会参与到遍历中，访问频次集中在depth和RectangleContainsScreenPoint以及sort排序。


##### [EventSystemHandler]

事件处理者，需要触发哪个事件就实现哪个接口

主要分为三大类：

IPointerXXXXXHandler

IXXXXXDragHandler

IXXXXXHandler

##### [ExecuteEvents]

遍历所有实现的事件接口，执行事件。



