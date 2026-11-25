# ComfyUI 本地环境与核心性能记录

本仓库记录本机 ComfyUI 便携版的环境信息、核心版本升级过程和性能实测数据。
模型、自定义节点、内嵌 Python、输出文件等大体积内容不入库，恢复方式见 [第 8 节](#8-仓库说明与恢复)。

---

## 1. 测试环境

| 项目 | 配置 |
|---|---|
| 操作系统 | Windows 10 Enterprise 19045 |
| GPU | NVIDIA RTX 3070 8GB（compute capability 8.6） |
| 内存 | 64GB |
| Python | 3.12.7（python_embeded） |
| ComfyUI | v0.46.2（2026-11-20） |
| PyTorch | 2.9.1+cu130 |
| NVIDIA 驱动 | 621.24 |
| 模型存储 | 3TB 机械盘（WDC WD30EFRX），182.4GB / 67 个文件 |
| 页面文件 | 32–64GB（D:） |
| 启动参数 | `--windows-standalone-build --reserve-vram 0.8` |

---

## 2. 核心升级记录（v0.3.9 → v0.46.2）

| 组件 | 升级前 | 升级后 |
|---|---|---|
| ComfyUI 核心 | v0.3.9-10-g57f330c（2024-12-22） | v0.46.2（2026-11-20） |
| PyTorch | 2.5.1+cu124 | 2.9.1+cu130 |
| torchvision / torchaudio | 0.20.1 / 2.5.1 | 0.24.1 / 2.9.1（cu130） |
| NVIDIA 驱动 | 561.17（2024-10-30） | 621.24（2026-11-07） |
| CUDA 运行时 | 12.4 | 13.0 |
| ComfyUI-Manager | V2.55.5 | V3.43.2 |
| 自定义节点 | 13 个（2024-10 ~ 2025-01） | 全部更新到最新 |
| 回滚分支 | — | `backup_branch_2026-11-23_19_26_12` |

核心性能相关的变更与处理：

1. **旧核心与新 PyTorch 不兼容**：启动时报 `comfy_kitchen` 算子签名 `stride: list[int]` 不受支持，升级 PyTorch 到 2.9.1 后解决。
2. **CUDA 13 运行时**：新核心提示 `PYTORCH CUDA VERSION BELOW 13.0 ... Performance will be terrible`，先升级驱动到 621.24，再离线安装 cu130 wheel，警告消失。
3. **依赖重建**：新增 comfyui-frontend-package 1.59.6、alembic、SQLAlchemy、av、blake3、comfy-kitchen 等；为兼容节点回退 numpy 1.26.4、protobuf 4.25.8、scipy 1.14.1、pydantic 2.10.4。
4. **DWPose 加速**：安装 onnxruntime-gpu 1.32.0，从 CPU 推理切到 CUDA。
5. **BizyAir 禁用**：避免启动 502 报错，启动更干净。
6. **验证结果**：全量启动无 `IMPORT FAILED`，注册 2425 个节点类；图像与视频工作流均出图/出片成功。

---

## 3. 核心性能基准（暖态 vs 冷启动）

| 测试 | 冷启动（含加载） | 暖态 | 显存峰值 |
|---|---|---|---|
| SD1.5，512×512，20 步 | 40.0s | 4.0s | 3.09GB |
| Flux fp8，1024×1024，8 步 | 164.5s | 23.3s | 7.43GB |
| Hunyuan Video，512×320，33 帧，20 步 | 281.8s | 118.8s | 7.90GB |

结论：

- 暖态算力发挥正常，常用模型（如 Flux）可以一次加载持续复用。
- 冷启动 93% 的时间消耗在机械盘读取（Flux 约 16.5GB、Hunyuan 约 32GB），与 100MB/s 的盘速吻合。
- 视频测试中步时从 0.49s 递增到 4.51s，是 8GB 显存顶满后权重换入换出的表现，不是 GPU 降频。

---

## 4. 批量生成实测（SD1.5 512×512 / 20 步）

| 模式 | 总耗时 | 单张耗时 | 吞吐（张/秒） | 相对串行 | 显存峰值 |
|---|---|---|---|---|---|
| 串行（4 个不同 prompt） | 15.95s | 3.99s | 0.251 | 1.00× | 3539MB |
| 排队（4 个不同 prompt） | 15.72s | 3.93s | 0.254 | 1.01× | 3059MB |
| batch=4（同 prompt） | 7.56s | 1.89s | 0.529 | 2.11× | 5195MB |
| batch=8（同 prompt） | 12.97s | 1.62s | 0.617 | 2.46× | 6961MB |

结论：

- 同 prompt 批量变体用 `batch_size` 提升最明显：batch 4 约 2.11 倍、batch 8 约 2.46 倍吞吐。
- 不同 prompt 仅靠排队提交几乎没有提速（1.4%），瓶颈在 GPU 计算而不是客户端等待。
- 8GB 显存建议 batch 4–6；batch 8 峰值 6.96GB，叠加桌面程序容易 OOM。

---

## 5. 常驻服务关键配置

当前启动脚本 [run_nvidia_gpu.bat](run_nvidia_gpu.bat)：

```bat
.\python_embeded\python.exe -s ComfyUI\main.py --windows-standalone-build --reserve-vram 0.8
```

服务化建议追加（已验证参数存在且可用）：

| 参数 | 作用 |
|---|---|
| `--disable-auto-launch` | 不自动打开浏览器 |
| `--vram-headroom 0.5` | 额外保留显存，缓解视频步时递增 |
| `--high-ram` | 利用 64GB 内存减少模型重载 |
| `--cache-lru 8` | 限制节点结果缓存，防止长期运行内存膨胀 |
| `--disable-assets` | 关闭后台资产扫描，减少常驻 IO |

系统层：

- 页面文件 32–64GB：避免大模型加载时分配失败。
- `TdrDelay = 60`：减少长任务触发驱动重置。
- 服务启动后先跑一个最小任务预热（图像 512×512 / 1 步；视频 256×256 / 5 帧 / 1 步）。

不推荐：`--highvram`、`--gpu-only`（8GB 会 OOM）、`--fast`（画质换速度）、`torch.compile`（Windows 收益不确定）。

---

## 6. 性能瓶颈与后续优化

| 优化项 | 预期收益 | 状态 |
|---|---|---|
| 模型迁移 NVMe + `--fast-disk` | 冷启动 5–10×（收益最大） | 待实施 |
| Hunyuan 换 fp8 模型（约 12GB） | 加载时间减半、显存压力下降 | 待下载 |
| SageAttention（需实测） | 采样 20–40% | 待验证 |
| 电源计划改“高性能” | 小幅、免费 | 未执行 |
| 每日凌晨重启 + 看护进程 | 消除长时间运行性能漂移 | 待实施 |

硬件建议：1TB NVMe 专放 models；12–16GB 显卡可显著减少动态卸载。

---

## 7. 文档索引

| 文档 | 内容 |
|---|---|
| [ComfyUI_系统升级与性能优化记录_20261124.md](ComfyUI_系统升级与性能优化记录_20261124.md) | 升级全过程、基准测试、视频测试、优化清单、回滚手册 |
| [ComfyUI_常驻服务优化方案_20261124.md](ComfyUI_常驻服务优化方案_20261124.md) | 常驻服务参数、缓存预热、请求层与运维优化 |
| [ComfyUI_批量生成性能测试_20261124.md](ComfyUI_批量生成性能测试_20261124.md) | 串行 / 排队 / batch 对比与批量策略 |
| [README_VERY_IMPORTANT.txt](README_VERY_IMPORTANT.txt) | 官方启动与更新说明 |

---

## 8. 仓库说明与恢复

### 8.1 仓库范围

本仓库只收录文档、启动/更新脚本与示例工作流。以下目录已被 `.gitignore` 排除：

- `ComfyUI/`：上游源码（自带独立 `.git`，指向 comfyanonymous/ComfyUI）、`models/`（182GB）、`custom_nodes/`（5.6GB）、`output/`、`user/` 等
- `python_embeded/`：内嵌 Python 与依赖（5.8GB）
- `tmp/`、`.screenshots/`、`*.safetensors`、`*.log`、`*.mp4`、`*.png`

### 8.2 恢复步骤

1. 下载 ComfyUI 便携版 v0.46.2 并解压到 `F:\ComfyUI`。
2. 按本文档第 2 节升级 PyTorch 到 2.9.1+cu130、驱动到 621.24。
3. 恢复 `models/`、`custom_nodes/` 与 `python_embeded/`（建议使用升级前的完整备份 `H:\ComfyUI_backup_20261123`）。
4. 用本仓库的启动脚本与示例工作流覆盖对应文件。
5. 按第 5 节调整启动参数、页面文件与 TdrDelay。

### 8.3 版本基线

ComfyUI v0.46.2 / PyTorch 2.9.1+cu130 / Manager V3.43.2 / 驱动 621.24 / onnxruntime-gpu 1.32.0 / numpy 1.26.4。
