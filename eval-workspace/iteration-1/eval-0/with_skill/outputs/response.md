以下是完整的从环境搭建到上证指数预测的操作流程。

## 完整步骤

### Step 0: 克隆仓库 + 环境初始化

```bash
cd ~
git clone https://github.com/shiyu-coder/Kronos.git
cd Kronos

python3 -m venv .venv
echo 'export HF_ENDPOINT=https://hf-mirror.com' >> .venv/bin/activate
source .venv/bin/activate
pip install -r requirements.txt
pip install pytest pytest-timeout
pip install -e tdxdata/
```

验证：
```bash
python -c "import torch; import numpy; import pandas; import tdxdata; print('依赖OK')"
python -c "import torch; print('CUDA:', torch.cuda.is_available())"
```

产物：`.venv/` 目录，约 2-3GB。

---

### Step 1: 数据导入

```bash
.venv/bin/python scripts/discover_stocks.py --output /tmp/tdx_all_stocks.txt

.venv/bin/python scripts/tdx_import.py \
  --symbol-file /tmp/tdx_all_stocks.txt \
  --dividend-type back \
  --periods 1d \
  --output-dir ./data/tdx_import \
  --train-range 2024-06-01 2025-10-31 \
  --val-range   2025-11-01 2026-01-31 \
  --test-range  2026-02-01 2026-04-30 \
  --no-continuity
```

验证产物：
| 文件 | 大小 |
|------|------|
| `train_data.pkl` | ~100MB |
| `val_data.pkl` | ~35MB |
| `test_data.pkl` | ~25MB |

---

### Step 2: 下载预训练模型

```bash
.venv/bin/python -c "
from huggingface_hub import hf_hub_download
for model_id in ['NeoQuasar/Kronos-Tokenizer-base', 'NeoQuasar/Kronos-base']:
    path = hf_hub_download(repo_id=model_id, filename='model.safetensors')
    print(f'{model_id}: {path}')
    hf_hub_download(repo_id=model_id, filename='config.json')
"
```

| 模型 | 文件大小 |
|------|----------|
| Kronos-Tokenizer-base | ~15.8MB |
| Kronos-base | ~409MB |

---

### Step 3: 微调训练

#### 3a. Tokenizer 微调

```bash
.venv/bin/python finetune/train_tokenizer_tdx.py \
  --data-dir ./data/tdx_import/1d \
  --epochs 30
```

- 显存占用：~5GB（fp32，batch_size=50）
- 训练时间：约 50-90 分钟（30 epochs）

验证产物：
| 文件 | 大小 |
|------|------|
| `model.safetensors` | ~15.1MB |

#### 3b. Predictor 微调

```bash
.venv/bin/python finetune/train_predictor_tdx.py \
  --data-dir ./data/tdx_import/1d \
  --tokenizer-path ./outputs/tdx_finetune/tdx_tokenizer/checkpoints/best_model \
  --epochs 30
```

- 显存占用：~6.3GB（AMP fp16，predictor_batch_size=12，gradient accumulation=4，等效 batch=48）
- 训练时间：约 4-6 小时（30 epochs）

验证产物：
| 文件 | 大小 |
|------|------|
| `model.safetensors` | ~390MB |

#### OOM 处理方案

**Tokenizer OOM**：在 `finetune/config_tdx.py` 中调小：
```python
self.batch_size = 32   # 从 50 降到 32，显存降到 ~3.5GB
```

**Predictor OOM**：两个参数可调：
```python
self.predictor_batch_size = 8    # 从 12 降到 8，显存降到 ~4.5GB
self.predictor_accumulation = 6  # 从 4 增到 6，保持等效 batch=48 不变
```

---

### Step 4: 预测上证指数

```bash
.venv/bin/python scripts/tdx_import.py \
  --symbols sh000001 \
  --dividend-type none \
  --periods 1d \
  --output-dir ./data/tdx_import_sse \
  --no-split \
  --no-continuity

.venv/bin/python scripts/predict_sse.py
```

---

### 常见坑速查

| 问题 | 原因 | 解决 |
|------|------|------|
| `bad interpreter` | venv 路径硬编码 | `rm -rf .venv` 后重建 |
| `missing N required positional arguments` | HF 不可达，config.json 下载失败 | 确认 `HF_ENDPOINT` 在 activate 脚本中 |
| `CUDA: False` | nvidia 驱动未装 | 确认 `nvidia-smi` 能输出 |
| Tokenizer OOM | batch_size=50 太大 | 改 `batch_size = 32` |
| Predictor OOM | batch_size=12 超显存 | 改 `predictor_batch_size = 8` |
| HF 下载超时 | 国内网络 | 确认 `HF_ENDPOINT=https://hf-mirror.com` |
| 训练中断恢复 | Tokenizer best_model 已存 | 不重跑 Tokenizer，用 `--tokenizer-path` 指向已有 best_model |