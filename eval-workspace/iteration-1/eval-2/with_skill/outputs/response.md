这是典型的 venv 硬编码路径问题。项目迁移后，`.venv` 里所有脚本的 shebang 和 `VIRTUAL_ENV` 变量仍然指向旧路径 `~/financial/Kronos`，导致全部失效。

按照一键重建流程修复：

**第一步：确认新路径存在**

```bash
ls ~/peiking88/Kronos/.venv
```

**第二步：执行一键重建**

```bash
cd ~/peiking88/Kronos
rm -rf .venv
python3 -m venv .venv
echo 'export HF_ENDPOINT=https://hf-mirror.com' >> .venv/bin/activate
source .venv/bin/activate
pip install -r requirements.txt
pip install pytest pytest-timeout
pip install -e tdxdata/
```

**第三步：验证**

```bash
pip --version
python -c 'import torch; print(torch.__version__)'
python -c 'import tdxdata; print("ok")'
echo $HF_ENDPOINT  # 应输出 https://hf-mirror.com
```

**三个报错的根因和修复对应关系：**

| 报错 | 根因 | 修复步骤 |
|------|------|----------|
| `bad interpreter` | venv 内 pip/python 的 shebang 指向旧路径 | `rm -rf .venv` 后重建 |
| `import torch` 失败 | venv 损坏，所有包均不可用 | 重建后 `pip install -r requirements.txt` |
| `missing N required positional arguments` | `config.json` 从 HF 下载失败 | 确保 `HF_ENDPOINT` 写入 activate 脚本并 source |

关键点：不要试图手动修补 shebang，直接删掉整个 `.venv` 重建。确保 `HF_ENDPOINT` 写入 `.venv/bin/activate` 而不是 `.bashrc`。