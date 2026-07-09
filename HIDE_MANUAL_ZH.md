# Hide 中文使用手册

> 本手册基于当前仓库源码阅读整理，面向使用 Heaps 开发游戏时如何把 Hide 作为资源与场景编辑器使用。Hide 是可扩展编辑器，不同项目可通过 `res/props.json` 和插件增加自己的 prefab、菜单、视图与资源规则。
>
> 相关源码入口可从 [README.md](README.md)、[hide_js/hide/Ide.hx](hide_js/hide/Ide.hx)、[hide_js/hide/view/FileBrowser.hx](hide_js/hide/view/FileBrowser.hx)、[hide_js/hide/view/Prefab.hx](hide_js/hide/view/Prefab.hx)、[hide_js/hide/comp/SceneEditor.hx](hide_js/hide/comp/SceneEditor.hx)、[hrt/prefab/Prefab.hx](hrt/prefab/Prefab.hx) 开始阅读。

## 1. Hide 在 Heaps 开发流程中的作用

Hide（Heaps IDE）不是一个替代 Haxe/Heaps 代码的“完整游戏编辑器”，而是一个**资源中间件与可视化数据编辑器**。它主要解决以下问题：

- 预览和调试 3D 模型、贴图、音频、粒子、动画、shader 等资源。
- 创建和维护 `.prefab` / `.l3d` 场景数据，用层级树描述游戏对象、灯光、材质、地形、特效、引用等。
- 编辑 CastleDB（`.cdb`）游戏数据表，并让 prefab、代码、资源路径互相引用。
- 通过图编辑器创建 shader graph、texture graph、animation graph、blend space、FX timeline 等数据。
- 允许项目通过插件注册自定义 prefab、文件视图、菜单和属性面板。

Hide 的核心指导思想是：

1. **代码负责行为，Hide 负责数据与资源装配。** 复杂逻辑仍写在 Haxe/Heaps 代码中；Hide 用来组织可视对象、资源引用、调参数据、动画/特效时间线、数据库。
2. **编辑的是 prefab 数据树，运行时生成 Heaps 对象。** `.prefab` / `.l3d` / `.fx` 保存 JSON/BSON 风格数据树，运行时通过 `hrt.prefab.Prefab.make()` 生成 `h2d.Object` / `h3d.scene.Object` 等实际对象，参见 [hrt/prefab/Prefab.hx:152-168](hrt/prefab/Prefab.hx#L152-L168)。
3. **局部修改优先同步，结构变化再重建。** 属性面板修改会调用 `updateInstance()` 同步运行时对象；新增/删除/父子关系变化会由 `SceneEditor.queueRebuild()` / `rebuild()` 重建对应子树，参见 [hide_js/hide/comp/SceneEditor.hx:5377](hide_js/hide/comp/SceneEditor.hx#L5377) 与 [hide_js/hide/comp/SceneEditor.hx:5564](hide_js/hide/comp/SceneEditor.hx#L5564)。
4. **资源路径相对项目资源目录。** Hide 打开的是项目根目录或 `res` 目录的父目录；资源通常通过 `hxd.res.Loader.currentInstance.load(path)` 或 Heaps 的资源系统加载。
5. **编辑器可被项目扩展。** 自定义 prefab 通过 `Prefab.register()` 注册，自定义文件视图通过 `Extension.registerExtension()` 注册，见 [README.md:58-159](README.md#L58-L159)。

### 1.1 数据与运行时的衔接

- 每个 prefab 节点都有 `type`、`name`、`source`、`children` 等序列化字段，参见 [hrt/prefab/Prefab.hx:57-120](hrt/prefab/Prefab.hx#L57-L120)。
- 类型注册由 `Prefab.register(typeName, prefabClass, ?extension)` 完成，反序列化时 `createFromDynamic()` 根据 `type` 创建类实例，见 [hrt/prefab/Prefab.hx:961-983](hrt/prefab/Prefab.hx#L961-L983) 与 [hrt/prefab/Prefab.hx:1034-1050](hrt/prefab/Prefab.hx#L1034-L1050)。
- 3D prefab 继承 `Object3D`，最终创建 `h3d.scene.Object` 并同步 transform，见 [hrt/prefab/Object3D.hx](hrt/prefab/Object3D.hx)。
- 2D prefab 继承 `Object2D`，最终创建 `h2d.Object`，见 [hrt/prefab/Object2D.hx](hrt/prefab/Object2D.hx)。
- `ContextShared` 保存当前 `root2d/root3d/current2d/current3d` 与资源加载上下文，是 prefab 树与 Heaps 场景图之间的桥，见 [hrt/prefab/ContextShared.hx](hrt/prefab/ContextShared.hx)。

### 1.2 建议工作流

1. 用 Haxe/Heaps 建立项目结构与运行时代码。
2. 把美术、音频、配置、数据库放入项目资源目录（通常是 `res`）。
3. 用 Hide 打开项目根目录或 `res` 的父目录。
4. 在 Hide 中创建 `.prefab`、`.l3d`、`.fx`、`.cdb`、shader/texture/animation graph 等资源。
5. 在运行时代码中加载这些资源：模型/贴图/音频直接用 Heaps 资源系统，prefab/FX/graph 用对应 `toPrefab()`、`toModel()`、`toTexture()` 等转换。
6. 在 Hide 和游戏运行时之间保持清晰边界：Hide 产出数据；游戏代码解释、实例化、播放和驱动这些数据。

Heaps 官方示例页 [sample games for learning](https://heaps.io/documentation/sample-games-for-learning.html) 把示例分成“无资源模板”和“带资源项目”。学习 Hide 时建议先跑通无资源模板，再研究带资源示例的 `res` 目录结构、资源加载和数据驱动流程。

## 2. 安装、启动与项目配置

### 2.1 编译与启动

源码编译和运行方式见 [README.md:21-51](README.md#L21-L51)：

- 安装 Haxe、Heaps、CastleDB、HashLink、hxnodejs、domkit 等依赖。
- 执行 `haxe hide.hxml` 生成 [bin/hide.js](bin/hide.js)。
- 安装 NW.js SDK 到 `bin/nwjs`。
- Windows 运行 [bin/hide.cmd](bin/hide.cmd)，Linux 运行 `nwjs/nw .`，macOS 打开 NWJS 应用。

### 2.2 打开项目

- 菜单：`Project > Open...`。
- 如果选择的是 `res` 目录，Hide 会自动使用其父目录作为项目根，见 [hide_js/hide/Ide.hx:1472-1479](hide_js/hide/Ide.hx#L1472-L1479)。
- 最近打开项目在 `Project > Recently opened`。
- 项目打开后，默认资源目录、数据库文件、渲染器、插件、快捷键等由配置层叠决定。

### 2.3 `res/props.json`

项目资源目录下可创建 `props.json` 覆盖默认设置。默认配置集中在 [bin/defaultProps.json](bin/defaultProps.json)。常用项：

- `plugins`：加载项目自定义 JS/CSS 插件。
- `renderers` / `defaultRenderer`：注册和选择渲染器。
- `cdb.databaseFile`：默认数据库文件，通常是 `data.cdb`。
- `haxe.classPath`：供脚本/shader/代码解析使用的 classpath。
- `sceneeditor.*`：场景编辑器网格、快捷键、可新建 prefab 分组、地形画刷等。
- `fs.convert`：资源转换规则，如贴图压缩。
- `fx.shaders` / `shadergraph.libfolders`：FX 与 shader graph 可用 shader 路径。
- `domkit.css`：DomKit Studio 可加载的样式文件。

## 3. 编辑器界面与通用操作

### 3.1 主菜单

主菜单定义在 [bin/app.html:67-146](bin/app.html#L67-L146)，并由 [hide_js/hide/Ide.hx:1450-1758](hide_js/hide/Ide.hx#L1450-L1758) 绑定。主要菜单：

- `Project`：打开项目、最近项目、选择渲染器、Build Files、清理 profile、退出。
- `View`：打开 Resources、File Browser、Domkit Studio、About、Debug、Editor Gym。
- `Database`：打开 CDB、Custom Types、Formulas、Diff、导入/导出本地化文本、Proofreading、Categories、Compression。
- `Layout`：布局自动保存、保存、另存为。
- `Tools`：Remote console、Memory profiler、DevTools、GPU dump、Screen Capture。
- `Settings`：User settings、Project settings。

### 3.2 文件浏览器 / Resources

打开方式：`View > Resources` 或 `View > File Browser`。

文件浏览器由 [hide_js/hide/view/FileBrowser.hx](hide_js/hide/view/FileBrowser.hx) 实现，支持：

- 树形模式、图库/缩略图模式、水平/垂直混合布局。
- 双击文件打开对应编辑器；双击目录展开或进入目录。
- 右键目录：`New...` 创建目录或已注册资产类型。
- 右键文件：复制路径、绝对路径、打开系统资源管理器、Find References、Clone、Rename、Move、Delete、Replace Refs With、刷新缩略图等，见 [hide_js/hide/view/FileBrowser.hx:1132-1377](hide_js/hide/view/FileBrowser.hx#L1132-L1377)。
- Favorites：右键 `Mark as Favorite` / `Remove from Favorite`。
- 搜索和过滤：工具栏搜索框、全路径搜索开关、过滤按钮。
- Gallery 中可用 Ctrl + 鼠标滚轮调整缩略图大小，Alt 悬停预览。

### 3.3 新建文件逻辑

可新建文件类型来自 `Extension.registerExtension()` 的 `createNew` 选项，见 [hide_js/hide/Extension.hx:18-35](hide_js/hide/Extension.hx#L18-L35)。右键目录选择 `New...` 后，Hide 会：

1. 询问文件名。
2. 如果未输入扩展名，使用注册扩展名的第一段。
3. 调用对应 `FileView.getDefaultContent()` 写入默认内容。
4. 打开新文件。

具体实现见 [hide_js/hide/view/FileBrowser.hx:1035-1068](hide_js/hide/view/FileBrowser.hx#L1035-L1068)。

### 3.4 文件类型注册与创建对照表

Hide 的文件编辑器由 [hide_js/hide/Extension.hx](hide_js/hide/Extension.hx) 注册。普通文件按扩展名分派；`.json` 会先解析根对象的 `type` 字段，并尝试查找 `json.<type>`，因此 `particles2D` / `particles3D` 这类 typed JSON 会打开专用编辑器，而不是普通 JSON 文本编辑器。

| 类型 | 扩展名 / 类型名 | 是否可从 File Browser 新建 | 默认编辑器 | 保存内容 | 运行时入口 |
| --- | --- | --- | --- | --- | --- |
| 图片 / 纹理 | `.jpg .jpeg .gif .png .raw .dds .hdr .tga .envd .envs` | 否 | Image | 压缩/转换规则到 `props.json` | `toImage()` / `toTexture()` |
| Prefab | `.prefab` | 是：Prefab | Prefab | prefab JSON | `toPrefab().load().make(...)` |
| Level | `.l3d` | 否 | Prefab | prefab JSON | `level3d` prefab extension |
| Model | `.hmd .fbx` | 否 | Model | `model.props`、动画事件等 | `toModel().toHmd()` / `hrt.prefab.Model` |
| 2D 粒子 | `json.particles2D` | 是：Particle 2D | Particles2D | `h2d.Particles.save()` JSON | `h2d.Particles.load(...)` |
| 3D 粒子 | `json.particles3D` | 是：Particle 3D | Particles3D | `h3d.parts.GpuParticles.save()` JSON | `h3d.parts.GpuParticles.load(...)` |
| FX | `.fx` | 是：FX | FXEditor | prefab JSON | `hrt.prefab.fx.FX` / `FXAnimation` |
| FX 2D | `.fx2d` | 是：FX 2D | FXEditor | prefab JSON | `hrt.prefab.fx.FX2D` / `FX2DAnimation` |
| Texture Graph | `.texgraph` | 是：Texture Graph | TextureEditor | graph JSON | `TexGraph.generate()` |
| Shader Graph | `.shgraph` | 是：Shader Graph | ShaderEditor | graph JSON | `ShaderGraph.compile()` / `DynamicShader` |
| Animation Graph | `.animgraph` | 是：Anim Graph | AnimGraphEditor | graph JSON | `hrt.animgraph` |
| Blend Space 2D | `.bs2d` | 是：Blend Space 2D | BlendSpace2DEditor | blend space JSON | `BlendSpace2D.makeAnimation(...)` |
| CDB | 默认 `data.cdb` | 在 CDB view 内创建 sheet | CdbTable | CastleDB 数据库 | CastleDB API / 生成类型 |
| DomKit Less | `.less` | 否 | DomkitLess | Less 文本 | DomKit UI 样式 |
| 声音 | `.wav .mp3 .ogg` | 否 | Sound | 通常不改写本体 | `toSound()` |
| 文本/脚本 | `.hx .js .json .xml .html` | 否 | Script | 文本 | 源码/配置/自定义数据 |
| SVG/MSDF | `.svg` | 否 | MSDFView | 相关配置/生成链路 | 字体/图标/MSDF 渲染 |

### 3.5 Tab、停靠与布局

- Hide 使用 GoldenLayout 管理窗口与 tab。
- 打开文件时 `Ide.openFile()` 根据扩展名找到对应视图；若已打开则激活原 tab，见 [hide_js/hide/Ide.hx:1771-1784](hide_js/hide/Ide.hx#L1771-L1784)。
- tab 右键通常支持 Save、Save As、Reload、Copy Path、Open in Explorer、Find References、Close、Close Others、Close All。
- `F11` 全屏当前 View。
- `Ctrl-Shift-T` 重新打开最近关闭的 tab。
- `Layout > Keep on close` 可控制布局自动保存。

### 3.5 通用快捷键

默认快捷键在 [bin/defaultProps.json:26-112](bin/defaultProps.json#L26-L112)。常用：

| 功能 | 快捷键 |
| --- | --- |
| 取消 / 取消选择 | `Escape` |
| 撤销 / 重做 | `Ctrl-Z` / `Ctrl-Y` |
| 保存 | `Ctrl-S` |
| 搜索 | `Ctrl-F` |
| 重命名 | `F2` |
| 复制 / 剪切 / 粘贴 | `Ctrl-C` / `Ctrl-X` / `Ctrl-V` |
| 复制/克隆 | `Ctrl-D` 或 `Ctrl-Shift-D` |
| 删除 | `Delete` |
| 全屏当前 View | `F11` |
| 刷新 View | `F5` |
| 刷新应用 | `Ctrl-F5` |
| 重新打开关闭的 tab | `Ctrl-Shift-T` |

快捷键按鼠标所在 View 分发，文本输入框会阻止大多数编辑器快捷键抢占，见 [hide_js/hide/ui/Keys.hx](hide_js/hide/ui/Keys.hx) 与 [hide_js/hide/ui/View.hx](hide_js/hide/ui/View.hx)。

### 3.6 保存、外部变更与备份

所有文件视图继承 [hide_js/hide/view/FileView.hx](hide_js/hide/view/FileView.hx)。通用行为：

- `Ctrl-S` 调用当前视图 `save()`。
- 标题带 `*` 表示未保存。
- 外部文件变化会提示是否重载，见 [hide_js/hide/view/FileView.hx:74-93](hide_js/hide/view/FileView.hx#L74-L93)。
- 保存前会基于内容签名判断变化；部分视图保存时会在 `res/.tmp/...` 生成最多 10 个备份，见 [hide_js/hide/view/FileView.hx:132-157](hide_js/hide/view/FileView.hx#L132-L157)。

## 4. 场景 / Prefab 编辑器通用操作

`.prefab`、`.l3d`、`.fx` 等场景类资产最终都使用或扩展 `SceneEditor`。界面一般包括：

- 中央 Heaps 视口。
- Scene Tree 层级树。
- Properties 属性面板。
- Render Props 树。
- 顶部工具栏。

### 4.1 相机导航

相机控制器在 [hide_js/hide/view/CameraController.hx](hide_js/hide/view/CameraController.hx)，设置面板在 [hide_js/hide/comp/CameraControllerEditor.hx](hide_js/hide/comp/CameraControllerEditor.hx)。常见方式：

- 鼠标滚轮：缩放或在 FPS/Flight 拖拽状态下调节移动速度。
- 中键 / 右键 / `Alt + 左键`：根据当前控制器执行平移、旋转、视角环绕或 FPS look。
- 按住右键或中键时，`WASD` / `ZQSD` / 方向键移动相机。
- `F`：聚焦选中对象。
- 工具栏相机按钮：透视相机、Top camera、Camera Settings。
- Camera Settings 可切换 Legacy / FPS / Flight / Ortho 等控制方式，并调整 FOV、速度、near/far、snap to ground。

源码注释说明 Legacy 与 FPS 的差异：Legacy 中键旋转、右键平移；FPS 中键平移、右键环视并配合方向键/ZQSD 飞行，见 [hide_js/hide/comp/CameraControllerEditor.hx:88-90](hide_js/hide/comp/CameraControllerEditor.hx#L88-L90)。

### 4.2 选择对象

- 在视口点击对象选择。
- `Ctrl + 点击`：多选/切换选择。
- `Shift + 点击`：选择父级或同级范围（取决于命中对象和树）。
- 点击空白：清空选择。
- Scene Tree 单击同步选择；双击通常聚焦对象。
- `Escape`：取消选择。
- `Ctrl-A`：全选，`Ctrl-I`：反选。
- `Backspace`：选择父对象。

选择逻辑集中在 [hide_js/hide/comp/SceneEditor.hx:1746-1883](hide_js/hide/comp/SceneEditor.hx#L1746-L1883) 与 [hide_js/hide/comp/SceneEditor.hx:4023-4216](hide_js/hide/comp/SceneEditor.hx#L4023-L4216)。

### 4.3 Transform Gizmo

工具栏与快捷键：

| 功能 | 快捷键 / 按钮 |
| --- | --- |
| 平移 | `W` / arrows 图标 |
| 旋转 | `E` / refresh 图标 |
| 缩放 | `R` / expand 图标 |
| 切换模式 | `Space` |
| 本地坐标 | `D` / compass 图标 |
| Snap 开关 | `X` / magnet 图标 |
| 网格显示 | `G` |
| 吸附到地面 | anchor 图标 |
| 尺子工具 | `M` / ruler 图标 |
| 隐藏/显示树与属性列 | `Tab` |

Snap 规则：

- Snap Toggle 开启时默认吸附，按 `Ctrl` 临时反转。
- Snap Toggle 关闭时按 `Ctrl` 临时吸附。
- Snap Settings 可设置平移、旋转、缩放步长和 Force On Grid。

Transform 实现见 [hide_js/hide/comp/SceneEditor.hx:3234-3463](hide_js/hide/comp/SceneEditor.hx#L3234-L3463)。

### 4.4 Scene Tree 右键菜单

在 `.prefab` / `.l3d` / `.fx` 的 Scene Tree 中右键可：

- `New...`：按分组新建 prefab 节点。
- Rename / Delete / Duplicate。
- Enable、Editor only、In game only。
- Hide Selection、Isolate、Show All、Group、Reparent。
- Export（部分对象支持）。
- Tag（如果项目配置了 tags）。
- 自定义 prefab 可注册自己的右键菜单。

新增菜单来自 prefab registry 和 `sceneeditor.newgroups` 配置，见 [hide_js/hide/comp/SceneEditor.hx:5689-5815](hide_js/hide/comp/SceneEditor.hx#L5689-L5815) 与 [bin/defaultProps.json:160-165](bin/defaultProps.json#L160-L165)。

## 5. 各类资产的创建、编辑与使用

### 5.1 项目配置：`props.json`

**创建**

- 在资源目录 `res` 下手动创建 `props.json`，或由贴图压缩等编辑器自动写入局部 `props.json`。
- 可参考 [bin/defaultProps.json](bin/defaultProps.json)。

**编辑**

- 菜单 `Settings > Project settings` / `User settings` 可打开设置视图。
- 文本方式可用内置 JSON editor 打开。

**使用**

- Hide 启动/打开项目时加载全局、项目、用户配置。
- 项目配置影响文件视图、资源转换、CDB、场景编辑器、快捷键、插件等。

### 5.2 代码与文本：`.hx` / `.js` / `.json` / `.xml` / `.html`

**创建**

- Hide 默认只把这些文件注册为可打开视图，不一定提供右键 `New` 项。
- 可在外部编辑器创建，或通过项目自定义 FileView 添加创建项。

**编辑**

- `.hx` / `.js` 使用 Script 视图；`.hx` 会启用脚本检查器。
- `.json`、`.xml`、`.html` 使用 Monaco 编辑器。
- XML tab 菜单提供 `Count Words`，适合本地化文本统计。

**使用**

- Haxe 源码由项目构建系统使用。
- JSON/XML/HTML 可作为游戏配置、UI 或自定义数据。
- 视图实现见 [hide_js/hide/view/Script.hx](hide_js/hide/view/Script.hx)。

### 5.3 图像 / 贴图：`.png` / `.jpg` / `.jpeg` / `.gif` / `.raw` / `.dds` / `.hdr` / `.tga` / `.envd` / `.envs`

**创建**

- 通常由外部 DCC/图像工具生成后放入资源目录。
- Hide 负责预览和压缩配置，不负责绘制图像。

**编辑**

打开图像后可：

- 查看压缩后/未压缩/对比模式。
- 切换 RGBA 通道。
- 查看 array texture 的 layer、mip、曝光等。
- 配置压缩格式、mip maps、最大尺寸、滤镜、BC1 alpha threshold。
- `Save` 将压缩规则写入同目录 `props.json` 的 `fs.convert`。
- `Reset compression` 删除当前贴图的转换规则。

相关代码见 [hide_js/hide/view/Image.hx:100-423](hide_js/hide/view/Image.hx#L100-L423)。支持扩展名在 [hide_js/hide/Ide.hx:974](hide_js/hide/Ide.hx#L974)。

**使用**

- 运行时代码：通过 Heaps 资源系统加载为 `h3d.mat.Texture` 或 `h2d.Tile`。
- Prefab 中：材质贴图、粒子贴图、地形 surface、UI 背景等属性可选择图像路径。
- 贴图压缩规则由资源转换系统读取。

### 5.4 3D 模型：`.hmd` / `.fbx`

**创建**

- 通常由外部 DCC 或 Heaps pipeline 生成 `.fbx` / `.hmd`。
- Hide 注册 Model 视图用于预览与附加数据编辑，见 [hide_js/hide/view/Model.hx:3131](hide_js/hide/view/Model.hx#L3131)。

**编辑**

打开模型后可：

- 查看模型层级、mesh、material、joint、collision、animation events。
- 选择动画，使用 `PageUp` / `PageDown` 切换动画。
- 编辑材质属性、碰撞设置、动态骨骼等模型附加配置。
- 保存时写入 `model.props` 或动画事件数据，见 [hide_js/hide/view/Model.hx:478-578](hide_js/hide/view/Model.hx#L478-L578)。
- Tab 菜单可导出 HMD dump 或动画 dump，见 [hide_js/hide/view/Model.hx:594-620](hide_js/hide/view/Model.hx#L594-L620)。

**使用**

- 运行时代码可加载：`hxd.res.Loader.currentInstance.load(path).toModel().toHmd()`。
- Prefab 中可通过 `Model` 节点引用模型；`hrt.prefab.Model` 使用 `shared.loadModel(source)` 和 `shared.loadAnimation(animation)` 创建对象与动画，见 [hrt/prefab/Model.hx](hrt/prefab/Model.hx)。

### 5.5 声音：`.wav` / `.mp3` / `.ogg`

**创建**

- 由外部音频工具生成后放入资源目录。

**编辑**

- Hide 注册 Sound 视图用于打开音频，见 [hide_js/hide/view/Sound.hx](hide_js/hide/view/Sound.hx)。
- 通常用于试听/检查资源，而不是音频编辑。

**使用**

- 运行时代码通过 Heaps 音频资源加载播放。
- 可在 CDB 或 prefab 自定义字段中引用声音路径。

### 5.6 CastleDB 数据库：`.cdb`

**创建**

- 默认数据库文件来自 `cdb.databaseFile`，默认是 `data.cdb`，见 [bin/defaultProps.json:116](bin/defaultProps.json#L116)。
- 如果数据库没有 sheet，打开 `Database > View` 后界面会提示创建 sheet，见 [hide_js/hide/view/CdbTable.hx:232-241](hide_js/hide/view/CdbTable.hx#L232-L241)。

**编辑**

菜单 `Database > View` 打开 CDB 表格编辑器。支持：

- sheet tab、category tab、proofreading mode。
- 行/列编辑、插入、删除、复制、粘贴、重复。
- 过滤 regular / warning / error。
- `F12` 跳转引用，`Ctrl-F12` 显示引用。
- `Ctrl-P` 全局 sheet 搜索，`Ctrl-O` 当前 sheet ID 搜索，`Ctrl-Shift-O` 全局 ID 搜索。
- formulas、custom types、diff、导入/导出本地化 XML。

快捷键注册见 [hide_js/hide/comp/cdb/Editor.hx:132-186](hide_js/hide/comp/cdb/Editor.hx#L132-L186)。CDB 视图入口见 [hide_js/hide/view/CdbTable.hx](hide_js/hide/view/CdbTable.hx)。

**使用**

- Hide 内部通过 [hide/tools/IdeData.hx:210-255](hide/tools/IdeData.hx#L210-L255) 加载 CDB，并通过 [hide/tools/IdeData.hx:301-354](hide/tools/IdeData.hx#L301-L354) 保存。
- 游戏代码可用 CastleDB API 或生成类型访问数据库。
- Prefab 可通过 `props` 或 CDB 引用把表行与场景对象关联。

### 5.7 Prefab：`.prefab`

**创建**

- 文件浏览器右键目录：`New... > Prefab`。
- 默认内容是一个空 `hrt.prefab.Prefab` 序列化结果，见 [hide_js/hide/view/Prefab.hx:580-582](hide_js/hide/view/Prefab.hx#L580-L582)。

**编辑**

打开 `.prefab` 后：

- Scene Tree 管理层级。
- Properties 编辑选中节点属性。
- 中央视口预览运行时实例。
- 右键 Scene Tree `New...` 添加 Object3D、Object2D、Model、Light、Material、Reference、Terrain、FX、RendererFX 等节点。
- 工具栏提供相机、gizmo、snap、local transform、ruler、overlays、view modes、filters、render props、speed 等。
- 保存时序列化 `data.serialize()` 到文件，见 [hide_js/hide/view/Prefab.hx:602-645](hide_js/hide/view/Prefab.hx#L602-L645)。

**使用**

运行时常见模式：

```haxe
var prefab = hxd.res.Loader.currentInstance.load("path/to/file.prefab").toPrefab().load();
prefab.make(scene3d); // 或传入 ContextMake / root object
```

编辑器和运行时都通过 [hrt/prefab/Resource.hx:46-60](hrt/prefab/Resource.hx#L46-L60) 加载 prefab 资源。

### 5.8 Level：`.l3d`

**创建**

- `.l3d` 是 level3d prefab 扩展，注册点在 [hrt/prefab/l3d/Level3D.hx:20](hrt/prefab/l3d/Level3D.hx#L20)。
- 当前文件浏览器没有给 `.l3d` 注册 `createNew`，通常通过已有文件、项目工具或自定义流程创建。

**编辑**

- 用 Prefab 视图打开，行为类似 `.prefab`，但语义上用于完整 3D level。
- 可放置灯光、模型、terrain、camera、environment、mesh spray、decal、render props 等。

**使用**

- 与 prefab 相同，通过 `toPrefab().load().make(...)` 加载并实例化。
- 可把 `.l3d` 作为关卡入口，游戏代码负责加载、切换、挂载到 scene。

### 5.9 3D Prefab 节点

Scene Tree `New... > 3D` 下常见类型来自注册系统，核心在 [hrt/prefab](hrt/prefab/) 与 [hrt/prefab/l3d](hrt/prefab/l3d/)。

#### Object3D

- 基础空 3D 节点，保存 transform/visible 等。
- 用作分组、挂点、父节点。
- 运行时创建 `h3d.scene.Object`，见 [hrt/prefab/Object3D.hx](hrt/prefab/Object3D.hx)。

#### Model

- 引用 `.hmd` / `.fbx`，可选择动画。
- 适合放置角色、道具、场景模型。
- 源码见 [hrt/prefab/Model.hx](hrt/prefab/Model.hx)。

#### Reference

- 引用另一个 `.prefab` / `.l3d` / `.fx`，用于复用对象集合。
- 支持编辑引用实例或 override。
- 源码见 [hrt/prefab/Reference.hx](hrt/prefab/Reference.hx)。

#### Instance

- 可从 CDB、模型或 prefab path 实例化对象。
- 适合数据驱动放置。
- 源码见 [hrt/prefab/l3d/Instance.hx](hrt/prefab/l3d/Instance.hx)。

#### Light

- 支持 Point、Directional、Spot、Capsule、Rectangle。
- 根据当前 renderer 创建 PBR 或 forward light。
- 可编辑颜色、power、shadow、cookie、cascade 等。
- 源码见 [hrt/prefab/Light.hx](hrt/prefab/Light.hx)。

#### Camera

- 用作关卡中的相机点位或运行时引用对象。
- 源码见 [hrt/prefab/l3d/Camera.hx](hrt/prefab/l3d/Camera.hx)。

#### Environment

- 管理环境贴图、光照环境等。
- 源码见 [hrt/prefab/l3d/Environment.hx](hrt/prefab/l3d/Environment.hx)。

#### Material / MaterialSelector / RenderProps

- `Material` 不创建独立 scene object，而是查找附近/父级对象材质并应用贴图、颜色、PBR props、material library、overrides。
- `RenderProps` 保存 renderer 属性。
- 源码见 [hrt/prefab/Material.hx](hrt/prefab/Material.hx)、[hrt/prefab/MaterialSelector.hx](hrt/prefab/MaterialSelector.hx)、[hrt/prefab/RenderProps.hx](hrt/prefab/RenderProps.hx)。

#### Terrain / HeightMap

- Terrain 创建 `TerrainMesh`、tiles、height/normal/surface texture 等。
- 属性面板和 TerrainEditor 提供画刷、surface、tile、undo 等编辑能力。
- 源码见 [hrt/prefab/terrain/Terrain.hx](hrt/prefab/terrain/Terrain.hx)、[hide_js/hide/prefab/terrain/TerrainEditor.hx](hide_js/hide/prefab/terrain/TerrainEditor.hx)。

#### MeshSpray / PrefabSpray / Decal / Polygon / Box / Text3D

- 用于场景布置、散布模型或 prefab、贴花、简单几何、3D 文本。
- 源码见 [hrt/prefab/l3d/MeshSpray.hx](hrt/prefab/l3d/MeshSpray.hx)、[hrt/prefab/l3d/PrefabSpray.hx](hrt/prefab/l3d/PrefabSpray.hx)、[hrt/prefab/l3d/Decal.hx](hrt/prefab/l3d/Decal.hx)、[hrt/prefab/l3d/Polygon.hx](hrt/prefab/l3d/Polygon.hx)、[hrt/prefab/l3d/Box.hx](hrt/prefab/l3d/Box.hx)、[hrt/prefab/l3d/Text3D.hx](hrt/prefab/l3d/Text3D.hx)。

### 5.10 2D Prefab 节点

Scene Tree `New... > 2D` 下常见类型在 [hrt/prefab/l2d](hrt/prefab/l2d/)。

- `Object2D`：基础 2D 分组节点。
- `Bitmap` / `Tile` / `Atlas`：贴图、tile、图集显示。
- `Text`：2D 文本。
- `Flow`：2D 布局容器。
- `Gradient` / `Blur` / `NoiseGenerator`：2D 效果或生成器。
- `Anim2D`：2D 动画。
- `Particle2D`：在 prefab 中使用 2D 粒子。
- `Layers2D`：在 3D 场景中组合 2D 层。

运行时最终挂到 `h2d.Object` 树，见 [hrt/prefab/Object2D.hx](hrt/prefab/Object2D.hx)。

### 5.11 独立 2D 粒子：`json.particles2D` / `.json`

**创建**

- 文件浏览器右键：`New... > Particle 2D`。
- 实际扩展名是 `.json`，JSON 根对象 `type` 用于匹配 `json.particles2D` 视图。
- 默认创建一个 `h2d.Particles`，含 `Default` group，见 [hide_js/hide/view/Particles2D.hx:17-21](hide_js/hide/view/Particles2D.hx#L17-L21)。

**编辑**

- 每个 group 可编辑显示、发射、生命周期、速度、尺寸、旋转、动画等。
- group 标题右键可 Enable、Copy、Paste、MoveUp、MoveDown、Delete。
- Manage 区可新建 group、显示 bounds。
- Background 区可设置背景贴图、偏移、smooth。
- 源码见 [hide_js/hide/view/Particles2D.hx](hide_js/hide/view/Particles2D.hx)。

**使用**

- 运行时代码可创建 `h2d.Particles` 并 `load()` 该 JSON。
- Prefab 中可用 `Particle2D` 节点引用/承载粒子数据。

### 5.12 独立 3D 粒子：`json.particles3D` / `.json`

**创建**

- 文件浏览器右键：`New... > Particle 3D`。
- 默认创建 `h3d.parts.GpuParticles`，含 `Default` group，见 [hide_js/hide/view/Particles3D.hx:12-16](hide_js/hide/view/Particles3D.hx#L12-L16)。

**编辑**

- group 可编辑 texture、color gradient、sort、3D transform、relative、material、emit、life、speed、size、rotation、animation。
- Manage 区可设置预览模型、attach joint/object、animation、show bounds、enable lights、新建 group、reset camera。
- 源码见 [hide_js/hide/view/Particles3D.hx](hide_js/hide/view/Particles3D.hx)。

**使用**

- 运行时代码可创建 `h3d.parts.GpuParticles` 并 `load()` JSON。
- Prefab 中 `Particles3D` 节点也可通过 `source` 或内嵌 `data` 加载，见 [hrt/prefab/l3d/Particles3D.hx](hrt/prefab/l3d/Particles3D.hx)。

### 5.13 FX / Timeline：`.fx` / `.fx2d`

**创建**

- 文件浏览器右键：`New... > FX` 或 `New... > FX 2D`。
- 默认内容是 `hrt.prefab.fx.FX` / `FX2D` 的序列化数据，见 [hide_js/hide/view/FXEditor.hx:342-344](hide_js/hide/view/FXEditor.hx#L342-L344) 与 [hide_js/hide/view/FXEditor.hx:1858-1862](hide_js/hide/view/FXEditor.hx#L1858-L1862)。

**编辑**

FX Editor 由视口、Scene Tree、Properties、timeline/curve panel 组成，见 [hide_js/hide/view/FXEditor.hx:367-510](hide_js/hide/view/FXEditor.hx#L367-L510)。常用操作：

- 工具栏：camera、gizmo、snap、local/self transform、overlays、view modes、render props、pause、loop、speed。
- `Space`：播放/暂停。
- `O`：循环。
- `Ctrl-Space`：从头播放。
- `Ctrl-Shift-Space`：从上次 seek 处播放。
- Timeline 快捷提示：`Ctrl + Click` 插入 key，右键编辑 keyframe，拖动时 `Ctrl` snap，`Shift` 限制/缩放 Y 轴，`Alt` 限制/缩放 X 轴，`F` 缩放曲线，鼠标滚轮移动曲线图，见 [hide_js/hide/view/FXEditor.hx:457-465](hide_js/hide/view/FXEditor.hx#L457-L465)。
- 可添加 Event、Emitter、Trail、SubFX、ShaderTarget、Constraint、RendererFX 等子节点。

**使用**

- `.fx` 注册为 prefab 扩展，见 [hrt/prefab/fx/FX.hx:1178](hrt/prefab/fx/FX.hx#L1178)。
- 运行时创建 `FXAnimation`，由 `syncRec()` 推进时间线和粒子/事件/动画，见 [hrt/prefab/fx/FX.hx](hrt/prefab/fx/FX.hx)。
- 可作为普通 prefab 加载并 `make()`，也可被 Reference/SubFX 引用。

### 5.14 Shader Graph：`.shgraph`

**创建**

- 文件浏览器右键：`New... > Shader Graph`。
- 默认创建 `hrt.shgraph.ShaderGraph` 序列化数据，见 [hide_js/hide/view/shadereditor/ShaderEditor.hx:1997-1999](hide_js/hide/view/shadereditor/ShaderEditor.hx#L1997-L1999)。

**编辑**

- 图编辑器中添加节点、连接输入输出、设置参数。
- 可预览 mesh / bitmap / shader 输出。
- 修改节点或连线后会 `requestRecompile()`，编译逻辑见 [hide_js/hide/view/shadereditor/ShaderEditor.hx:2002-2021](hide_js/hide/view/shadereditor/ShaderEditor.hx#L2002-L2021)。
- 文件浏览器右键 `.shgraph` 可 `Convert Shadergraph to HXSL`，见 [hide_js/hide/view/FileBrowser.hx:1294-1306](hide_js/hide/view/FileBrowser.hx#L1294-L1306)。

**使用**

- Prefab 中可通过 `DynamicShader` / shader 节点引用 `.shgraph`。
- RendererFX 中也可使用 screen shader graph。
- 运行时编译为 HXSL shader 定义。

### 5.15 Texture Graph：`.texgraph`

**创建**

- 文件浏览器右键：`New... > Texture Graph`。
- 默认创建 `hrt.texgraph.TexGraph` 序列化数据，见 [hide_js/hide/view/textureeditor/TextureEditor.hx:150-153](hide_js/hide/view/textureeditor/TextureEditor.hx#L150-L153)。

**编辑**

- 使用图节点生成贴图。
- 右侧/下方可预览输出贴图，`Center preview` 可重置预览相机。
- 输出节点变化后捕获 texture pixels 并显示 2D preview，见 [hide_js/hide/view/textureeditor/TextureEditor.hx:162-198](hide_js/hide/view/textureeditor/TextureEditor.hx#L162-L198)。

**使用**

- 可作为程序化贴图生成资源，或被 shader / material / prefab 引用。
- 具体运行时用法取决于项目如何加载 `hrt.texgraph.TexGraph`。

### 5.16 Animation Graph：`.animgraph`

**创建**

- 文件浏览器右键：`New... > Anim Graph`。
- 默认创建 `hrt.animgraph.AnimGraph`，包含一个 Output 节点，见 [hide_js/hide/view/animgraph/AnimGraphEditor.hx:406-410](hide_js/hide/view/animgraph/AnimGraphEditor.hx#L406-L410)。

**编辑**

- 图编辑器添加 animation input、blend、parameter、output 等节点。
- 可编辑参数列表，移动参数顺序，预览模型动画。
- 删除键删除选中节点/连接。

**使用**

- 运行时通过 `hrt.animgraph` 系统构建动画状态/混合逻辑。
- 模型预览和节点输入使用 `hxd.res.Loader.currentInstance.load(path).toModel()` 加载动画，见 [hrt/animgraph/nodes/Input.hx](hrt/animgraph/nodes/Input.hx)。

### 5.17 Blend Space 2D：`.bs2d`

**创建**

- 文件浏览器右键：`New... > Blend Space 2D`。
- 默认创建 `hrt.animgraph.BlendSpace2D`，见 [hide_js/hide/view/animgraph/BlendSpace2DEditor.hx:350-353](hide_js/hide/view/animgraph/BlendSpace2DEditor.hx#L350-L353)。

**编辑**

- 编辑 X/Y 范围、smooth、边界外速度缩放。
- 添加/选择 point，为 point 配置动画。
- 可预览模型动画混合。
- Tab 菜单 `Reset Model Folder` 可重置动画目录。

**使用**

- 运行时调用 `BlendSpace2D.makeAnimation(...)` 生成可播放动画混合，预览逻辑见 [hide_js/hide/view/animgraph/BlendSpace2DEditor.hx:355-387](hide_js/hide/view/animgraph/BlendSpace2DEditor.hx#L355-L387)。

### 5.18 DomKit / Less：`.less`

**创建**

- `.less` 作为 Domkit Studio 相关资源打开，Hide 默认注册 Less 视图但不提供 createNew，见 [hide_js/hide/view/DomkitStudio.hx:373](hide_js/hide/view/DomkitStudio.hx#L373)。

**编辑**

- 菜单 `View > Domkit Studio`。
- 项目配置 `domkit.css` 决定可加载样式文件。

**使用**

- 用于 Heaps DomKit UI 样式调试和布局预览。

### 5.19 SVG / MSDF：`.svg`

**创建**

- 外部矢量工具创建 `.svg`。

**编辑/预览**

- Hide 注册 MSDF 视图打开 `.svg`，见 [hide_js/hide/view/MSDFView.hx](hide_js/hide/view/MSDFView.hx)。
- 可用于检查/生成 MSDF 字形或图标相关资源。

**使用**

- 项目可把 SVG/MSDF 作为字体、图标、UI 资源使用。

## 6. 在代码中使用 Hide 资产

### 6.1 加载 prefab / level / fx

```haxe
// 加载 prefab 数据
var res = hxd.res.Loader.currentInstance.load("objects/chest.prefab").toPrefab();
var prefab = res.load();

// 生成到 3D 场景
prefab.make(s3d);
```

对于 `.l3d` 和 `.fx`，本质也是 prefab extension：

```haxe
var level = hxd.res.Loader.currentInstance.load("levels/level01.l3d").toPrefab().load();
level.make(s3d);

var fx = hxd.res.Loader.currentInstance.load("fx/explosion.fx").toPrefab().load();
fx.make(s3d);
```

### 6.2 加载模型、贴图、动画

```haxe
var tex = hxd.res.Loader.currentInstance.load("textures/diffuse.png").toTexture();
var modelLib = hxd.res.Loader.currentInstance.load("models/hero.hmd").toModel().toHmd();
var obj = modelLib.makeObject();
s3d.addChild(obj);
```

### 6.3 使用 CDB

- 在 Hide 中维护 `data.cdb`。
- 在代码中使用 CastleDB API 或生成类型访问数据。
- 推荐把资源路径、prefab path、数值参数放 CDB；把逻辑和行为写在 Haxe。

### 6.4 自定义 prefab

最小示例见 [README.md:95-124](README.md#L95-L124)。要点：

- 继承 `hrt.prefab.Object3D` / `Object2D` / `Prefab`。
- 重写 `make()`、`makeObject()` 或 `makeInstance()` 创建运行时对象。
- `#if editor` 中重写 `getHideProps()` 返回名称、图标、可选资源类型。
- 静态调用 `Library.register` 或 `Prefab.register` 注册类型。
- 把自定义类编译进 Hide plugin，并在 `res/props.json` 的 `plugins` 中加载。

### 6.5 自定义文件视图

见 [README.md:127-159](README.md#L127-L159)。要点：

- 继承 `hide.view.FileView` 或 `hide.ui.View`。
- 实现 `onDisplay()` 构建 HTML/组件。
- `getPath()` 获取文件实际路径。
- `Extension.registerExtension(CustomView, ["ext"], { icon, createNew, name })` 注册扩展名。
- JSON 视图可用 `json.someType` 匹配根对象 `type: "someType"`。

## 7. 建议与最佳实践

1. **把可调数据放 Hide，行为放代码。** 例如敌人 AI 写 Haxe，血量、速度、掉落、prefab path 放 CDB 或 prefab props。
2. **Prefab 粒度适中。** 可复用对象做 `.prefab`，完整关卡做 `.l3d`，复杂特效做 `.fx`。
3. **Reference 优先于复制。** 需要复用对象组合时，使用 Reference，避免多份 prefab 内容失去同步。
4. **资源路径保持相对和稳定。** 使用 FileBrowser 的 Rename/Move/Replace Refs With，减少手动改路径。
5. **使用 `editorOnly` / `inGameOnly`。** 编辑辅助对象可设为 editorOnly，运行时专用对象可设为 inGameOnly。
6. **复杂地形、FX、shader graph 变更后及时保存并在游戏中验证。** Hide 预览不等同于完整游戏环境。
7. **把项目扩展沉淀为插件。** 如果某类对象经常需要手填字段，应写自定义 prefab 和属性面板。
8. **用 Heaps 示例项目学习资源管理。** 先看无资源示例理解主循环，再看带资源示例学习 `res`、resource loader 与数据驱动。

## 8. 源码阅读中发现的潜在 bug / 疑点

> 以下问题是静态阅读发现的高置信度或中等置信度疑点，尚未运行 Hide 复现。仓库同时包含 JS/NW.js 编辑器与较新的 HL/HUI 编辑器代码；下列问题分别标注对应路径。

1. **图像视图在 2D camera 分支可能使用了错误的高度值。**
   - 位置：[hide_js/hide/view/Image.hx:443-446](hide_js/hide/view/Image.hx#L443-L446)
   - 现象：`cam2d.curPos.set(bmp.tile.width / 2, bmp.tile.width / 2, ...)` 第二个参数也使用了 width，非正方形贴图可能导致 Y 方向居中错误。
   - 建议：第二个参数应检查是否应为 `bmp.tile.height / 2`。

2. **NW.js open 事件处理函数内部重复注册自身。**
   - 位置：[hide_js/hide/Ide.hx:207-233](hide_js/hide/Ide.hx#L207-L233)
   - 现象：`nw.App.on("open", onOpen)` 在 `onOpen()` 内部再次调用，外部又注册一次。多次 open 后可能积累重复 handler，导致同一文件/URI 被重复处理。
   - 建议：确认是否需要内部注册；若无特殊原因，应只注册一次。

3. **CDB category tab 点击索引可能不正确。**
   - 位置：[hide_js/hide/view/CdbTable.hx:258-264](hide_js/hide/view/CdbTable.hx#L258-L264)
   - 现象：当点击 category tab 时，代码使用 `allCats[index % sheets.length]`。如果 sheet 数与 category 数不同，可能取到错误 category；更直观的索引通常应为 `index - sheets.length`。
   - 建议：添加包含多个 sheet/category 的用例验证。

4. **FileBrowser 局部 `getRoots()` 用字符串 contains 判断父子路径，存在误判风险。**
   - 位置：[hide_js/hide/view/FileBrowser.hx:1087-1115](hide_js/hide/view/FileBrowser.hx#L1087-L1115)
   - 现象：`StringTools.contains(file, dir2)` 不是路径边界判断，`foo/bar2` 可能被误判为包含在 `foo/bar` 下。不过当前右键 Move/Delete 路径使用的是 `FileManager.inst.getRoots(...)`，需要确认这个局部函数是否仍被调用。
   - 建议：若仍有调用，应改为规范化路径后按 `file == dir || file.startsWith(dir + "/")` 判断。

5. **HL 模型碰撞设置加载时循环变量用错，可能导致碰撞配置无法编辑或保存异常。**
   - 位置：[hide_hl/hide/view/Model.hx](hide_hl/hide/view/Model.hx)
   - 现象：加载 collision settings 时遍历 `obj.getMeshes()`，但循环体内 downcast 的是外层 `obj` 而不是循环变量 `o`。如果模型根节点不是 mesh，每个实际 mesh 的 `HMDModel` 可能都无法读取。
   - 影响：已有碰撞设置不显示、编辑面板异常、保存时配置丢失或覆盖。
   - 建议：将循环体内 downcast 目标改为当前 mesh 变量并补充模型根为容器节点的测试。

6. **HL 纹理压缩预览失败后仍读取临时 DDS，可能显示旧预览或直接报错。**
   - 位置：[hide_hl/hide/view/Texture.hx](hide_hl/hide/view/Texture.hx)
   - 现象：压缩转换失败时只调用错误回调，但后续仍继续读取 `%TEMP%/tempTexture.dds`。如果临时文件不存在会报错；如果存在上一次成功生成的旧 DDS，会显示陈旧预览。
   - 建议：转换失败后立即 return，或显式回退到原始纹理路径。

7. **HL/HUI 虚拟网格在窄宽度或未布局时可能除以 0。**
   - 位置：[hrt/ui/HuiVirtualGrid.hx](hrt/ui/HuiVirtualGrid.hx)
   - 现象：`itemsPerRow = floor(innerWidth / itemBaseWidth)` 没有保证至少为 1。控件未完成布局、宽度为 0、分栏过窄时可能出现 `items.length / itemsPerRow` 除以 0 或异常索引。
   - 建议：`itemsPerRow = max(1, floor(...))`，并为未布局状态延迟刷新。

8. **HL/HUI File Browser 的 Gallery/分栏模式可能未完整接线。**
   - 位置：[hrt/ui/HuiFileBrowser.hx](hrt/ui/HuiFileBrowser.hx)、[hide_hl/hide/view/FileBrowser.hx](hide_hl/hide/view/FileBrowser.hx)
   - 现象：代码创建了 `HuiVirtualGrid<File>`，但需要确认是否配置了 `generateItem`、`setItems`、双击打开、选择、拖放、右键菜单和 refresh 数据源。上层菜单允许切换 Gallery/Horizontal/Vertical，可能进入空白或不可用状态。
   - 建议：补齐 gallery 数据绑定与交互，或在未完成前隐藏对应布局选项。

9. **HL 启动时自动打开最近项目缺少路径校验。**
   - 位置：[hide_hl/hide/App.hx](hide_hl/hide/App.hx)、[hide/tools/IdeData.hx](hide/tools/IdeData.hx)
   - 现象：启动时直接取 `recentProjects[0]` 调用 `setProject`。如果目录被删除、移动、磁盘断开或权限变化，可能启动失败或进入坏状态。
   - 建议：启动时校验路径存在且可访问；失败时从 recent 列表跳过并回退到当前工作目录或项目选择界面。

10. **HL Prefab 备份目录只创建一级，嵌套资源可能备份失败。**
   - 位置：[hide_hl/hide/view/Prefab.hx](hide_hl/hide/view/Prefab.hx)
   - 现象：备份到 `res/.tmp/<嵌套路径>` 时只调用一次 `createDirectory(tmpDir)`。多级父目录不存在时可能抛错，导致“Backup save failed”。
   - 建议：使用递归目录创建或逐级创建父目录；明确提示“主文件已保存但备份失败”。

11. **HL/HUI 创建文件失败时提示写成 “create directory”。**
   - 位置：[hrt/ui/HuiFileBrowser.hx](hrt/ui/HuiFileBrowser.hx)
   - 现象：`createNewFile()` 中 `saveContent` 失败时提示 `Couldn't create directory`，会误导用户以为目录创建失败。
   - 建议：改为 `Couldn't create file`，并显示目标文件路径。

## 9. 参考入口清单

- 项目说明：[README.md](README.md)
- 默认配置：[bin/defaultProps.json](bin/defaultProps.json)
- 主菜单：[bin/app.html:67-146](bin/app.html#L67-L146)
- IDE 主类：[hide_js/hide/Ide.hx](hide_js/hide/Ide.hx)
- 扩展名注册：[hide_js/hide/Extension.hx](hide_js/hide/Extension.hx)
- 文件浏览器：[hide_js/hide/view/FileBrowser.hx](hide_js/hide/view/FileBrowser.hx)
- 文件视图基类：[hide_js/hide/view/FileView.hx](hide_js/hide/view/FileView.hx)
- Prefab 视图：[hide_js/hide/view/Prefab.hx](hide_js/hide/view/Prefab.hx)
- SceneEditor：[hide_js/hide/comp/SceneEditor.hx](hide_js/hide/comp/SceneEditor.hx)
- Prefab 数据模型：[hrt/prefab/Prefab.hx](hrt/prefab/Prefab.hx)
- Prefab 资源加载：[hrt/prefab/Resource.hx](hrt/prefab/Resource.hx)
- 3D 对象：[hrt/prefab/Object3D.hx](hrt/prefab/Object3D.hx)
- 2D 对象：[hrt/prefab/Object2D.hx](hrt/prefab/Object2D.hx)
- 材质：[hrt/prefab/Material.hx](hrt/prefab/Material.hx)
- 灯光：[hrt/prefab/Light.hx](hrt/prefab/Light.hx)
- Terrain：[hrt/prefab/terrain/Terrain.hx](hrt/prefab/terrain/Terrain.hx)
- FX：[hrt/prefab/fx/FX.hx](hrt/prefab/fx/FX.hx)
- 2D 粒子视图：[hide_js/hide/view/Particles2D.hx](hide_js/hide/view/Particles2D.hx)
- 3D 粒子视图：[hide_js/hide/view/Particles3D.hx](hide_js/hide/view/Particles3D.hx)
- 图像视图：[hide_js/hide/view/Image.hx](hide_js/hide/view/Image.hx)
- 模型视图：[hide_js/hide/view/Model.hx](hide_js/hide/view/Model.hx)
- CDB 视图：[hide_js/hide/view/CdbTable.hx](hide_js/hide/view/CdbTable.hx)
- Script 视图：[hide_js/hide/view/Script.hx](hide_js/hide/view/Script.hx)
- Shader Graph：[hide_js/hide/view/shadereditor/ShaderEditor.hx](hide_js/hide/view/shadereditor/ShaderEditor.hx)
- Texture Graph：[hide_js/hide/view/textureeditor/TextureEditor.hx](hide_js/hide/view/textureeditor/TextureEditor.hx)
- Animation Graph：[hide_js/hide/view/animgraph/AnimGraphEditor.hx](hide_js/hide/view/animgraph/AnimGraphEditor.hx)
- Blend Space 2D：[hide_js/hide/view/animgraph/BlendSpace2DEditor.hx](hide_js/hide/view/animgraph/BlendSpace2DEditor.hx)
