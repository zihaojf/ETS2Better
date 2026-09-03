# ETS2Better DX11 Shader 与材质替换插件设计文档

## 1. 概述

ETS2Better 是一个面向《Euro Truck Simulator 2》DX11 渲染路径的原生 C++ 插件。插件通过 `dxgi.dll` 与 `dinput8.dll` 代理 DLL 注入进程，使用 MinHook hook D3D11/DXGI COM 虚函数，在运行时替换指定 shader、贴图与材质参数，并附加可调后处理效果。

首要目标：

1. 用配置文件精确命中 `effect.scs` 中指定 shader 或材质路径对应的资源。
2. 在 D3D11 调用层替换为预先编译好的 DXBC/CSO 文件。
3. 支持不同游戏版本的独立配置包。
4. 提供外部 CLI 与游戏内 ImGui 调试工具，后期新增替换项无需修改源码。
5. 支持最终画面后处理参数调整。
6. 保持 DX11 COM 接口调用约定与虚函数签名一致，优先保证稳定性。

当前参考环境：

- 游戏：Euro Truck Simulator 2 `1.60.1.7`
- 渲染 API：Direct3D 11，目标 profile 以 `sm51` 为主
- 构建工具：CMake + Visual Studio 2022 x64
- 第三方库：仓库内 MinHook 1.3.4、Dear ImGui 1.92.7、待引入 `nlohmann/json`

已确认的设计方向：

- 替换层：混合方案，shader 在 D3D11 创建/设置层替换，材质与贴图用内容指纹辅助命中。
- 材质范围：同时支持贴图资源和材质参数。
- 调试工具：外部 CLI 与游戏内 ImGui 并存。
- 后处理：独立最终全屏 pass 与替换游戏原后处理 shader 并存。
- 旧 DLL：按全新实现处理，不复用游戏目录中已有的 `dxgi.dll` 和 `dinput8.dll`。
- 版本策略：v1 仅维护 `1.60.1.7` 对应的 `1.60` 配置，但配置结构和选择逻辑从第一版起支持多版本。
- Shader 来源：替换 shader 由项目维护的 HLSL 源码编译生成，调试工具负责编译与校验。
- 优先场景：先做天气、光照、天空、路面等环境效果，之后做全局后处理。

## 2. 总体架构

### 2.1 模块划分

工程建议生成以下目标：

| 目标 | 类型 | 职责 |
|---|---|---|
| `ets2better_core` | DLL | Hook 管理、配置管理、替换引擎、后处理、日志 |
| `dxgi` | 代理 DLL | 转发系统 `dxgi.dll` 导出，触发核心初始化 |
| `dinput8` | 代理 DLL | 转发系统 `dinput8.dll` 导出，作为备用注入入口 |
| `ets2better_tool` | CLI | 扫描归档、生成/校验/迁移配置、离线分析 |
| `ets2better_tests` | 测试 | Hash、SCS、DXBC 解析与离屏 D3D11 测试 |

两个代理 DLL 都只负责导出转发和一次性核心加载，不直接包含业务逻辑。`DllMain` 中不执行 MinHook 初始化、文件扫描、线程创建或其他可能触发 loader lock 的操作。

### 2.2 注入流程

1. 游戏从自身目录加载代理 `dxgi.dll` 或 `dinput8.dll`。
2. 代理 DLL 用绝对路径加载 `C:\Windows\System32\dxgi.dll` 或 `dinput8.dll`。
3. 在导出函数首次被调用时，通过一次性初始化例程加载 `ets2better_core.dll`。
4. 核心模块读取全局配置，识别游戏版本，选择版本包。
5. 等待 D3D11 设备或交换链对象创建后，再安装 D3D11/DXGI hook。

失败策略为 fail-open：

- 配置缺失或非法：禁用全部替换，仅保留日志。
- 游戏版本不匹配：禁用替换，提示用户选择版本包。
- 识别到 DX12 路径：v1 禁用核心替换逻辑，不尝试兼容。
- Hook 初始化失败：卸载自身 hook，继续调用原始 API。

## 3. Hook 设计

### 3.1 接口签名原则

所有 hook 都以 Windows SDK 的 `d3d11.h`、`dxgi.h` 声明为唯一签名来源。COM 虚函数 hook 的实现必须满足：

- 使用 `STDMETHODCALLTYPE`。
- 显式声明 `this` 指针为第一个参数。
- 参数类型、顺序、常量性与 SDK 完全一致。
- 不使用手工推导的虚表布局。
- Hook 安装前检查接口指针和目标 vfunc 是否有效。

示例形式：

```cpp
using CreatePixelShader_t = HRESULT(STDMETHODCALLTYPE*)(
    ID3D11Device* self,
    const void* pShaderBytecode,
    SIZE_T BytecodeLength,
    ID3D11ClassLinkage* pClassLinkage,
    ID3D11PixelShader** ppPixelShader);
```

### 3.2 需要 hook 的 D3D11 接口

核心替换层：

- `ID3D11Device::CreateVertexShader`
- `ID3D11Device::CreatePixelShader`
- `ID3D11Device::CreateGeometryShader`
- `ID3D11Device::CreateHullShader`
- `ID3D11Device::CreateDomainShader`
- `ID3D11Device::CreateComputeShader`

热替换层：

- `ID3D11DeviceContext::VSSetShader`
- `ID3D11DeviceContext::PSSetShader`
- `ID3D11DeviceContext::GSSetShader`
- `ID3D11DeviceContext::HSSetShader`
- `ID3D11DeviceContext::DSSetShader`
- `ID3D11DeviceContext::CSSetShader`

设备与状态记录：

- `ID3D11Device::CreateTexture2D`
- `ID3D11Device::CreateShaderResourceView`
- `ID3D11Device::CreateBuffer`
- `ID3D11DeviceContext::PSSetShaderResources`
- `ID3D11DeviceContext::UpdateSubresource`
- `ID3D11DeviceContext::Map`

交换链与覆盖层：

- `IDXGISwapChain::Present`
- `IDXGISwapChain::ResizeBuffers`
- `IDXGISwapChain::ResizeTarget`

### 3.3 Shader 替换流程

`Create*Shader` 的流程：

1. 计算原始 `pShaderBytecode` 的 SHA-256。
2. 在当前版本包的 shader 索引中查找该哈希。
3. 找到且配置启用时，读取替换 DXBC/CSO。
4. 校验替换文件是否为有效 DXBC。
5. 通过 `D3DReflect` 比较输入输出签名、constant buffer、SRV/UAV/sampler。
6. 校验通过后，用替换字节调用原始 `Create*Shader`。
7. 建立原始 shader 对象与替换对象之间的映射。
8. 校验失败或配置歧义时调用原始字节码，记录失败原因。

`*SetShader` 的流程：

1. 检查传入的原始 shader 指针是否存在替换映射。
2. 存在且启用时，改为设置替换后的 shader 指针。
3. 不存在映射时直接调用原始函数。
4. 热重载配置后，重建映射并继续使用已有替换 shader 对象。

### 3.4 贴图与材质替换

材质范围包含两类对象：

1. 贴图资源：DDS/TGA 等，通过内容哈希或资源指纹命中。
2. 材质参数：constant buffer、SRV 绑定和参数字段，通过 shader 哈希与绑定信息命中。

运行时 D3D11 API 不携带 SCS 虚拟路径，因此路径与哈希不能直接从 `CreateShaderResourceView` 获得。推荐方案：

- 离线扫描器为每个材质文件生成内容哈希和路径哈希。
- 调试模式记录实际创建的贴图尺寸、格式、mipmap、数据指纹和绑定调用。
- 用户在 ImGui 中选择候选贴图并确认路径。
- 生成配置时记录替换文件与资源匹配特征。

参数补丁通过以下字段定位：

```json
{
  "type": "material_parameter",
  "game_version": "1.60.1.7",
  "match": {
    "shader_sha256": "...",
    "constant_buffer": "PerFrame",
    "field": "exposure"
  },
  "value": 1.25,
  "enabled": true
}
```

## 4. SCS 归档与版本包

### 4.1 归档格式

当前 `effect.scs` 使用 HashFS v2：

- 魔数：`SCS#`
- 版本：`2`
- 哈希：`CITY`
- 条目数：约 24521
- 入口表偏移：约 `0x241BA0`
- 入口表大小：约 `0x1D4C0`
- 数据区起始：约 `0x3C41C60`

CLI 内置 HashFS v2/CityHash64 解析器，不依赖外部 `scs_extractor.exe`。路径哈希采用 SCS 的规范化规则，并必须用 `1.60.1.7` 归档实测校验。

### 4.2 `.vso` / `.fso` 处理

游戏 shader 文件以 `SHDO` 包装开头，不是裸 DXBC。维护工具需要：

1. 解析 `SHDO` 结构。
2. 提取内部 DXBC 字节码。
3. 计算原始 DXBC SHA-256。
4. 记录 shader 阶段与 profile。
5. 将信息写入版本包。

运行时替换文件必须是裸 DXBC/CSO。插件不复刻 `SHDO` 包装。

### 4.3 版本包结构

建议路径：

```text
config/
  settings.json
  packs/
    1.60.1.7.json
assets/
  hlsl/
    1.60.1.7/
      road/
  dxbc/
    1.60.1.7/
      road/
  textures/
    1.60.1.7/
```

版本包核心字段：

```json
{
  "schema_version": 1,
  "game_version": "1.60.1.7",
  "render_api": "d3d11",
  "profile": "sm51",
  "archive": {
    "name": "effect.scs",
    "hashfs_version": 2,
    "path_hash": "cityhash64_v2",
    "fingerprint": "..."
  },
  "shaders": [],
  "textures": [],
  "materials": [],
  "parameter_patches": [],
  "postprocess": []
}
```

版本选择优先级：

1. `settings.json` 强制指定版本包。
2. 游戏执行文件版本精确匹配。
3. 归档指纹匹配。
4. 前缀版本匹配，如 `1.60`。
5. 以上均失败时禁用替换。

版本维护策略：

- v1 只实际维护 `1.60.1.7` 对应的版本包。
- 配置 schema、包发现、版本选择和迁移接口从第一版起按多版本设计。
- 新游戏版本通过新增版本包接入，不要求修改核心源码。
- `1.60.x` 前缀匹配只作为候选策略；包内资源指纹必须匹配后才启用。

## 5. 调试工具设计

### 5.1 CLI 命令

| 命令 | 作用 |
|---|---|
| `scan` | 扫描 `.scs`、解包目录、`.rfx`、`.vso`、`.fso` 与材质文件 |
| `add` | 根据原始路径和替换文件生成配置项 |
| `validate` | 校验文件、DXBC、反射签名、路径哈希和配置完整性 |
| `migrate` | 将旧版本包按路径、哈希和 profile 迁移到新版本 |
| `dump` | 导出某个路径的原始 shader/材质信息 |
| `diagnose` | 合并游戏内日志，分析命中失败原因 |

### 5.2 ImGui 面板

建议包含：

- 替换项列表：路径、阶段、哈希、状态、失败原因。
- 实时命中统计：Create、Set、成功替换、跳过、失败。
- Shader 资源查看器：当前绑定的 VS/PS/CS 与资源。
- 贴图候选选择器：将运行时资源与离线扫描路径关联。
- 参数调试面板：曝光、对比度、饱和度、色温等。
- 配置热重载与导出。
- 日志过滤与复制。

### 5.3 ImGui 输入与鼠标锁定

- 主快捷键使用 `Insert`，用于打开/关闭 ImGui 调试面板。
- 面板打开时必须解除游戏鼠标锁定，并让鼠标输入进入调试 UI。
- 游戏在面板打开时继续运行，方便实时观察画面变化。
- 面板打开期间鼠标事件只交给 ImGui，不转发给游戏。
- 面板关闭时恢复游戏输入状态与鼠标锁定。
- 鼠标解锁失败时显示警告，不允许 UI 抢占输入导致游戏不可操作。
- 输入状态切换必须可重复执行，避免长时间运行或 Alt+Tab 后失效。

### 5.4 HLSL 调试编译

调试版本支持在 ImGui 中实时编辑 HLSL：

1. 编辑 HLSL 源码。
2. 调用本地 D3D 编译器生成 `sm51` DXBC。
3. 校验 DXBC 与替换对象的兼容性。
4. 热加载新的替换 shader。
5. 保存源码和编译产物到替换资产目录。

用户发布版本使用固定预编译 DXBC，不依赖运行时 HLSL 编辑和编译功能。

## 6. 后处理设计

后处理采用两种机制并存：

1. 独立最终全屏 pass：在 `Present` 前渲染到中间纹理，进行统一滤镜处理。
2. 原游戏 shader 替换：通过普通 shader 替换流程修改游戏自带后处理链。

v1 参数建议：

- 曝光
- 对比度
- 饱和度
- 色温
- LUT 强度
- 暗角
- 锐化
- 噪点强度

技术要求：

- 正确处理 sRGB 与 HDR 交换链格式。
- 支持窗口缩放后重建中间纹理。
- 不修改游戏原始 back buffer 内容。
- 参数值通过 constant buffer 传递，避免每帧重新编译 shader。

优先级：

1. 天气、光照、天空、路面与环境 shader。
2. 原游戏后处理 shader 替换。
3. 插件附加最终全屏 pass。

## 7. 并发与安全

线程模型：

- Hook 函数只做快速查表和必要的资源创建。
- 文件 IO、JSON 解析、归档扫描、DXBC 反射放在后台线程。
- 渲染线程与配置线程使用 SRWLock 或读写锁。
- 配置热重载采用不可变快照加原子指针切换。

稳定性要求：

- 所有 hook 遵守 fail-open。
- 无效配置不会阻止原始调用。
- 替换 shader 反射签名不匹配时默认拒绝加载。
- 缓存替换后的 COM 对象，避免重复创建。
- 代理 DLL 必须完整转发原导出，避免游戏或系统调用缺导出崩溃。
- 在异常路径中避免释放游戏持有的资源指针。

配置共享：

- 用户间共享使用 `.zip` 包。
- 包内包含版本包 JSON、HLSL/DXBC/贴图资产、依赖资产清单和校验哈希。
- 导入时先解压到临时目录，完整校验路径、文件哈希与 DXBC 结构，再原子安装。
- 校验失败时拒绝导入或禁用对应替换项，不影响现有配置。
- v1 不强制签名校验，但配置结构保留 `signature` 字段，后续可启用。

贴图资源：

- 允许使用高分辨率替换贴图。
- 创建资源失败或显存不足时保持原始资源并弹出行内警告。
- ImGui 面板显示替换贴图尺寸、格式、mipmap 数量和预估占用。
- 不自动降级或擅自缩放用户资产，由用户决定关闭替换或换用低分辨率版本。

## 8. 测试计划

单元测试：

- CityHash64 与 SCS 路径哈希
- 路径规范化大小写规则
- SHA-256
- `SHDO` 解包
- HashFS v2 头部与入口表解析
- JSON schema 与默认值
- 版本选择和迁移

离屏 D3D11 测试：

- 创建 WARP 设备
- 安装 hook
- 用测试 DXBC 验证 Create/Set 替换
- 验证签名不匹配时拒绝替换
- 验证交换链 resize 后覆盖层和后处理仍可用

游戏内验收：

0. 链路冒烟测试：先选择路面 shader，将其替换为纯红色或纯绿色，确认路径、哈希、Create/Set 替换和画面验证整条链路可用。
1. 全部替换关闭，插件无副作用。
2. 启用单个 shader，确认画面变化且无崩溃。
3. 启用多个 shader，确认调用统计正确。
4. 启用贴图替换。
5. 启用材质参数补丁。
6. 启用后处理并热调参。
7. 窗口缩放、Alt+Tab、切换画质、长时间运行。
8. 配置损坏、文件缺失、版本不匹配时 fail-open。

## 9. 已确认决策

以下决策已由用户确认，作为 v1 实现依据：

1. v1 暂时只维护 `1.60.1.7`，但架构必须支持多个版本配置。
2. 替换 shader 由项目维护的 HLSL 源码自行编译。
3. 第一优先级是天气、光照等环境效果，其次才是全局后处理。
4. ImGui 使用 `Insert` 键开关，必须正确处理游戏鼠标锁定与解锁。
5. 调试版本支持实时编辑 HLSL 并编译；用户版本使用固定预编译资源。
6. 后处理参数不需要按天气、时间、车内/车外自动切换。
7. 支持用户间共享配置和资产包，v1 不强制签名校验。
8. 允许高分辨率贴图；显存不足或创建失败时提示用户，不自动降级。
9. 目录结构使用 `config/packs/1.60.1.7.json` 与 `assets/hlsl`、`assets/dxbc`、`assets/textures` 分类存放。
10. 用户间共享包使用 `.zip` 格式，导入前完整校验并原子安装。
11. 第一阶段先用路面替换为红色或绿色验证完整替换链路，再扩展到天气、光照等正式效果。
12. ImGui 打开时游戏继续运行，但鼠标输入被阻断且不转发给游戏。
