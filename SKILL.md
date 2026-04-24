---
name: RTM_FL2TP_skills
description: 根据用户提供的LRS文档和DR_FL表格，自动填写RTM（需求追溯矩阵）模板中的DR-FL和FL-TP两个Sheet。当用户Prompt中含有FL、TP、RTM、模板、填写、DR-FL、FL-TP、需求分解、测试点分解等字样时触发此skill。
priority: 10
---

# RTM 表格填写 Skill

## 概述与输入输出

本skill填写RTM模板的两个Sheet：**DR-FL**（需求→Feature分解）和**FL-TP**（Feature→Test Point分解）。

**输入**：LRS文档(.docx/.md) + DR_FL表格(.xlsx)
**输出**：填写完成的RTM模板Excel文件

---

## ⚠️ 关键注意事项（必读）

### 0. DR-FL数据必须原样复制
- **禁止**改写输入DR_FL表格的Feature描述（任何简化/概括都会丢失细节）
- **必须**从输入文件直接读取并原样填写
- **验证**：填写后逐字段对比，确保100%字符一致

### 0.1 FL-TP不填写checker和Testcase编号
- FL-TP Sheet的E列（checker编号）和F列（Testcase编号）**保持为空**
- 不填写任何内容，留空处理

### 1. 表格格式：显式设置边框，不依赖模板复制
直接`ws.cell()`会覆盖原有样式。正确做法：显式定义thin_border和no_fill。
```python
thin_border = Border(left=Side('thin'), right=Side('thin'), top=Side('thin'), bottom=Side('thin'))
no_fill = PatternFill(fill_type=None)
cell.border, cell.fill = thin_border, no_fill
```

### 2. TP描述格式（精简版）
- ✅ 正确格式：`条件：xxx 激励：xxx 期望：xxx，参考LRS §xxx`
- ❌ 避免：过于笼统（如"模块正常工作"）
- ❌ 避免：过于详细（不必精确到第几拍）

**示例（推荐）：**
```
条件：test_mode_i=1，en_i=1，lane_mode_i=16-bit模式
激励：通过WR_CSR/RD_CSR访问VERSION(0x00)、CTRL(0x04)寄存器
期望：CSR读写正确，返回STS_OK(0x00)，参考LRS §4.6
```

**必须包含的要素：**
- 关键信号/参数：`test_mode_i=1 / lane_mode_i=16-bit / addr=0x10000000`
- 命令/操作：`WR_CSR / RD_CSR / AHB_WR32 / AHB_RD_BURST×4`
- 状态码：`STS_OK(0x00) / STS_FRAME_ERR(0x01) / STS_ALIGN_ERR(0x20)`
- LRS章节引用：`参考LRS §4.8`

---

### 2.1 TP描述质量关键原则

**核心规则：TP描述需清晰明确，包含关键参数和LRS引用，但不必过于详细。**

**笼统 vs 精简对比：**

| 维度 | 笼统（禁止） | 精简（要求） |
|------|-------------|-------------|
| 信号引用 | "配置模块" | "test_mode_i=1，en_i=1，lane_mode_i=16-bit" |
| 状态码 | "返回错误" | "返回STS_ALIGN_ERR(0x20)" |
| LRS关联 | 无 | "参考LRS §4.8" |

**TP描述自检：**
- 条件：是否包含关键信号值？
- 激励：是否明确操作类型？
- 期望：是否指明预期状态码？是否引用LRS章节？

### 3. insert_rows不调整合并单元格（最易出错）
**根因**：`insert_rows`只下移单元格值，合并单元格范围不变，留原位覆盖数据区，MergedCell是只读的，写入被静默吞掉。
**解决**：先`unmerge_cells`→`insert_rows`→在新位置`merge_cells`。见步骤5代码。

---

## 执行步骤

### 步骤1-4：读取文档→提取需求→建立映射→撰写TP

1. **读取LRS**：`Document(file).paragraphs`或`open().read()`
2. **提取需求项**：找`APLC.LRS.XXX.NN`编号及完整描述
3. **建立映射**：Feature对应LRS需求编号+信号名+参数
4. **撰写TP**：每条TP含具体条件(信号/寄存器状态)+激励(opcode/数据)+期望(状态码/信号值)

### 步骤5：填写模板（含合并单元格处理）

```python
from openpyxl import load_workbook
from openpyxl.styles import Border, Side, PatternFill
import re

def get_style():
    return Border(left=Side('thin'), right=Side('thin'), top=Side('thin'), bottom=Side('thin')), PatternFill(fill_type=None)

def insert_with_merges(ws, insert_at, count, merges):
    """先解合并→插入行→重合并。merges如['A23:F23', 'B24:F24'...]"""
    for m in merges: ws.unmerge_cells(m)
    ws.insert_rows(insert_at, count)
    new_merges = []
    for m in merges:
        c1, r1, c2, r2 = re.match(r'([A-Z]+)(\d+):([A-Z]+)(\d+)', m).groups()
        nm = f'{c1}{int(r1)+count}:{c2}{int(r2)+count}'
        ws.merge_cells(nm)
        new_merges.append(nm)
    return new_merges

def fill_data(ws, data, start=3):
    thin, nof = get_style()
    for i, row in enumerate(data):
        for j, v in enumerate(row, 1):
            c = ws.cell(row=start+i, column=j, value=v)
            c.border, c.fill = thin, nof

# ===== 完整填写流程 =====
wb = load_workbook('模板.xlsx')
ws_dr, ws_tp = wb['DR-FL'], wb['FL-TP']

# DR-FL：清空数据区后填写
for r in range(3, 22):
    for c in range(1, 6): ws_dr.cell(row=r, column=c, value=None)
fill_data(ws_dr, dr_fl_data, 3)

# FL-TP：先处理合并单元格，再填写
tp_merges = ['A23:F23', 'B24:F24', 'B25:F25', 'B26:F26', 'B27:F27', 'B28:F28']
insert_with_merges(ws_tp, 23, 11, tp_merges)  # 插入11行，merges下移到Row34-39
for r in range(3, 34):
    for c in range(1, 7): ws_tp.cell(row=r, column=c, value=None)
fill_data(ws_tp, fl_tp_data, 3)
```

### 步骤6：反标TP编号到DR-FL E列

```python
fl_to_tp = {}
for row in ws_tp.iter_rows(min_row=3, values_only=True):
    if row[0] and row[2]: fl_to_tp.setdefault(row[0], []).append(row[2])
thin, nof = get_style()
for i, row in enumerate(dr_fl_data):
    if row[2] in fl_to_tp:
        c = ws_dr.cell(row=3+i, column=5, value=', '.join(fl_to_tp[row[2]]))
        c.border, c.fill = thin, nof
wb.save('输出.xlsx')
```

---

## 模板结构

| Sheet | 数据区 | 填写说明区 | 合并单元格 |
|-------|--------|-------------|-------------|
| DR-FL | Row3~21(19行) | Row22-27, 合并A22:E22等 | 标题+5行说明 |
| FL-TP | Row3~22(20行) | Row23-28, 合并A23:F23等 | 标题+5行说明 |

---

## Feature类别编号

| 编号 | 类别 | 关键词 |
|------|------|--------|
| 001-时钟 | clk,clock,频率 | 008-中断 | IRQ,interrupt |
| 002-复位 | reset,rst | 009-异常 | error,FRAME_ERR |
| 003-寄存器 | CSR,register | 010-低功耗 | power,CG |
| 004-工作模式 | mode,test_mode | 011-性能 | latency,吞吐 |
| 005-数据接口 | pdi,pdo,pcs_n | 012-DFX | debug,观测 |
| 006-控制接口 | opcode,WR_CSR | 013-Memory map | 地址映射 |
| 007-配置接口 | CTRL,LANE_MODE | 014-总线接口 | AHB,AXI,bus |

---

## 输出检查清单

### 格式检查
- [ ] DR-FL/FL-TP数据行有thin边框？
- [ ] 数据行fill_type=None（无高亮）？
- [ ] 表头/标题行保持不变？

### 数据一致性检查
- [ ] **DR-FL Feature描述与输入文件100%一致？（逐字段对比验证）**

### Gotchas验证
- [ ] **Gotcha 1**: DR-FL E列TP编号已反标？（不能为空）
- [ ] **Gotcha 2**: 数据行样式正确？（有边框无高亮，不依赖模板复制）
- [ ] **Gotcha 3**: 填写说明区域完整保留？（内容+合并都正确）
- [ ] **Gotcha 4**: 数据区无合并单元格？（`[m for m in ws.merged_cells.ranges if 3<=m.min_row<=数据末行]`为空）
- [ ] **Gotcha 5**: 填写说明合并单元格在下移后的正确位置？

### 内容检查
- [ ] FL编号格式FL_xxx_yyy？TP编号格式TP_xxx？
- [ ] TP描述含条件+激励+期望？（引用关键信号名/命令码/状态码）
- [ ] TP描述引用LRS章节？
- [ ] checker编号和Testcase编号列为空？（不填写）

---

## Gotchas详解（必读）

### Gotcha 1: 忘记反标TP编号
**现象**：DR-FL E列为空。
**原因**：遗漏步骤6。
**修正**：填写顺序：DR-FL → FL-TP → 反标E列 → 保存

### Gotcha 2: 数据行样式错误
**现象**：数据行有高亮（因复制了表头样式）或无边框（因模板单元格本身无边框）。
**错误**：`copy_cell_style(ws.cell(2,5), target)` 复制了表头高亮。
**修正**：显式定义样式，见步骤5代码。`thin_border + no_fill`。

### Gotcha 3: insert_rows导致填写说明被覆盖
**现象**：数据超出行数，写入覆盖了填写说明。
**修正**：先`insert_rows`在填写说明前插入空行，填写说明自动下移。

### Gotcha 4: insert_rows后合并单元格残留数据区（关键）
**现象**：FL-TP某些行只能写A/B列，C-F列为空；程序不报错但值写不进。
**根因**：`insert_rows`不调整合并单元格范围，原合并范围（如A23:F23）留在数据区，MergedCell是只读代理。
**修正**：必须使用`insert_with_merges()`函数：先解合并→插入→重合并。
**验证**：检查数据区无合并：`assert not [m for m in ws.merged_cells.ranges if 3<=m.min_row<=data_end]`

### Gotcha 5: 手动重写填写说明导致内容丢失
**现象**：填写说明原文有换行和编号，输出变成简化版。
**根因**：误以为需要手动重建填写说明区，凭记忆简化了内容。
**修正**：绝不重写填写说明，依赖`insert_rows`自动下移保留原文。

---

## 常见错误速查

| 错误 | 原因 | 修正 |
|------|------|------|
| Feature描述不一致 | 主观改写/简化输入数据 | 从输入文件原样读取，禁止改写 |
| 边框丢失 | 直接cell()覆盖样式 | 显式设置border |
| TP过于笼统 | 未结合LRS | 引用关键信号/opcode/状态码+LRS章节 |
| TP过于详细 | 逐拍描述 | 精简为条件/激励/期望三段，不必精确到第几拍 |
| TP与Feature不匹配 | 未建立映射 | 每个TP对应LRS需求项 |
| 数据行A/B列有值C-F空 | 合并单元格残留 | unmerge→insert→merge |
| 填写说明简化 | 手动重写 | 依赖insert_rows下移保留 |
| TP数量不稳定 | 无固定分解规则 | 按正交分解法六条规则生成（见references/tp_completeness_methodology.md） |
| TP覆盖不完整 | 未建立覆盖矩阵 | 执行完备性验收检查清单（见references/tp_completeness_methodology.md） |

---

## 参考文档

| 文件 | 说明 |
|------|------|
| `references/template_structure.md` | RTM模板结构定义 |
| `references/tp_completeness_methodology.md` | TP完备性保证方法论（正交分解法+覆盖矩阵+验收清单） |
| `scripts/fill_rtm_template.py` | 填写辅助脚本 |
| `evals/evals.json` | 评估用例 |
