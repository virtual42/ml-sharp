# SHARP 项目介绍与本地安装配置

## 这个项目是做什么的？

SHARP 是 Apple 发布的单张照片新视角合成研究项目。它利用神经网络从一张 RGB 照片预测场景的 **3D 高斯表示（3D Gaussian Splatting，3DGS）**，再从原始相机附近的新位置渲染画面。可以用它制作照片的立体视差效果、轻微推进或横移的视频，以及可在兼容查看器中浏览的三维场景。

处理流程：**照片 → 深度与空间结构估计 → 3D 高斯场景 → 新视角画面或运镜视频**。

| 能力 | 输入与输出 | 条件或边界 |
| --- | --- | --- |
| 单图场景预测 | 单张照片 → `.ply` 3DGS 场景 | 不需要围绕场景拍摄多张照片 |
| 批量处理 | 图片文件夹 → 每张图片对应的场景 | 每张图片独立处理，不自动合并成一个场景 |
| 新视角视频 | 照片或已有 `.ply` → `.mp4` | 仓库内置渲染需要 CUDA GPU |
| 本地推理 | 下载好的模型 + 本地图片 | 预测支持 CPU、CUDA、MPS；速度依设备而异 |
| 外部查看 | `.ply` → 兼容 3DGS 的查看器 | 需要留意坐标方向、缩放及场景中心 |

输出以三维高斯表示场景，主要用于图像渲染。它不是直接生成可编辑的 CAD 实体或完整的网格模型；单张照片没有拍到的背面、遮挡区域及大幅绕拍效果不能保证准确。README 中的“不到一秒”指特定 GPU 条件下的模型预测，不等于下载、加载和视频渲染的总耗时。以上能力以[仓库说明](../README.md)和当前 CLI 实现为依据。

## 当前状态与执行约定

**状态：文档准备完成；尚未为本项目创建 Conda 环境、安装依赖、下载模型或执行推理/渲染。**

- 文档日期：2026-10-14。
- 用户要求：先记录配置方案，等待后续明确指令再开始安装或运行。
- 下文命令是待执行步骤，不能视作已经完成的操作。
- 后续每次实际配置或排错，都应更新本文的验证清单和变更记录。
- GitHub 只同步可以公开的代码、配置模板和文档；个人输入及生成结果留在本地。

## 环境方案

主线为 **Windows + 独立 Conda 环境 + NVIDIA CUDA**。Conda 管理 Python 和环境隔离，`pip` 在该环境内安装 SHARP 依赖。

| 组件 | 计划版本/选择 | 依据与验证状态 |
| --- | --- | --- |
| Conda 环境 | `sharp` | 计划新建；不修改 `base` 的项目依赖 |
| Python | 3.13 | `.python-version` 和 README 推荐；未创建 |
| PyTorch | 2.8.0，CUDA 12.8 构建 | 对应仓库锁定版本；未安装 |
| torchvision | 0.23.0 | 对应 `requirements.txt`；未安装 |
| gsplat | 1.5.3 | 对应 `requirements.txt`；Windows 编译/渲染待验证 |
| 其他依赖 | `requirements.txt` | 当前仓库的可复现基线 |

PyTorch 官方提供 Windows 的上述 CUDA 12.8 安装组合，CUDA 12.8 构建支持 Blackwell 架构，适用于 RTX 50 系列的版本选择。参见 [PyTorch 旧版本安装说明](https://pytorch.org/get-started/previous-versions/)和 [PyTorch Blackwell 支持说明](https://pytorch.org/blog/pytorch-2-7/)。整个组合在本项目上的实际推理和渲染仍待验证。

目前仅做过只读检查：`python -m conda --version` 和环境列表查询可用，直接调用 `conda.exe` 曾报错。故障原因尚未确认，不能据此断言 Conda 损坏，也不能把查询成功等同于环境创建/运行成功。后续先验证终端中的启动方式，再决定是否需要修复。

## 本地数据与隐私目录

优先把个人照片和结果放在仓库外。如果需要统一使用相对路径，可以采用下列本地目录；整个 `local-data/` 已被 Git 忽略。

```text
local-data/
  inputs/          # 照片，仅本地
  outputs/         # 3DGS 场景，仅本地
  renderings/      # 视频，仅本地
  checkpoints/     # 手动下载的模型
  cache/           # 模型及 CUDA 扩展缓存
  logs/            # 未脱敏的诊断日志
```

场景文件、深度图和渲染视频也可能还原个人照片中的信息，应和原始照片采用相同的保护方式。公开文档使用相对路径；不复制个人用户名、邮箱、真实照片名、EXIF/GPS、令牌或完整日志。

`.gitignore` 还覆盖常见图片、RAW、视频、音频、场景和权重扩展名，包含大小写变体。它不会让已跟踪文件自动停止跟踪，也不能识别任意格式的私人内容。仓库自带的公开演示图片 `data/teaser.jpg` 继续保留。

## 安装步骤（等待明确指令后执行）

### 1. 检查 Conda 和显卡

在 PowerShell 或已初始化的 Conda 终端中进入仓库根目录，先运行只读检查：

```powershell
conda --version
conda env list
nvidia-smi
```

如果直接调用 Conda 报错，在带有 Conda 模块的基础 Python 中尝试：

```powershell
python -m conda --version
python -m conda env list
```

若已有 `sharp` 环境，先查看其用途和依赖，不覆盖、删除或直接复用未知环境。`nvidia-smi` 的 CUDA 字段表示驱动支持能力，不证明本机已安装 `nvcc` 或完整 CUDA Toolkit。

### 2. 创建并验证独立环境

```powershell
conda create -n sharp python=3.13 pip
conda activate sharp
python --version
python -m pip --version
```

确认 Python 是 3.13，且 Python/pip 路径属于 `sharp` 后再继续。不要在未激活的 `base` 中执行下面的安装命令。

若无法使用直接启动器，可在基础环境中尝试模块入口：

```powershell
python -m conda create -n sharp python=3.13 pip
python -m conda run -n sharp --no-capture-output python --version
```

这条备用路线尚未验证。只有 `run` 检查成功后，才把后续命令以 `python -m conda run -n sharp --no-capture-output` 为前缀运行；不把该前缀当作当前终端已经激活的标志。参见 [Conda run 文档](https://docs.conda.io/projects/conda/en/stable/commands/run.html)。

### 3. 安装匹配的 GPU 依赖

以下命令默认已经激活 `sharp`，并位于仓库根目录：

```powershell
python -m pip install --upgrade pip
python -m pip install torch==2.8.0 torchvision==0.23.0 --index-url https://download.pytorch.org/whl/cu128
python -m pip install -r requirements.txt
python -m pip check
```

先安装指定的 PyTorch GPU 构建，再安装仓库依赖。安装后必须核实实际构建没有变成 CPU 版或其他 CUDA 版本；出现依赖冲突时先记录具体错误及版本，不通过整体升级所有包跳过问题。

### 4. 验证命令和 GPU 计算

```powershell
sharp --help
python -c "import torch; print('torch:', torch.__version__); print('CUDA runtime:', torch.version.cuda); print('CUDA available:', torch.cuda.is_available()); assert torch.cuda.is_available(); print('GPU:', torch.cuda.get_device_name(0)); x=torch.randn((32,32),device='cuda'); print('GPU result:', (x@x).mean().item())"
```

预期能显示 CLI 帮助，PyTorch 为计划版本且使用 CUDA 12.8，并能实际完成 GPU 运算。导入成功或 `torch.cuda.is_available()` 返回 `True` 都不足以单独证明 SHARP 可运行；下一阶段还要做图片推理和渲染验证。

### 5. 准备 gsplat 视频渲染

根据 [gsplat 官方 Windows 指南](https://github.com/nerfstudio-project/gsplat/blob/main/docs/INSTALL_WIN.md)，有些包需要在安装或首次使用时编译 CUDA 扩展。后续应检查是否存在匹配 **Python、PyTorch、CUDA、Windows** 的预编译包；不能假设任意 wheel 都包含可用的 CUDA 二进制。

如需本地编译，应准备兼容的 CUDA Toolkit 和 Visual Studio C++ Build Tools，并在正确加载编译环境的终端中检查：

```powershell
nvcc --version
where.exe cl
```

优先使用与 PyTorch 构建匹配的 CUDA Toolkit 12.8，具体 MSVC 工具集兼容性在安装时核对。PyTorch wheel 内的运行库不能替代编译器。使用 Visual Studio 的开发者终端加载 C++ 环境后再激活 Conda；不要假设在普通 PowerShell 中运行一个 `.bat` 就会把其环境变量保留到父终端。

本阶段尚未安装或验证工具链。推理通过、`import gsplat` 成功和实际视频渲染通过必须分别记录。

## 首次运行与以后启动（等待明确指令后执行）

### 1. 激活环境并准备本地路径

每次新开终端，需要重新激活环境及设置当前会话的缓存路径。下面的环境变量只影响当前终端：

```powershell
conda activate sharp
New-Item -ItemType Directory -Force -Path .\local-data\inputs, .\local-data\outputs, .\local-data\renderings, .\local-data\checkpoints, .\local-data\cache, .\local-data\logs
$env:TORCH_HOME = Join-Path (Get-Location) 'local-data\cache\torch'
$env:TORCH_EXTENSIONS_DIR = Join-Path (Get-Location) 'local-data\cache\torch-extensions'
```

### 2. 先验证场景生成

首次验证使用用户选定的非敏感图片；将示例中的 `sample.jpg` 替换为本地文件名，不把文件名写入公开记录。

```powershell
sharp predict -i .\local-data\inputs\sample.jpg -o .\local-data\outputs --device cuda
```

没有指定 `-c` 时，代码会从 Apple 的模型地址加载默认权重，并由 PyTorch 下载/缓存。上述 `TORCH_HOME` 设置将缓存放到忽略目录下；不设置时，通常位于用户目录的 `.cache/torch/hub/checkpoints/`。首次下载和后续加载耗时应与推理耗时分开记录。

使用已手动下载的模型可以指定路径：

```powershell
sharp predict -i .\local-data\inputs\sample.jpg -o .\local-data\outputs -c .\local-data\checkpoints\sharp_2572gikvuh.pt --device cuda
```

验收：命令正常结束、生成非空 `.ply`，再用本地兼容工具检查。首次联网安装/下载不代表需要上传照片；本地处理时不得把输入或结果交给云端查看服务。依赖和权重齐备后的离线运行仍需单独实测。

### 3. 再验证视频渲染

```powershell
sharp render -i .\local-data\outputs -o .\local-data\renderings
```

也可以在预测时加 `--render`：

```powershell
sharp predict -i .\local-data\inputs\sample.jpg -o .\local-data\outputs --device cuda --render
```

验收：生成可播放的 `.mp4`，画面有预期的视角运动；记录耗时、峰值显存和画面异常的文字摘要。首次 CUDA 扩展编译时间不计入稳定推理基准。不同子目录中的输入避免使用相同文件主名，因为当前 CLI 的输出按主名命名，可能覆盖。

## 常见问题与处理边界

| 情况 | 后续排查方式 |
| --- | --- |
| `conda.exe` 启动异常 | 检查实际终端、启动器及模块入口；目前仅确认模块查询可用，原因未定 |
| Python 或 pip 指向错误环境 | 停止安装，重新确认 `sharp` 激活及解释器路径 |
| CUDA 不可用或 GPU 运算报错 | 核实驱动、PyTorch CUDA 构建及 GPU 架构支持；记录脱敏错误 |
| gsplat 编译失败 | 核实 `nvcc`、MSVC、终端环境及版本组合；保留本地原始日志 |
| 推理成功、视频失败 | 单独定位渲染工具链；不把整个安装标记为完成 |
| 显存不足 | 记录失败阶段和资源占用；不能仅凭显卡型号承诺容量足够 |
| 模型下载失败 | 检查网络及官方来源；可用本地权重配合 `-c`，不在文档中保存代理凭据 |

## GitHub 同步与隐私检查

每轮完成可公开改动后，检查并同步到用户的 fork。不要上传个人输入、生成的三维数据/视频、模型、环境目录、未经脱敏的日志或 Obsidian 笔记。

```powershell
git remote -v
git status --short
git diff --check
git diff
```

逐个添加本轮审查通过的代码/文档文件，避免 `git add .`。本轮文档工作的暂存示例为：

```powershell
git add -- .gitignore AGENTS.md README.md docs/INSTALL.zh-CN.md
git diff --cached --name-status
git diff --cached
```

提交前再确认暂存区没有私人材料、内嵌图片、凭据或个人路径，作者及提交者使用 GitHub 隐私邮箱。推送前检查远程地址和待推送提交，完成后核实远程分支哈希。仓库是否公开不改变上述限制；不使用强制推送绕过冲突。

## 验证清单

- [x] 已阅读当前仓库说明、CLI 和固定依赖版本。
- [x] 已确认 Conda 模块可查询版本与环境列表。
- [x] 已编写配置方案和隐私忽略规则。
- [ ] 获得本次环境安装/运行的明确指令。
- [ ] 创建并核实独立 `sharp` 环境。
- [ ] 安装依赖且 `pip check` 通过。
- [ ] `sharp --help` 与实际 GPU 运算通过。
- [ ] 权重下载/本地加载通过。
- [ ] 非敏感图片生成 `.ply` 通过。
- [ ] gsplat 实际渲染及 `.mp4` 播放通过。
- [ ] 依赖与权重齐备后的离线运行通过。

## 配置变更记录

| 日期 | 阶段 | 实际操作与结果 | 未完成事项 |
| --- | --- | --- | --- |
| 2026-10-14 | 文档准备 | 读取仓库及依赖；完成 Conda 只读查询；编写本指南、协作约定和隐私忽略规则 | 等待明确指令；未创建环境、安装依赖、下载权重或运行模型 |

后续每次更新应记录实际 Python/PyTorch/CUDA/gsplat 版本、脱敏后的操作、成功或失败证据、解决办法及剩余问题。不要用预期版本覆盖真实结果，不把查询成功写成安装成功。

## 参考资料

- [上游项目](https://github.com/apple/ml-sharp)与[项目展示页](https://apple.github.io/ml-sharp/)
- [当前仓库 README](../README.md)、[依赖版本](../requirements.txt)、[预测实现](../src/sharp/cli/predict.py)、[渲染实现](../src/sharp/cli/render.py)
- [PyTorch 官方安装组合](https://pytorch.org/get-started/previous-versions/)
- [gsplat 官方 Windows 安装说明](https://github.com/nerfstudio-project/gsplat/blob/main/docs/INSTALL_WIN.md)
- [Conda run 文档](https://docs.conda.io/projects/conda/en/stable/commands/run.html)
- [代码许可](../LICENSE)与[模型许可](../LICENSE_MODEL)
