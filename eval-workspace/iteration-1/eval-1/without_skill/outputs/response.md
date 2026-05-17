项目结构清楚了。SKILL.md 里已经写明了这个场景的处理方式。

**不需要重新跑 Tokenizer。** Tokenizer 的 best_model 已经保存好了，Predictor 训练只需要用它作为输入，不依赖 Tokenizer 的训练过程状态。

直接跑 Predictor：

```bash
.venv/bin/python finetune/train_predictor_tdx.py \
  --data-dir ./data/tdx_import/1d \
  --tokenizer-path ./outputs/tdx_finetune/tdx_tokenizer/checkpoints/best_model \
  --epochs 30
```

说明：
- `--tokenizer-path` 指向你已保存的 Tokenizer best_model 目录，Predictor 加载里面的 `model.safetensors` 和 `config.json`，只做推理（encode），不涉及 Tokenizer 训练状态。
- Predictor 训练是独立的第二阶段，从头开始（epoch 0），跟上次跑到哪无关——它不是续训 checkpoint，而是用已完成的 Tokenizer 重新训练 Predictor。
- 确保 `./data/tdx_import/1d/` 下 `train_data.pkl` 和 `val_data.pkl` 还在（数据导入不需要重跑，只要文件还在）。