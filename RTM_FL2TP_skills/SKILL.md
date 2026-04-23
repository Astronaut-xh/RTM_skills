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

### 1. 表格格式：显式设置边框，不依赖模板复制
直接`ws.cell()`会覆盖原有样式。正确做法：显式定义thin_border和no_fill。
```python
thin_border = Border(left=Side('thin'), right=Side('thin'), top=Side('thin'), bottom=Side('thin'))
no_fill = PatternFill(fill_type=None)
cell.border, cell.fill = thin_border, no_fill
```

### 2. TP描述必须深度结合LRS文档，分点展开
- ❌ 错误：`条件：模块正常工作 / 激励：执行时钟测试 / 期望：符合规格`
- ✅ 正确格式：条件/激励/期望三段落，每段用编号列表分点展开，必须引用具体信号名、opcode值、状态码、FSM状态

**示例：**
```
条件：
1. test_mode_i=1，en_i=1，lane_mode_i=2'b01(4-bit模式)
2. front_state=IDLE(3'd0)，外部CSR File已实现地址0x04(CTRL)

激励：
1. 拉低pcs_n_i，4-bit模式发送WR_CSR帧（opcode=0x10, reg_addr=0x04, wdata=0x00000001）
2. 第12拍rx_count=48=expected_rx_bits，frame_valid下一拍拉高
3. 观察CSR接口和响应时序

期望：
1. 第13拍frame_valid寄存输出拉高，chk_pass=1，task_req=1
2. 第14拍task_ack=1，csr_wr_en_o产生1周期脉冲，同周期csr_addr_o=0x04、csr_wdata_o
3. 前端FSM：ISSUE(3'd2)→WAIT_RESP(3'd3)→TA(3'd4)→TX(3'd5)
4. 返回STS_OK(0x00)，4-bit模式2周期输出8bit状态码
5. 参考LRS 4.8.2：12 RX + backend + 1 TA + 2 TX
```

**必须引用的要素：**
- 信号名+精确值：`test_mode_i=1 / lane_mode_i=2'b01 / rx_count=48`
- Opcode值：`0x10(WR_CSR)/0x11(RD_CSR)/0x20(AHB_WR32)/0x21(AHB_RD32)`
- 状态码：`STS_OK(0x00)/STS_FRAME_ERR(0x01)/STS_BAD_OPCODE(0x02)/STS_AHB_ERR(0x40)`等
- FSM状态+编码双写：`IDLE(3'd0)/ISSUE(3'd2)/WAIT_RESP(3'd3)/TA(3'd4)/TX(3'd5)`及AHB的`STATE_REQ(3'b001)/STATE_WAIT(3'b010)/STATE_ERR(3'b100)`
- 关键周期数：`frame_valid延迟1拍寄存输出`，`csr_rd_en_o后1cycle采样csr_rdata_i`
- LRS章节引用：`参考LRS 4.8.2时序`

---

### 2.1 TP描述质量关键原则（防止笼统模糊）

**根因**：信息提取深度不足。LRS中80%的可量化细节在概括时被"压缩"掉了。

**核心规则：TP读者是DV工程师，需要可直接翻译成testbench的规格，不是功能概述。**

撰写TP时必须做**逆向追踪**：每个FL → 找到LRS所有相关章节 → 逐个抄出量化值填入条件/激励/期望，不要凭印象概括。

**笼统 vs 细化对比：**

| 维度 | 笼统（禁止） | 细化（要求） |
|------|-------------|-------------|
| 信号引用 | "观察模块行为" | "第12拍rx_count=48，frame_valid下一拍拉高" |
| FSM状态 | "回到IDLE" | "front_state→IDLE(3'd0)" |
| 状态码 | "返回错误码" | "返回STS_FRAME_ERR(0x01)" |
| 时序关系 | "执行CSR写" | "csr_wr_en_o脉冲1周期，同周期csr_addr_o/csr_wdata_o有效" |
| LRS关联 | 无 | "参考LRS 4.8.2时序" |

**逐拍描述原则**：当TP涉及帧接收/发送、AHB 2-phase时序等需要精确对齐的场景时，按拍描述数据；对于配置检查、状态验证等场景，只需列出信号精确值即可，不必逐拍。

**TP描述质量自检：**
- 条件：每个条件项是否都有具体信号值？FSM状态是否写了编码？
- 激励：是否引用了具体opcode/地址/数据值？关键周期数是否标注？
- 期望：状态码是否写了编码值？FSM转换是否完整？是否引用了LRS章节？

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

### Gotchas验证
- [ ] **Gotcha 1**: DR-FL E列TP编号已反标？（不能为空）
- [ ] **Gotcha 2**: 数据行样式正确？（有边框无高亮，不依赖模板复制）
- [ ] **Gotcha 3**: 填写说明区域完整保留？（内容+合并都正确）
- [ ] **Gotcha 4**: 数据区无合并单元格？（`[m for m in ws.merged_cells.ranges if 3<=m.min_row<=数据末行]`为空）
- [ ] **Gotcha 5**: 填写说明合并单元格在下移后的正确位置？

### 内容检查
- [ ] FL编号格式FL_xxx_yyy？TP编号格式TP_xxx？
- [ ] TP描述含具体条件+激励+期望？（引用信号名/寄存器名/命令码/状态码）
- [ ] TP与LRS需求细节一致？

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
| 边框丢失 | 直接cell()覆盖样式 | 显式设置border |
| TP过于通用 | 未深入分析LRS | 提取具体信号/opcode/状态码 |
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
