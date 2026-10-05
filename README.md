# dll-build

用 GitHub Actions 在 Windows x64（MSVC）上自动编译最新 64 位 Windows DLL 全家桶。手动触发一次，构建产物会打包成 artifact，并自动发布 GitHub Release（正文含各库编译版本表）。

仓库本身不含任何源码，所有上游（zlib / zstd / freetype / vcpkg / MemProcFS / LeechCore）都在构建时实时拉取最新版。

## 产物清单（dlls-x64）

| 产物 | 说明 | 版本来源 |
|---|---|---|
| `zlib1.dll` | zlib（官方分发名；CMake 输出 z.dll，构建后改名） | 最新 tag |
| `zstd.dll` | zstd | 最新 tag |
| `boost_random-*.dll` | Boost.Random | vcpkg 最新 |
| `embree4.dll` | Embree（运行时依赖 `tbb12.dll`，包内已配齐） | vcpkg 最新 |
| `tbb12.dll` | oneTBB | vcpkg 最新 |
| `freetype.dll` | FreeType **静态依赖版**：zlib/libpng/bzip2/brotli 全部内嵌，单文件即可用 | 最新 tag |
| `vmm.dll` | MemProcFS（运行时依赖 `leechcore.dll`，包内已配齐） | master |
| `leechcore.dll` | LeechCore | master |

配套附带：各库 `.lib` 导入库、zlib/zstd 头文件（`include/`）。

## 使用方法

1. 把 `.github/workflows/build-dlls.yml` 放进本仓库并 push；
2. 打开仓库 **Actions** 页签 → 左侧选中 **Build Windows x64 DLLs** → **Run workflow**（手动触发，不会自动跑）；
3. 跑完后两种方式拿产物：
   - run 页面底部 **Artifacts** → 下载 `dlls-x64`；
   - 仓库 **Releases** 页签 → 最新 release 的 `dlls-x64.zip`。

## Release 说明

每次构建成功自动发布 Release：

- tag / 标题：`dlls-<时间戳>`（如 `dlls-20261005-153000`，重复运行不冲突）；
- 正文自动生成**版本表**，显示每个库本次实际编译的版本：
  - zlib / zstd / freetype：取到的 tag（freetype 显示为 `2.14.3` 形式）
  - boost / embree / tbb：vcpkg 实际安装版本
  - vmm / leechcore：master 短 commit hash

## 设计要点

- **全部取最新**：zlib/zstd/freetype 每次运行用 `git ls-remote` 解析最新 tag（不走 GitHub API，避免匿名限流）；vcpkg 更新 registry；vmm/leechcore 直接 clone master；
- **仅手动触发**（`workflow_dispatch`），不会在 push 时自动跑；
- **vcpkg 只编 Release**：自定义 triplet 跳过 debug 构建，时间约省一半；`installed + downloads` 走 Actions 缓存，registry 没更新时秒过；
- **freetype 静态依赖版**：4 个依赖（zlib/libpng/bzip2/brotli）以静态库链进 freetype.dll，产物只有单个文件；
- **内置校验**：zlib/zstd 做 PE 架构 + ctypes 加载 + 版本断言；collect 后对全部 8 个 DLL 做 PE=x64 检查；
- **显式失败**：clone / 补丁下载应用 / vcpkg install / 构建任一环节失败都会立刻报错，不会静默产出错误产物。

## 目录结构

```
.github/workflows/build-dlls.yml   # 唯一的 workflow 文件（全部逻辑）
```

## 权限要求

仓库 **Settings → Actions → General → Workflow permissions** 需允许 workflow 写权限（workflow 文件内已声明 `contents: write`；若仓库默认设为只读，文件内声明仍生效，但改成 "Read and write permissions" 更省心）。修改设置后需**重新触发**一次 run 才生效。

## 用途说明

`vmm.dll` / `leechcore.dll` 属于 PCILeech / MemProcFS 开源内存取证工具链（DMA 内存读取、取证分析等合法场景）。请勿在在线游戏中使用 DMA / 外设作弊工具——账号和设备风险都很高，本仓库仅用于软件构建技术目的。
