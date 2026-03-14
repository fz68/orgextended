# OrgExtended Python 3.3 → 3.8 迁移开发计划

**基于代码扫描结果的精简迁移计划**

---

## 📊 扫描结果摘要

**扫描日期**: 2025-03-14
**代码规模**: 92 个 Python 文件
**发现问题**: 18 处

| 问题类型 | 数量 | 优先级 | 修复时间 |
|---------|------|-------|---------|
| collections.abc 导入 | 2 | 高 | 5分钟 |
| encode("ascii") | 8 | 高 | 15分钟 |
| basestring 兼容代码 | 5 | 中 | 10分钟 |
| 其他（已注释） | 3 | 低 | 5分钟 |

**总修复时间**: **约 40 分钟**

---

## ✅ 好消息

以下原计划担心的问题**实际不存在**：
- ❌ `import imp` 使用
- ❌ yaml.load() 不安全调用
- ❌ sys.version 版本检查
- ❌ async 关键字冲突
- ❌ raw_input() / xrange() 等过时函数

**结论**: 代码库比预期干净，大部分兼容性代码已在前期维护中清理。

---

## 🔧 迁移步骤

### Step 1: 修复 collections.abc 导入 (5分钟)

**文件**: `orgparse/node.py`, `orgparse/sublimenode.py`

**当前代码**:
```python
from collections import Sequence
```

**修改为**:
```python
from collections.abc import Sequence
```

**操作**:
```bash
# 方法1: 手动编辑
# 打开文件，将第7行/第11行修改

# 方法2: 命令行替换
cd orgparse
sed -i '' 's/from collections import Sequence/from collections.abc import Sequence/' node.py
sed -i '' 's/from collections import Sequence/from collections.abc import Sequence/' sublimenode.py
```

---

### Step 2: 修复 basestring 使用 (10分钟)

**文件**: `orgparse/loader.py`

**当前代码**:
```python
import OrgExtended.orgparse.utils.py3compat as bs
# ...
if isinstance(path, bs.basestring):
    # ...
```

**修改为**:
```python
# 删除 py3compat 导入
# import OrgExtended.orgparse.utils.py3compat as bs  ← 删除这行

# 替换 bs.basestring 为 str
if isinstance(path, str):  # ← 改这里
    # ...
```

**具体位置**:
- 行 117: 删除 `import OrgExtended.orgparse.utils.py3compat as bs`
- 行 165: `isinstance(path, bs.basestring)` → `isinstance(path, str)`
- 行 174: `isinstance(path, bs.basestring)` → `isinstance(path, str)`

---

### Step 3: 修复 sourceblock.py 编码问题 (15分钟)

**文件**: `orgsourceblock.py`

**问题**: `encode("ascii")` 会在遇到非ASCII字符时报错

**查找位置**:
```bash
grep -n 'encode("ascii")' orgsourceblock.py
```

**当前代码** (行 1198-1201):
```python
tmp.write((self.curmod.WrapStart(self) + "\n").encode("ascii"))
tmp.write(self.source.encode('ascii'))
tmp.write(("\n" + self.curmod.WrapEnd(self)).encode("ascii"))
```

**修复方案** (二选一):

**方案 A**: 改用 UTF-8 编码
```python
tmp.write((self.curmod.WrapStart(self) + "\n").encode("utf-8"))
tmp.write(self.source.encode('utf-8'))
tmp.write(("\n" + self.curmod.WrapEnd(self)).encode("utf-8"))
```

**方案 B**: 改用文本模式 (推荐)
```python
# 需要同时修改文件打开模式
# 从: with open(path, 'wb') as tmp:
# 改: with open(path, 'w', encoding='utf-8') as tmp:

tmp.write(self.curmod.WrapStart(self) + "\n")
tmp.write(self.source)
tmp.write("\n" + self.curmod.WrapEnd(self))
```

**注意**: 需要检查所有 `encode("ascii")` 的位置（扫描发现8处）

---

### Step 4: 清理 py3compat 模块 (可选, 5分钟)

**文件**: `orgparse/utils/py3compat.py`, `orgparse/utils/_py3compat.py`

由于目标是完全迁移到 Python 3.8，这些兼容性文件可以删除：

```bash
# 确认没有其他引用后再删除
grep -r "py3compat" --include="*.py" . | grep -v "Binary"

# 如果只有 loader.py 使用（已修复），可以删除
rm orgparse/utils/py3compat.py
rm orgparse/utils/_py3compat.py
```

---

### Step 5: 验证测试 (10分钟)

1. **启动 Sublime Text 4**
2. **打开测试文件**: `tests/testfile.org`
3. **运行测试命令** (通过 Ctrl+Shift+P):
   - `Org Rebuild Db`
   - `Org Show TestFile`
   - `Org Show Table Tests`
   - `Org Show Source Tests`

4. **测试核心功能**:
   - [ ] 文件加载和解析
   - [ ] TODO 状态切换
   - [ ] 折叠/展开
   - [ ] 源块执行（使用中文/emoji测试）
   - [ ] 表格公式

5. **检查控制台日志** (Ctrl+`):
   - 无 ImportError
   - 无 UnicodeEncodeError
   - 无 AttributeError

---

## 🎯 快速执行脚本

如果需要自动化执行 Step 1-2，可以使用：

```bash
#!/bin/bash
# 备份
cp orgparse/node.py orgparse/node.py.bak
cp orgparse/sublimenode.py orgparse/sublimenode.py.bak
cp orgparse/loader.py orgparse/loader.py.bak

# 修复 collections
sed -i '' 's/from collections import Sequence/from collections.abc import Sequence/' orgparse/node.py
sed -i '' 's/from collections import Sequence/from collections.abc import Sequence/' orgparse/sublimenode.py

# 修复 basestring
sed -i '' '/import OrgExtended.orgparse.utils.py3compat as bs/d' orgparse/loader.py
sed -i '' 's/bs\.basestring/str/g' orgparse/loader.py

echo "✅ 完成！请手动处理 Step 3 (sourceblock.py)"
```

---

## 📋 测试清单

完成修复后，使用以下清单验证：

| 功能 | 测试方法 | 预期结果 |
|-----|---------|---------|
| 文件加载 | 打开任意 .org 文件 | 正常显示，无报错 |
| TODO 切换 | 光标在 TODO 行，按 Enter | 状态切换 |
| 折叠 | 按 Tab | 折叠/展开 |
| 源块 | 执行包含中文的源块 | 正常执行，无编码错误 |
| 议程 | `Org Show Agenda` | 显示议程 |
| 表格 | 编辑表格，按 Tab | 格式正确 |

---

## ⚠️ 注意事项

1. **非 ASCII 字符测试**: 使用包含中文、emoji 的 Org 文件测试
2. **源块执行**: 重点测试 `orgsourceblock.py` 修复后的代码
3. **备份**: 修改前建议备份代码
4. **依赖包**: 确保安装了所有依赖

---

## 🔍 后续验证

修复完成后，再次运行扫描确认：

```bash
python3 scan_py38_migration.py
```

期望结果：
- ✅ collections: 0 处
- ✅ encode_ascii: 0 处
- ✅ basestring: 0 处

---

## 📦 依赖项检查

虽然代码扫描未发现问题，但确认以下依赖已安装：

```bash
# Python 3.8 依赖
pip install pathlib pyyaml python-dateutil requests regex websocket-client six
```

或通过 Sublime Text 的 Package Control 安装依赖。

---

## 🚀 迁移时间表

| 时间 | 任务 |
|------|-----|
| 0:00 | 备份代码 |
| 0:05 | Step 1: collections.abc 导入 |
| 0:15 | Step 2: basestring 替换 |
| 0:30 | Step 3: sourceblock 编码修复 |
| 0:40 | Step 4: 清理 py3compat (可选) |
| 0:50 | Step 5: 验证测试 |

**总计**: **约 1 小时**（包含测试）

---

## 🔄 回滚方案

如果遇到问题：

```bash
# 恢复备份
cp orgparse/node.py.bak orgparse/node.py
cp orgparse/sublimenode.py.bak orgparse/sublimenode.py
cp orgparse/loader.py.bak orgparse/loader.py
```

或使用 Git 回滚：
```bash
git checkout HEAD -- orgparse/node.py orgparse/sublimenode.py orgparse/loader.py
```

---

## 📝 发布说明

迁移完成后，更新版本说明：

```markdown
## [版本号] - 2025-03-14

### Changed
- 升级到 Python 3.8 (Sublime Text 4)
- 修复 collections.abc 导入
- 改进 UTF-8 编码处理
- 移除 Python 2/3.3 兼容代码

### System Requirements
- Sublime Text 4
- Python 3.8+
```

---

## 🎉 总结

基于代码扫描，Python 3.8 迁移工作量远小于预期：

- **原计划**: 1-3 周
- **实际**: 约 1 小时

主要修复：
1. 2 处 collections 导入
2. 2 处 basestring 使用
3. 8 处 encode("ascii")

**建议**: 先执行 Step 1-2，测试后根据情况处理 Step 3-4。
