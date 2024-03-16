# BandofFluffy

视频链接：[https://sidequestvr.com/app/26456/animal-dash](https://sidequestvr.com/app/26456/animal-dash)

## 基础架构

## 渲染

### 渲染系统

#### 光照

每一个关卡都只采用一盏动态光（动态光对VR性能影响很大，每多一盏都可以看得出来）和少许几盏静态光作补光所有静态光都自定义了MaxDrawDistance与MaxDistanceFadeRange。

#### 视效

使用Niagara System制作一些简单特效给予玩家游戏状态的提示。

#### 后处理

采用MSAA*4抗锯齿，考虑过FSAA的但是实际测试出来FSAA却差一点，之前有看到UE对VR的MSAA算法有优化，可能是因为这个吧。

#### 基于物理渲染

基于物理的渲染(Physically based rendering)(PBR)意味着表面接近光线在真实世界的表现方式，而不是我们直观以为的应有方式。相较于完全依赖美术师直觉来设置参数的着色工作流程，遵守PBR原则的材质更准确，并且通常看起来更自然。不过由于游戏风格影响仅少部分是基于物理的渲染例如附有青苔的石头等。

#### 大气雾效

你可能会有这种情况：`on mobile the SkyAtmosphere component needs a mesh with a material tagged as lsSky and using the SkyAtmosphere nodes to visualize the Atmosphere.`那么问题很明显，你当前使用的SkyAtmosphere需要做一点改动，[原文](https://forums.unrealengine.com/t/help-completely-lost-with-the-message-on-mobile-skyatmosphere-needs-a-mesh-tagged-issky/252845/11)：Go to the outliner and select your SkyAtmosphere. In the details, choose “SkyAtmosphere (Instance)”, click “Add”, and add a Static Mesh. Click your new mesh, go to the Static Mesh property, and change it for SM_SkySphere. Click your new mesh, go to the Materials property, and where it says Element_0, change in the dropdown for SkySphereMaterial. In your content browser, go to All/Engine/Content/EditorMaterials/Thumbnails and open SkySphere Material.

In SkySphereMaterial, follow the comment

1. In Details->Material->Blend Mode, Opaque不用改
2. In Details->Material->Shading Model, default Lit -> Unlit
3. In Details->Material (Advanced)->Is Sky, tick the box to mark as True

但是，其实直接用BP_Sky_Sphere就行，不用作上面操作，这个警告就会消失

另外由于性能问题将大气雾效的参数拉的很低，然后在远处放置一个倾斜的大面片做一个假的云雾特效。

### 优化

比较详细的在另一篇文章：[https://windcrazy123.github.io/2021/04/18/%E4%BC%98%E5%8C%96](https://windcrazy123.github.io/2021/04/18/%E4%BC%98%E5%8C%96)

## 动画

### 角色动画

#### 事件图表

初始化角色变量和物理模拟组件变量之后有用，每帧更新玩家运动状态和物理模拟所需要的参数如：开启物理模拟后的四个骨骼location和一个rotation

#### 动画图表

基础走跑跳蹲爬state与物理模拟state相互转换，转换条件是：是否进行物理模拟且是否死亡，死亡时客户端在动画蓝图中同步骨骼位置与旋转，由于设置了物理资产中的物理动画的参数，因此在本地模拟物理的同时还会去跟动画蓝图中的Location和Rotation靠近，之后判断是否恢复，如果是，则动画从当前物理模拟保存的姿势开始播放起身蒙太奇动画

## 物理

### 角色触发Ragdoll逻辑

继承接口，当障碍物造成伤害时会对角色调用事件接口，过高跌落会触发重伤逻辑接口；之后根据不同程度伤害触发不同声音并改变角色的最大移动速度，调用不同的布娃娃响应事件

![img](.\Img\Phy\1.png)

![img](.\Img\Phy\2.png)

#### 布娃娃轻中伤害

调用bp_ragdoll_component组件中的接口，并设置physical animation偏向动画的权重，同时设置定时器计时结束则从布娃娃状态中恢复

![img](.\Img\Phy\3.png)

![img](.\Img\Phy\4.png)

![img](.\Img\Phy\5.png)

![img](.\Img\Phy\6.png)

![img](.\Img\Phy\7.png)

#### 重伤\死亡

向服务器发送请求更改physicalanimation profile配置，调用bp_ragdoll_component组件中的接口开始执行布娃娃，并在一定时间后组播起身布尔值使之可以向服务器发送起身命令

![img](.\Img\Phy\8.png)

在重伤时同步布娃娃中胶囊体的位置

![img](.\Img\Phy\9.png)

服务器和客户端都模拟物理，但数据传输是每秒五次进行骨骼位置的复制

### 物理模拟组件

>参考：[Mutiplayer Ragdoll](https://www.unrealengine.com/marketplace/profile/Master+Dinochan)

组件初始化：

![](\Img\Phy\10.png)

Notify初始化CharacterMesh(Replicated)

公开暴露方法：

![](\Img\Phy\11.png)

一个是开始物理模拟另一个是结束物理模拟

开始物理模拟：保存当前速度和胶囊体位置停止蒙太奇动画使胶囊体无碰撞设置MovementMode==NONE，所有客户端和主机开始模拟物理，使用之前保留的速度设为当前物理速度作为抛射速度。轻重伤和主机与客户端的逻辑有区别，具体根据项目配置改变即可

tick：更新胶囊体位置和Character的Rotation与Location还有骨骼的四个Location一个Rotation（复制）轻重伤和主机与客户端的逻辑有区别，具体根据项目配置改变即可

退出物理模拟：从当前物理模拟保存的姿势开始播放起身蒙太奇动画，恢复之前改变的参数，轻重伤和主机与客户端的逻辑有区别，具体根据项目配置改变即可

> 较详细的有：[物理模拟](https://windcrazy123.github.io/2021/04/18/%E7%89%A9%E7%90%86%E6%A8%A1%E6%8B%9F/)
>
> ​					  [杂记-物理](https://windcrazy123.github.io/2021/04/18/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E6%9D%82/#%E7%89%A9%E7%90%86)

## 音效

## GamePlay

## 工具链

## 网络

## 参考

[剖析虚幻渲染体系-XR专题](https://www.cnblogs.com/timlly/p/16357850.html#152-xr%E6%8A%80%E6%9C%AF)
