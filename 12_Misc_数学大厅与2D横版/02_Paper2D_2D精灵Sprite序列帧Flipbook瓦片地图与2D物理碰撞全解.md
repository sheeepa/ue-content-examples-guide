# 02 · `Paper2D` 全解：2D 精灵 Sprite、序列帧 Flipbook、瓦片地图与受限平面物理碰撞

> **对应关卡路径**：`/Game/Maps/Misc/Paper2D`
> **核心资产目录**：`/Game/ExampleContent/Paper2D/`（联动引擎内置插件 `/Paper2D/`）
> **关卡实测规模**：总 Actor 数 `102` | 官方展台（`BP_DemoDisplay`）总数 **`7` 座** | 演示示例 Actor 数 `46` | 关卡蓝图动态时间轴 `Timeline_0`

---

## 序章：为什么要用虚幻引擎做 2D 与 HD-2D 游戏？

很多人提起虚幻引擎（Unreal Engine），第一印象都是写实 3A 巨制；但近年来以《八方旅人（Octopath Traveler）》系列为代表的 **HD-2D（次世代三维光影 + 复古 2D 像素精灵）** 风潮席卷全球，越来越多独立团队与中型工作室发现：**把 2D 精灵放进 UE5 的世界里，天然就能白拿 Lumen 实时全局光照、体积雾（Exponential Height Fog）、电影级景深（CineCamera Tilt-Shift 移轴微缩景观感）以及完整的材质节点与 Chaos 物理生态！**

承载这一切的官方核心基座，正是引擎内置的 **`Paper2D` 插件**。打开 `/Game/Maps/Misc/Paper2D`，你会发现连这座展厅的建筑结构都别出心裁——在 `BP_DemoRoom` 配置中，Epic 特意将 `roof`（屋顶）和 `enclosed Right`（右侧墙体）设为 `false`，让整条长廊沐浴在自然天光与可移动太阳光之下：

![Paper2D 展厅总览：敞开式自然采光长廊与 7 座核心展台实拍](../images/maps/Paper2D_00_overview.png)

在这条沿 $X$ 轴从 `495` 延伸至 `6,495` 的展廊中，7 座展台浓缩了从**单张贴图切分、渲染网格减面降 Overdraw、自定义法线受光材质、序列帧非均匀帧率节奏、蓝图状态机控制、受限平面 3D 物理弹珠台到复古像素无损采样**的完整工业流水线。下面让我们逐台拆解全部实测资产，并延伸讲透 **TileMap 瓦片地图**与 **HD-2D 实战架构**！

---

## 一、 展厅全局架构与第 1 座展台：插件基座与光照环境

### 展台 #1：`Paper 2D`（插件总览与动态天光舞台）
- **展牌标识与坐标**：`BP_DemoDisplay7` | 坐标 `(495, -900, 0)`
- **官方展牌原文**：`Examples using Paper2D Plugin`
- **插件启用检查**：`Paper2D` 作为官方内置插件默认已在大多数模板开启；若在新工程未见相关右键菜单，前往 `Edit -> Plugins` 搜索 **`Paper2D`** 勾选启用即可（底层模块路径为 `/Script/Paper2D`）。
- **展台关联环境与关卡脚本实测拆解**：
  1. **全动态三维光照套件（共 5 个全局组件 Actor）**：
     - `DirectionalLight`（`mobility = Movable`，亮度 `Intensity = 8.0`，开启阴影 `castShadows = true`）；
     - `SkyLight`（`mobility = Movable`，亮度 `Intensity = 2.0`）；
     - `SkyAtmosphere` + `ExponentialHeightFog`（营造通透空气透视）；
     - 挂载 `ChaosDebugDrawComponent_0` 的物理调试器。
     为什么要给一个 2D 展厅配齐整套动态太阳光与天光？紧接着在 **`#2` 号展台右侧的那枚发光法线圆章**上，你立刻就能看到 2D 精灵与 3D 动态光照碰撞出的火花！
  2. **关卡蓝图 `/Game/Maps/Misc/Paper2D.Paper2D_C`**：内置了一条核心时间轴组件 **`Timeline_0`**，用于在运行时周期性触发 `#6` 号物理展台的底部径向力场脉冲（`RadialForceActor`），让满屏 2D 硬币周而复始地沸腾弹跳。

---

## 二、 2D 渲染的原子单元：`UPaperSprite`、受光法线材质与几何裁剪优化（展台 `#2`～`#3`）

### 展台 #2：`Sprites`（2D 精灵本质、像素比例尺与受光顶点动画材质）
- **展牌标识与坐标**：`BP_DemoDisplay1` | 坐标 `(1515, -900, 0)`
- **展台 3D 悬浮标注**：左侧 `'Sprite' / '512x512 Texture' / '5 Pixel per Unreal Unit' / 'Default Material'`；右侧 `'modified properties' / 'Sprite Color = Red' / 'Custom Material'`

![展台 #2 Sprites：默认无光照精灵 vs 叠加红色顶点色与法线高光的自定义材质精灵](../images/maps/Paper2D_01_stand_02.png)

#### 1. 什么是 Paper2D 中的 `Sprite`？四种标准创建姿势
在虚幻引擎底层，`UPaperSprite` 并不是古老的屏幕空间 HUD 贴片，而是**一块完全生活在三维世界坐标系中的平面多边形网格（Planar Mesh）+ 关联材质（Material）+ 可选三维挤压碰撞体（`UBodySetup`）**！
在内容浏览器（Content Browser）中共有 **4 种创建途径**：
1. **单图直转**：右键单张贴图 $\to$ `Sprite Actions -> Create Sprite`；
2. **图集智能切分（最常用！）**：右键整张序列帧大图（Sprite Sheet）$\to$ `Sprite Actions -> Extract Sprites`！对话框支持两种模式：
   - **`Auto`（自动透明孤岛检测）**：自动沿着透明通道（Alpha = 0）轮廓框出每一帧独立小图并批量生成 Sprite 资产；
   - **`Grid`（等距网格切分）**：按指定像素宽高（`Cell Width / Height`）与边距（`Margin / Spacing`）整齐切块。
3. **空白新建**：点击 `Add / New -> Miscellaneous -> Sprite`，再手动指定源贴图；
4. **专业工具链导入**：点击 `Import` 直接导入 **TexturePacker** 或 **Aseprite** 导出的 **`*.json`（`.paper2dsprites`）图集描述文件**——引擎会自动导入贴图、切好所有子 Sprite 的源区域与旋转裁剪框，甚至一键连 Flipbook 序列帧都帮你拼好！

#### 2. 核心参数深度拆解：`Pixels Per Unreal Unit`（像素/厘米换算公式）
撬开左侧演员 `PaperSprite` 引用的资产 `/Game/ExampleContent/Paper2D/Sprites/PaperSprite`：
- **源贴图尺寸**：`sourceTexture = /Game/Global/Textures/UE_LOGO_CARD`（分辨率 `512 x 512` 像素）；
- **像素密度比例尺**：**`pixelsPerUnrealUnit = 5.0`**（简称 **PPU**）；
- **轴心点模式**：`pivotMode = Center_Center`（`customPivotPoint = (256, 256)`）；
- **默认材质与备用材质**：
  - `defaultMaterial = /Paper2D/MaskedUnlitSpriteMaterial`（默认遮罩无光照材质，镂空圆章外的透明像素）；
  - `alternateMaterial = /Paper2D/OpaqueUnlitSpriteMaterial`（用于双段式几何渲染的不透明材质，详见展台 `#3`）。

> [!TIP]
> **世界尺寸心算公式（1 UU = 1 cm）**：
> $$\text{WorldSize (cm)} = \frac{\text{TextureDimension (px)}}{\text{PixelsPerUnrealUnit (px/cm)}}$$
> 本例中 $512 \div 5 = \mathbf{102.4\text{ cm}}$（约 1 米高的圆牌）！在立项一款横版或 HD-2D 游戏的第一天，美术与程序**必须先锁死全项目统一的 `Pixels Per Unreal Unit`**（例如角色高 64 像素，希望对应 UE 标准人类胶囊体高度约 160 cm，则全项目统一设 `PPU = 64 / 160 = 0.4`），否则场景道具与角色拼在一起会出现灾难性的像素颗粒大小不一！

#### 3. 19 节点自定义受光 Sprite 材质全拆解（`CustomSpriteMaterial`）
再看展台右侧的 `PaperSprite_3`：它引用的明明是**同一个 `PaperSprite` 资产**，仅仅在组件面板上改了两处——把 `Sprite Color` 设为红色，并把材质重载为 `/Game/ExampleContent/Paper2D/Materials/CustomSpriteMaterial`，结果它不仅受场景动态光照明暗折射、表面浮现出立体凹凸浮雕，还在上下悬浮呼吸并周期闪烁红光！
打开 `CustomSpriteMaterial` 的 **19 个节点与核心属性**，揭开自定义 Paper2D 着色器的三大铁律：

1. **铁律一：必须声明名为 `"SpriteTexture"` 的 2D 纹理参数！**
   材质中使用了 `TextureSampleParameter2D`（`parameterName = "SpriteTexture"`，默认占位图指向 `/Paper2D/PlaceholderTextures/DummySpriteTexture`）。当材质赋给 `PaperSpriteComponent` 或 `PaperFlipbookComponent` 时，C++ 渲染代理会自动把当前帧 Sprite 的真实源贴图覆盖塞进 `"SpriteTexture"` 槽位！
2. **铁律二：组件面板的 `Sprite Color` 是通过 `VertexColor`（顶点色）传进着色器的！**
   为什么在组件上调 `Sprite Color = Red` 会生效？因为 Paper2D 在组装平面网格顶点缓冲时，直接把 `SpriteColor (R,G,B,A)` 写进了每个顶点的顶点色通道！在 `CustomSpriteMaterial` 中：
   - **闪烁呼吸自发光链路**：`Time * Constant(0.5, "Blinking speed") -> Sine -> Clamp -> Multiply(Constant 50, "Glow") -> Multiply(VertexColor) -> Multiply(SpriteTexture.A)` 连入 `MP_EmissiveColor`——正因乘上了 `VertexColor`，组件的红色属性直接把脉冲辉光染成了烈焰红！
3. **铁律三：开启双面受光 + 法线贴图 + WPO 顶点悬浮**：
   - 材质属性设为 **`MSM_DefaultLit`（默认受光）**、**`BLEND_Masked`**、**`twoSided = true`**（保证横版角色左右翻转缩放 `Scale.X = -1` 时背面不黑且不剔除），粗糙度 `Roughness = 0.5`；
   - **2D 法线贴图**：采样 `/Game/Global/Textures/UEMan_Logo_N` 连入 **`MP_Normal`**，让扁平纸片在头顶 `DirectionalLight` 照射下呈现出真实的金属徽章凹凸高光！（注：如果希望每个不同 Sprite 自动切换自己的专属法线贴图，只需在 Sprite 资产的 `Additional Source Textures` 数组第 0 槽填入法线图，并在材质中采样名为 **`"SpriteTexture_0"`** 的纹理参数即可自动配对！）
   - **GPU 顶点上下浮动**：`Time -> Sine -> Multiply(Constant3Vector(0, 0, 20))` 连入 **`MP_WorldPositionOffset`**，让圆章在 Z 轴上下优雅浮动 $\pm 20\text{ cm}$！

---

### 展台 #3：`Sprite Editor`（源区域裁切、多边形渲染减 Overdraw、九宫格与碰撞编辑）
- **展牌标识与坐标**：`BP_DemoDisplay2` | 坐标 `(2500, -900, 0)`
- **快捷打开方式**：在内容浏览器双击任意 Sprite 资产，或在关卡视口选中已摆放的 `PaperSpriteActor` 直接按下快捷键 **`Ctrl + E`**！
- **实测 7 个陈列 Actor 与 4 份对比资产逐一拆解**：

本展台从左到右分成三大实战教学区，直指 2D 游戏性能优化与物理交互命门：

#### 1. 左区：`Edit Source Region`（单张大图集裁切多精灵）
- **陈列资产**：`PaperSprite_2`（引用 `/Game/ExampleContent/Paper2D/Sprites/PaperSprite2`）
- **实测数据**：源贴图是包含完整横幅标志的 `/Game/ExampleContent/Paper2D/Textures/T_UnrealLogo`，通过 `Edit Source Region` 框选子区域 `sourceDimension = (2033, 288)`（`pixelsPerUnrealUnit = 15`，`pivotMode = Bottom_Left`）。
- **核心价值**：将几十个 UI 图标、路面碎石、草丛装饰合并打包在一张 `2048x2048` 纹理图集（Texture Atlas）中，让不同 Sprite 各自框选不同的 `Source Region`——在渲染时引擎可以对共用同一张贴图与材质的大批 Sprite 进行批量合批绘制，大幅削减 GPU 纹理切换与 DrawCall！

#### 2. 中区：`Edit Render Geometry`（紧贴轮廓裁剪多边形，消灭透明像素 Overdraw！）
- **陈列对比组**（上下两排直观透视！）：
  - **左列（默认包围盒 `default`）**：上排正常渲染 `PaperSprite3`，下排挂载橙色线框材质 `M_Opaque_WireFrame_Unlit` 的同款副本 **`PaperSprite5`**。实测 `PaperSprite3` 的 `renderGeometry.geometryType = "TightBoundingBox"`，仅是一个 `283 x 408` 的 **4 顶点矩形方框**——圆形虚幻标志四角大片透明空白区域依然会被光栅化并执行像素着色器 `clip()` 丢弃，造成严重的 **Quad Overdraw（过度绘制浪费）** 与 Masked 阴影边缘浪费！
  - **右列（手工/自动轮廓贴合 `edited`）**：上排正常渲染 `PaperSprite4`，下排挂载橙色线框材质的同款副本 **`PaperSprite6`**！实测 `PaperSprite4` 的 `renderGeometry.geometryType = "FullyCustom"`，由 **16 个紧贴圆章边缘的多边形顶点**（`(437,326) -> (699,164.5) -> (1024,63) -> ...`）包络而成！原本四角无用的透明三角面被彻底切除！

| 渲染几何体模式 (`ESpritePolygonMode`) | 几何生成策略 | 顶点开销 vs 像素填充率 (Fillrate) 权衡 | 最佳适用对象 |
| :--- | :--- | :--- | :--- |
| **`Source Bounding Box`** | 完整源区域矩形（4 顶点，2 三角面） | 顶点数最少，透明浪费最大 | 填满整格的方形地砖、不透明背景墙 |
| **`Tight Bounding Box`（默认）** | 自动收缩至非透明像素外接矩形（4 顶点） | 顶点数极少，适合近似方形物体 | 箱子、门窗、紧凑像素小道具 |
| **`Shrink Wrapped`（自动收缩包裹）** | 按 `AlphaThreshold` 与 `SimplifyEpsilon` 自动追踪透明轮廓生成凸/凹多边形 | 顶点数略增（8～24 顶点），**大幅砍掉透明像素 Overdraw！** | 枝叶繁茂的大树、大型异形特效贴片、圆盾 |
| **`Fully Custom`（纯手绘顶点）** | 美术在编辑器内手动增删拖拽多边形顶点 | 极致精准可控 | 主角核心贴片、高频满屏复用的前景遮挡物 |
| **`Diced`（网格切块 + 双材质分离）** | 按 `32x32` 细分切格：纯实心格自动走 **`Opaque` 不透明通道**，仅半透边缘格走 `Masked`！ | 消除大面积实心精灵的 Early-Z 失效痛点！ | 占据半个屏幕的巨型 Boss 或近景山体贴片 |

> [!NOTE]
> **补充利器：Nine-Slice（九宫格无损拉伸）与 Sockets（骨骼级插槽挂点）**
> 在 Sprite Editor 工具栏中还有两项高频工程能力：
> - **九宫格缩放**：将 `Render Geometry Type` 设为 `Source Bounding Box` 后即可开启九宫格切线，拉伸对话框或横版木桥长度时，四个圆角保持 1:1 像素不拉伸变形，仅平铺拉伸中间段；
> - **Sockets（插槽）**：直接在角色开枪帧的枪口像素处新建一个名为 `MuzzleSocket` 的插槽，蓝图中调 `PaperFlipbookComponent -> GetSocketTransform("MuzzleSocket")` 就能精准生成子弹与枪口火光！

#### 3. 右区：`Edit Collision`（从 2D 轮廓自动挤压生成 3D 物理凸包！）
- **陈列对比组**：左侧默认方盒碰撞 `PaperSprite7` vs 右侧精细 33 顶点圆形多边形碰撞 **`PaperSpriteCollision`**！
- **实测资产数据**（`/Game/ExampleContent/Paper2D/Sprites/PaperSpriteCollision`）：
  - `collisionGeometry.geometryType = "FullyCustom"`，`alphaThreshold = 0.1`，`simplifyEpsilon = 8`，沿着圆形边缘生成了 **33 个多边形顶点**；
  - **`collisionThickness = 5.0`（碰撞挤压厚度，默认 `10.0`）**！
  - **视口验证指令**：如展台地面文字提示，在编辑器视口按下快捷键 **`Alt + C`**，或在 PIE 运行时按 `~` 输入控制台命令 **`show collision`**，你会看到这枚扁平的 2D 圆牌其实沿着垂直于纸面的 **Y 轴前后挤压出了厚度为 5 cm 的三维多面体棱柱（`UBodySetup`）**——这正是它能直接无缝接入 UE5 Chaos 三维物理引擎与射线检测的秘密！

---

## 三、 赋予 2D 角色灵魂：序列帧 `UPaperFlipbook` 与蓝图状态机控制（展台 `#4`～`#5`）

### 展台 #4：`Flipbooks`（关键帧序列动画与 `Frame Run` 节奏打击感秘诀）
- **展牌标识与坐标**：`BP_DemoDisplay3` | 坐标 `(3520, -900, 0)`
- **展台 3D 悬浮标注**：左侧 `'8 frame animation / extracted from one texture'`；右侧 `'same animation with different "Frame Run"'`

![展台 #4 Flipbooks：Epic 传奇吉祥物 Jazz Jackrabbit 8 帧奔跑序列帧与 Frame Run 节奏对比实拍](../images/maps/Paper2D_02_stand_04.png)

#### 1. `UPaperFlipbook` 数据结构本质
如上方实拍图所示，展台上奔跑的绿色像素兔子正是 Epic Games 1994 年横版射击名作《光速兔崽子（Jazz Jackrabbit）》的主角！
在 Paper2D 中，一个 **`PaperFlipbook`（翻页书序列帧资产）** 本质上是一个结构体数组 `KeyFrames` 加上全局基准帧率 `FramesPerSecond`：
- 创建方式：`Add / New -> Animation -> Sprite Flipbook`，或导入 `*.JSON` 图集自动生成；
- 每个关键帧槽位 `FPaperFlipbookKeyFrame` 仅包含两个字段：
  1. **`Sprite`**：指向这一拍要显示的 `UPaperSprite` 资产；
  2. **`FrameRun`（持续拍数，整数 $\ge 1$）**：这一张画面需要连续停留几个基准帧单位！

#### 2. 实测资产对比拆解：`RunAnim` vs `RunAnim2`（零额外内存做出手绘动画“一拍二”顿挫张力！）
让我们直接把展台左右两只兔子的底层序列化 JSON 拉出来做像素级对比：

| 属性维度 | 左侧兔子：`RunAnim` (`/Game/ExampleContent/Paper2D/RunAnim/RunAnim`) | 右侧兔子：`RunAnim2` (`/Game/ExampleContent/Paper2D/RunAnim/RunAnim2`) |
| :--- | :--- | :--- |
| **引用的 8 张子精灵顺序** | `Sprite4 -> Sprite5 -> Sprite -> Sprite2 -> Sprite6 -> Sprite3 -> Sprite7 -> Sprite8` | **完全相同的 8 张子精灵资产**（零额外贴图显存开销！） |
| **各帧 `FrameRun` 停留拍数** | `[1, 1, 1, 1, 1, 1, 1, 1]`（匀速一拍一，总计 **8 拍**） | `[1, 2, 1, 1, 1, 2, 1, 1]`（第 2 帧 `Sprite5` 与第 6 帧 `Sprite3` 设为 **`2` 拍**，总计 **10 拍**！） |
| **播放帧率 (`framesPerSecond`)** | `10.0 FPS` | `13.89263 FPS`（适度提速补偿多出的 2 拍停留） |
| **单圈循环总时长** | $8 \div 10.0 = \mathbf{0.80\text{ 秒}}$ | $10 \div 13.89263 \approx \mathbf{0.72\text{ 秒}}$ |
| **碰撞源 (`collisionSource`)** | `NoCollision`（横版角色通常由外层 `CapsuleComponent` 承担移动碰撞） | `NoCollision` |

> [!TIP]
> **传统二维原画大师的“时间切片”心法（Timing & Spacing）**：
> 为什么右边的 `RunAnim2` 只是把第 2 张（左脚蹬地极限张）和第 6 张（右脚跨步极限张）的 `FrameRun` 从 `1` 改成了 `2`，跑起来就更有弹性力量感？
> 在传统赛璐璐与像素动作设计中，匀速等间隔播放（`[1,1,1,1...]`）往往显得像发条玩具一样平淡机械；而在**发力蓄力帧（Anticipation）、挥砍命中帧（Impact）或跨步最高点（Extreme Pose）故意拉长 `FrameRun = 2 ~ 4` 拍，中间过渡帧保持 `FrameRun = 1` 拍快速掠过**，无需原画师多画哪怕一张新图，就能瞬间塑造出拳拳到肉的顿帧感与爆发节奏！

---

### 展台 #5：`Control via Blueprints`（蓝图交互控制与动画状态切换）
- **展牌标识与坐标**：`BP_DemoDisplay6` | 坐标 `(4505, -900, 0)`
- **实测 11 个空间交互 Actor 完整配对拆解**：
  展台正中央 `(4500, -575, 20)` 站立着核心交互蓝图实例 **`BP_Flipbook`**（`/Game/ExampleContent/Paper2D/Blueprints/BP_Flipbook`），其内部挂载了 `UPaperFlipbookComponent`（名为 `PaperFlipbook1`），默认播放包含 **26 个关键帧、帧率 `8 FPS`** 的呼吸眨眼待机动画 `/Game/ExampleContent/Paper2D/IdleAnim/IdleAnim`，并引用了行走动画 `/Game/ExampleContent/Paper2D/WalkAnim/WalkAnim`。
  在角色正前方的走廊地板上（`Y = -340`），从左到右一字排开 **5 个触发盒 `TriggerBox` 与 5 座三维悬浮立体图标**——当你操纵角色踩上不同踏板时，关卡/Actor 蓝图立即调用 `PaperFlipbookComponent` 的对应控制接口：

| 踏板从左至右位置 | 触发器 Actor | 悬浮三维图标网格 | 触发调用的 `UPaperFlipbookComponent` 核心函数 | 交互表现与 Gameplay 用途 |
| :---: | :---: | :--- | :--- | :--- |
| **1. `(4330, -340)`** | `TriggerBox3` | `SM_Play_2`（左指三角箭头 `SM_Play`） | **`Reverse()`** 或 `ReverseFromEnd()` | **倒放序列帧**（`PlayRate < 0`）：常用于时光倒流机制、机关门缩回、角色后撤步动画复用。 |
| **2. `(4420, -340)`** | `TriggerBox2` | `SM_Play_4`（方块图标 `SM_Stop`） | **`Stop()`** | **暂停冻结在当前帧**：常用于受击硬直顿帧（HitStop / Freeze Frame）或时间静止技能！ |
| **3. `(4515, -340)`** | `TriggerBox1` | `SM_Play`（右指三角箭头 `SM_Play`） | **`Play()`** 或 `PlayFromStart()` | **正向恢复播放**。 |
| **4. `(4610, -340)`** | `TriggerBox4` | `SM_Play_5`（环形箭头 `SM_Loop`，绿材质 `M_Arrows_Green`） | **`SetLooping(bNewLooping)`** + 切换图标红/绿材质 | **切换循环/单次模式**：待机/跑步必须开启 `Looping = true`；而**挥剑攻击、跳跃起跳、死亡倒地**必须设为 `Looping = false` 并绑定 `OnFinishedPlaying` 委托回调！ |
| **5. `(4700, -340)`** | `TriggerBox5` | `SM_Play_3`（双向切换箭头 `SM_Direction`） | **`SetFlipbook(NewFlipbook)`**（在 `IdleAnim` 与 `WalkAnim` 间轮换） | **热切换 Flipbook 资产**：这正是 2D 横版角色动画状态机（Idle $\leftrightarrow$ Run $\leftrightarrow$ Jump $\leftrightarrow$ Attack）的核心驱动函数！ |

#### 2D 横版角色 `APaperCharacter` 标准动画状态机蓝图范式
在实际开发 2D 横版动作游戏时，引擎提供了直接继承自 `ACharacter` 的 **`APaperCharacter`** 类（自带 `CapsuleComponent` + 完整的 `CharacterMovementComponent` 跳跃/重力/斜坡行走 + `PaperFlipbookComponent` 名为 `Sprite`）。其每帧状态同步的标准蓝图/C++ 范式只需三步：
```
[Event Tick / Movement Updated]
   ├── 1. 水平朝向翻转：读取 GetVelocity().X
   │       • 若 Velocity.X > +0.1 ──▶ SetControlRotation(0, 0, 0)    // 面向 +X 右方
   │       • 若 Velocity.X < -0.1 ──▶ SetControlRotation(0, 0, 180)  // 绕 Z 轴转 180° 面向 -X 左方（优于负缩放 Scale.X=-1，不会翻转绑定的子挂件坐标系！）
   ├── 2. 判定运动状态选择目标 Flipbook：
   │       • CharacterMovement.IsFalling() == true  ──▶ Select(Velocity.Z > 0 ? FB_JumpRise : FB_JumpFall)
   │       • CharacterMovement.IsFalling() == false ──▶ Select(VSizeXY(Velocity) > 5.0 ? FB_Run : FB_Idle)
   └── 3. 调用 Sprite -> SetFlipbook(SelectedFlipbook)（注：SetFlipbook 内部自带幂等检查，若传入与当前相同的 Flipbook 绝不会重置播放进度！）
```

---

## 四、 受限平面 3D 物理弹珠台与复古像素无损管线（展台 `#6`～`#7`）

### 展台 #6：`Physics`（为什么用“锁轴 3D Chaos 物理”统治 2D 游戏碰撞？）
- **展牌标识与坐标**：`BP_DemoDisplay4` | 坐标 `(5510, -900, 0)`
- **展台 3D 悬浮标注**：`'These sprites are using 3D physics but the Locked Axis is set to Y / this limits their movement to a plane'`

![展台 #6 Physics：11 枚圆形多边形碰撞精灵在 Y 轴锁定平面内被底部双径向力场抛起碰撞实拍](../images/maps/Paper2D_03_stand_06.png)

#### 1. 历史演进与架构真相：告别实验性 Box2D，拥抱“平面受限 3D Chaos 物理”！
早期 UE4 曾短暂实验过集成第三方 2D 物理库 Box2D，但很快发现纯 2D 物理世界与 UE 强大的三维射线检测（LineTrace）、三维导航网格（NavMesh）、三维触发器（TriggerBox）、三维角色移动组件（CharacterMovement）以及破坏力场完全割裂，两边坐标与事件无法互通！
因此，UE 官方确立了延续至今的黄金正统方案——**直接使用引擎原生 3D 物理引擎（UE5 Chaos），配合刚体平面自由度约束（Planar Constraints / Locked Axis）模拟纯正 2D 物理！**

#### 2. 实测 16 个物理舞台 Actor 精密解剖
对照上方实拍图（可见两侧黄色体积框线与底角两个声波状力场图标），看看这座全自动“2D 硬币爆米花机”是怎么搭出来的：
1. **11 枚动态刚体硬币（`PaperSpriteCollision_3`～`PaperSpriteCollision_14`）**：
   - 全部整齐排列在同一张垂直平面 **`Y = -610`** 上（初始高度 $Z$ 分布在 `150`～`500` 之间）；
   - 引用我们在 `#3` 拆解过的 **`PaperSpriteCollision`**（33 顶点精细圆轮廓 + `collisionThickness = 5` cm 的三维薄圆柱凸包 `BodySetup`）；
   - 开启 `Simulate Physics = true`，并将物理约束设为 **`Lock Axis = Y`（等价于勾选 `Lock Position Y` + `Lock Rotation X` + `Lock Rotation Z`）**！
   - **自由度数学分析**：三维刚体原本有 **6 个自由度（6-DOF：平移 X/Y/Z + 旋转 Roll/Pitch/Yaw）**。锁死 `Y` 轴位置以及绕 `X`、`Z` 轴的侧翻自转后，硬币只剩下 **平面内的上下左右移动（Translation X, Z）** 和 **顺逆时针平面滚动（Pitch 绕 Y 轴旋转）** 这 **3 个标准 2D 刚体自由度**！无论硬币之间撞得多猛烈，它们永远乖乖贴在 `Y = -610` 平面上，绝不会歪斜弹出纸面！
2. **三面隐形防逃逸围栏（`BlockingVolume1`、`BlockingVolume2`、`BlockingVolume3`）**：
   - 左挡墙 `BlockingVolume1` 位于 `X = 5220`，右挡墙 `BlockingVolume2` 位于 `X = 5780`，天花板封顶 `BlockingVolume3` 位于 `Z = 580`，与展台地板共同围成密闭矩形腔体。
3. **底部双涡轮脉冲发射器（`RadialForceActor1` & `RadialForceActor2`）**：
   - 分别安放在左下角 `(5275, -611, 30)` 与右下角 `(5725, -611, 30)`（均精确共面于 `Y = -611`！），由关卡蓝图 `Paper2D_C` 的 `Timeline_0` 周期性喷发径向冲击力，把落到底部的硬币不断炸向空中互相弹撞翻滚！

---

### 展台 #7：`Classic Pixel Sprites`（拯救糊成马赛克的像素画：复古像素三大黄金铁律！）
- **展牌标识与坐标**：`BP_DemoDisplay5` | 坐标 `(6495, -900, 0)`
- **展台 3D 悬浮标注**：左侧 `'Texture imported with default settings'`；右侧 `'Texture configured for classic pixel look' / 'NoMipmaps' / 'Filter: Nearest' / 'Compression Settings: Editor Icon'`

#### 1. 为什么美术画得无比锐利的 `61 x 35` 像素小人，直接拖进 UE 会糊成一团脏泥？
现代 3D 引擎默认把所有导入的 PNG 当成写实 3D 模型的漫反射贴图，默认会做三件对像素画“毁天灭地”的事：
1. **双线性/三线性插值过滤（`Filter = Default / Bi-linear`）**：放大显示时在相邻 $2\times 2$ 纹素之间做 `Lerp` 渐变混合，导致硬朗的 1 像素黑描边全部糊成半透明毛边；
2. **DXT1 / BC1 / BC3 有损块压缩（`Compression = Default`）**：把每 $4\times 4$ 像素强行压缩成两个 16 位 RGB565 端点色插值——当一个 $4\times 4$ 像素块里既有红头巾、绿皮肤又有蓝枪管时，颜色直接串色发脏！
3. **Mipmap 多级渐远纹理降采样**：镜头稍微拉远或斜视时，GPU 自动切换到半分辨率模糊 Mip 层级。

#### 2. 一键封神菜单：`Sprite Actions -> Configure For Retro Sprites`
对比展台左侧模糊发虚的 `T_Jazzrun_D_Sprite_Default`（源贴图 `T_Jazzrun_Default_D`）与右侧刀锋般锐利清脆的 `T_Jazzrun_D_Sprite`（源贴图 `T_Jazzrun_D`，两者均为 `61 x 35` 像素，放大比例 `pixelsPerUnrealUnit = 0.35`，即在世界里高 $35 \div 0.35 = 100\text{ cm}$）：
只需在内容浏览器右键像素贴图选择 **`Sprite Actions -> Configure For Retro Sprites`**，引擎会自动为你改好展台标注的三项黄金配置：
- **`Mip Gen Settings = NoMipmaps`**：彻底关闭 Mipmap 降采样链，远近永远读取 100% 原始像素；
- **`Filter = Nearest`（邻近点采样）**：关闭纹素间线性插值，每个纹素放大后都是边缘如刀切般锐利的正方形像素块；
- **`Compression Settings = UserInterface2D (RGBA)`（旧版显示名即展牌所写 `EditorIcon`）**：启用无损未压缩 BGRA8 格式，彻底杜绝 $4\times 4$ DXT 色块串色污染！（注：若项目纹理总量极大，也可配合调色板索引材质或 ASTC 高质量档位，但独立像素游戏首选 `UserInterface2D`）。
- **额外细节——脚底轴心对齐**：注意实测数据中 `T_Jazzrun_D_Sprite` 将 `pivotMode` 设为了 `Custom` `customPivotPoint = (171.5, 38)`（紧贴角色双脚踩地线），保证切换不同高度的动作帧时角色脚底永远稳稳踩在地面上不悬空抖动！

---

## 五、 工程进阶扩展：`TileSet` / `TileMap` 瓦片地图与次世代 HD-2D 管线实战

受限于早期展厅篇幅，`Paper2D` 地图未单独陈列瓦片地图展台，但在真实的 2D 横版银河恶魔城（Metroidvania）或俯视角 RPG 项目中，**瓦片地图（TileMap）** 与 **HD-2D 光影合成** 是不可或缺的左膀右臂。以下为你补齐整套生产级实战架构：

### 5.1 `UPaperTileSet`（瓦片集）与 `UPaperTileMap`（多层瓦片地图）全工作流

1. **创建与配置瓦片集（`Right Click Texture -> Sprite Actions -> Create Tile Set`）**：
   - 设定单块瓦片像素尺寸 **`Tile Size`**（如 `16x16` 或 `32x32`），若图集带防溢色留白（Padding）则同步填入 `Border Margin` 与 `Per-Tile Spacing`（**强烈建议给 TileSet 图集外围加 1~2 像素 Extrude 边缘延展**，防止镜头非整数坐标移动时采样到隔壁瓦片的杂色缝隙！）；
   - **直接在瓦片集里画地形碰撞**：在 TileSet 编辑器中点击 `Colliding Tiles` 过滤视图，一次性为“砖墙瓦片”画好矩形盒，为“$45^\circ$ 斜坡瓦片”画好直角三角形多边形！
2. **用 `PaperTileMap` 刷出多层视差 2D 关卡（`Add -> Miscellaneous -> Tile Map`）**：
   - 支持类似 Photoshop 的**多图层栈（Tile Layers）**：例如 `Foreground_Deco`（前景遮挡草木）$\to$ `Main_Gameplay_Collision`（主碰撞地面层）$\to$ `Background_Wall`（背景砖墙）；
   - **防 Z-Fighting 图层深度间隔**：调整属性 **`Separation Per Layer`**（默认如 `4.0 cm`），引擎会自动将每一层沿垂直纸面的 Y 轴错开固定物理距离——若配合透视相机（Perspective Camera），前后景图层天然就会产生极其自然的真实三维视差滚动（Parallax Scrolling）！
   - **一键烘焙合并 3D 物理体**：勾选 `Main_Gameplay_Collision` 图层的碰撞开关并设置 `Collision Thickness` 后，即使你刷了上万块地砖，Paper2D 也会在底层将相邻瓦片碰撞合并生成高效的 Chaos 静态刚体，角色站在上面与踩在普通 `StaticMesh` 上毫无二致！
   - **运行时可破坏地形黑科技**：在蓝图中调用 `PaperTileMapComponent -> SetTile(X, Y, Layer, EmptyTile)` 再调 `RebuildCollision()`，寥寥几个节点就能做出《泰拉瑞亚（Terraria）》式的挖矿破坏地形！

---

### 5.2 在 UE5 中搭建《八方旅人》级 HD-2D 渲染与动画管线的四大秘诀

如果你想把 `Paper2D` 的潜力在 UE 5.3～5.8 中发挥到极致，请收好这套经过商业项目验证的现代技术栈组合：

1. **精灵开启真实投影与双面受光材质**：
   - 将 Paper2D 组件的材质替换为插件内置的 **`/Paper2D/MaskedLitSpriteMaterial`**（或像展台 `#2` `CustomSpriteMaterial` 那样自建 `DefaultLit` / `Subsurface` 双面次表面透光材质）；
   - 在 `PaperFlipbookComponent` 细节面板勾选 **`Cast Shadow = true`**（如果是单面纸片在侧光下投影太薄，可进一步勾选 **`Cast Shadow as Two Sided = true`**）——在 UE5 **Lumen + Virtual Shadow Maps (VSM)** 下，像素小人走过篝火旁时，不仅身后会拉出柔和修长的动态实时剪影，火光还会照亮地面并把暖橙色间接反弹光洒在小人的像素衣角上！
2. **电影摄像机移轴景深（Tilt-Shift Diorama Look）**：
   - 不要死守正交相机！尝试给横版/斜俯视角主角挂一台 **`CineCameraActor`**，使用中长焦镜头（如 `Focal Length = 50mm ~ 85mm`）拉远距离并调大光圈（如 `Current Aperture = f/1.4 ~ f/2.8`），手动把对焦平面锁定在主角所在的 `Y` 深度平面上——前景灌木与远景城堡会自动化开奶油般的散景光斑（Bokeh），瞬间呈现惊艳的微缩沙盘模型质感！
   - *(注：若坚持做纯正复古等大正交视角，UE 5.3+ 已对 **Orthographic Camera 正交渲染管线** 进行了史诗级重构，正式打通了正交相机下的动态阴影、Lumen 与后期特效支持！)*
3. **复杂动作游戏的动画状态机升级方案（`PaperZD` 生态融合）**：
   - 原生 `PaperFlipbook` 不带 3D 骨骼动画那样的可视化连线状态机（AnimGraph）与动画帧通知（AnimNotify，如第 4 帧精准开启刀光攻击判定框、播放脚步音效）。在商业级 2D 动作项目中，业界普遍采用 Epic Fab 商城免费开源的王牌扩展插件 **`PaperZD`**——它直接构建在原生 `Paper2D` 资产之上，为 `PaperFlipbook` 补齐了完整的 **AnimBP 状态机蓝图、多方向混合树（Directional AnimSequence）与帧级精确 `AnimNotify` 事件轨道**！
4. **关闭像素画杀手——动态模糊与抗锯齿调优**：
   - 像素风项目务必在项目设置或 `PostProcessVolume` 中将 **`Motion Blur Amount`（动态模糊）设为 `0`**，否则角色高速冲刺跳跃时精美的像素序列帧会被涂抹成残影；同时如果使用 TSR / TAA 时发现精细像素边缘在移动中有轻微柔化拖影，可切换为 `MSAA` / `FXAA` 或调高 `r.TSR.History.SampleCount` 与锐化参数。

---

## 六、 附录：`Paper2D` 全部 7 座展台（`#1`～`#7`）与核心资产核对速查表

| 档案编号 | 官方展牌标题 (`Title`) | 展牌 Actor 标签 | 世界坐标 `(X, Y, Z)` | 实测核心示例资产与关键参数 | 正文章节定位 |
| :---: | :--- | :--- | :---: | :--- | :--- |
| **#1** | `Paper 2D` | `BP_DemoDisplay7` | `(495, -900, 0)` | 可移动太阳光/天光套件 + 关卡蓝图 `Paper2D_C` (`Timeline_0`) | §一 |
| **#2** | `Sprites` 📷 | `BP_DemoDisplay1` | `(1515, -900, 0)` | `PaperSprite` (`512x512`, `PPU=5`) + 19 节点受光材质 `CustomSpriteMaterial` | §二 |
| **#3** | `Sprite Editor` | `BP_DemoDisplay2` | `(2500, -900, 0)` | `PaperSprite2` (图集裁切)、`PaperSprite3/4` (16 顶点自定义几何减 Overdraw)、`PaperSpriteCollision` (33 顶点碰撞) | §二 |
| **#4** | `Flipbooks` 📷 | `BP_DemoDisplay3` | `(3520, -900, 0)` | Jazz Jackrabbit `RunAnim` (`10 FPS` 匀速) vs `RunAnim2` (`FrameRun=2` 关键帧顿挫) | §三 |
| **#5** | `Control via Blueprints` | `BP_DemoDisplay6` | `(4505, -900, 0)` | `BP_Flipbook` (`IdleAnim` / `WalkAnim`) + 5 组 `TriggerBox1~5` 播放/倒放/停止/循环/切换控制 | §三 |
| **#6** | `Physics` 📷 | `BP_DemoDisplay4` | `(5510, -900, 0)` | 11 枚 `PaperSpriteCollision` (`Locked Axis = Y`) + `BlockingVolume1~3` + `RadialForceActor1~2` | §四 |
| **#7** | `Classic Pixel Sprites` | `BP_DemoDisplay5` | `(6495, -900, 0)` | `T_Jazzrun_D_Sprite_Default` vs `T_Jazzrun_D_Sprite` (`NoMipmaps` + `Nearest` + `UserInterface2D`) | §四 |
