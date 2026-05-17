不需要重新跑 Tokenizer，直接从已保存的 Tokenizer checkpoint 开始跑 Predictor 即可。

命令如下：

```bash
.venv/bin/python finetune/train_predictor_tdx.py \
  --data-dir ./data/tdx_import/1d \
  --tokenizer-path ./outputs/tdx_finetune/tdx_tokenizer/checkpoints/best_model \
  --epochs 30
```

关键点：

- `--tokenizer-path` 指向你已有的 `best_model` 目录，Predictor 会加载它继续训练。
- Tokenizer 阶段已经完成（best_model 已保存），不需要重跑。
- 如果遇到显存不足（OOM），可以减小 `--predictor_batch_size` 或增大 `--predictor_accumulation`。
- Predictor 预计耗时约 5.4 小时，显存占用约 6.3GB。