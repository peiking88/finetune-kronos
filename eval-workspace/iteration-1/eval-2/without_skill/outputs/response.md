这是搬目录后 Python 虚拟环境路径硬编码导致的经典问题。下面逐个分析和修复：

## 问题根因

Python 的 `.venv` 里到处都硬编码了创建时的绝对路径。你把项目从 `~/financial/Kronos` 搬到 `~/peiking88/Kronos`，所有这些路径全失效了。具体涉及：

1. `pyvenv.cfg` 里的 `home = ...` 等条目
2. `bin/pip`、`bin/python` 等 shebang 行写死了旧路径
3. `site-packages/` 下安装的所有 `.dist-info/RECORD` 文件里记录了旧绝对路径
4. `.pth` 文件、`easy-install.pth` 等

所以 `pip` 报 bad interpreter（shebang 指向的旧路径不存在了），`import torch` 失败。

## 修复方案

### 方案 A：重建虚拟环境（推荐）

```bash
cd ~/peiking88/Kronos
rm -rf .venv
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 方案 B：批量替换路径（不想重装依赖时）

```bash
cd ~/peiking88/Kronos
OLD="/home/li/financial/Kronos"
NEW="/home/li/peiking88/Kronos"
find .venv -type f -exec sed -i "s|$OLD|$NEW|g" {} +
```

风险：二进制 `.so` 文件里也可能嵌入了旧路径（rpath），`sed` 替换会损坏它们。

### 方案 C：用 `virtualenv-clone` 工具

```bash
pip install virtualenv-clone
virtualenv-clone ~/financial/Kronos/.venv ~/peiking88/Kronos/.venv
```

## 关于 "missing N required positional arguments"

这个报错和搬目录不一定直接相关。常见原因：

1. 依赖版本不一致
2. 模型配置文件损坏
3. 代码里调用方式有误

## 总结建议

直接用方案 A 重建 venv，这是最稳妥的。