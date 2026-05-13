# MagicGeo++ 数据集问题分析报告

**生成日期**: 2025-12-06  
**验证工具**: `tools/validate_all_annotations.py`  
**DSL 规范版本**: v1.5.1 (含符号参数支持)

---

## 📊 验证结果概览

### 修复后状态 ✅

| 指标 | 数值 | 百分比 |
|------|------|--------|
| 总文件数 | 67 | 100% |
| ✅ 解析成功 | 67 | 100% |
| ✅ 约束提取成功 | 67 | 100% |
| 平均处理时间 | 0.64 ms | - |

### 修复前状态 (历史记录)

| 指标 | 数值 | 百分比 |
|------|------|--------|
| 总文件数 | 67 | 100% |
| ✅ 解析成功 | 47 | 70.1% |
| ❌ 解析失败 | 20 | 29.9% |
| 约束提取成功 | 47 | 100% (of parsed) |
| 平均处理时间 | 0.55 ms | - |

---

## 🔴 失败文件分类

### 类型 1: 变量参数问题 (7 个文件)

**问题描述**: DSL 中使用了变量名（如 `k1`, `m`, `n`）代替具体数值，但 Parser 期望的是数字。

| 文件 | 错误位置 | 问题语句 |
|------|---------|---------|
| `00015.json` | line 3 | `line_function(l, k1, b)` - 使用变量 `k1`, `b` |
| `00026.json` | line 7 | `parabola(p, a, b, c)` - 使用变量 `a`, `b`, `c` |
| `00037.json` | line 3 | `parabola(p, a, b, c)` - 使用变量 `a`, `b`, `c` |
| `00040.json` | line 3 | 类似问题 |
| `00041.json` | line 3 | 类似问题 |
| `00045.json` | line 3 | 类似问题 |
| `00046.json` | line 3 | 类似问题 |
| `00057.json` | line 4 | 类似问题 |
| `00059.json` | line 3 | 类似问题 |
| `00060.json` | line 9 | 类似问题 |
| `00062.json` | line 3 | 类似问题 |
| `00064.json` | line 4 | 类似问题 |

**示例**:
```json
// 错误写法
"parabola(p, a, b, c)"

// 正确写法 (需要具体数值)
"parabola(p, 1, -2, 3)"
```

**修复建议**:
1. 将变量替换为具体数值
2. 如果数值未知，可以使用占位值并在 `meta` 中说明
3. 或扩展 Parser 支持符号变量（需要较大改动）

---

### 类型 2: 关键字参数语法错误 (2 个文件)

**问题描述**: `right_triangle()` 使用了 `right_at=C` 的关键字参数语法，但 Parser 不支持。

| 文件 | 错误位置 | 问题语句 |
|------|---------|---------|
| `00007.json` | line 1 | `right_triangle(A, C, B, right_at=C)` |
| `00009.json` | line 1 | 类似问题 |

**示例**:
```json
// 错误写法
"right_triangle(A, C, B, right_at=C)"

// 正确写法
"triangle(A, C, B)"
"angle(ACB)=90"
```

**修复建议**:
1. 拆分为 `triangle()` + `angle()=90`
2. 或扩展 Parser 支持 `right_triangle` 关键字参数

---

### 类型 3: 非标准线段/点名 (2 个文件)

**问题描述**: 使用了非标准的线段名或点名格式。

| 文件 | 错误位置 | 问题语句 |
|------|---------|---------|
| `00004.json` | line 15 | `perpendicular_bisector(l, C, E)` - 参数顺序错误 |
| `00013.json` | line 5 | `median(D, A, B, C)` - 参数格式错误 |

**示例**:
```json
// 00004.json 错误写法
"perpendicular_bisector(l, C, E)"

// 正确写法 (根据 DSL v1.5 规范)
"perpendicular_bisector(M, AB)"  // M 在 AB 的垂直平分线上
```

```json
// 00013.json 错误写法
"median(D, A, B, C)"

// 正确写法 (根据 DSL v1.5 规范)
"median(M, A, BC)"  // M 在从 A 到 BC 的中线上
```

**修复建议**:
1. 按照 `dsl_spec.md` 规范修正参数格式

---

### 类型 4: 未实现的语法 (3 个文件)

**问题描述**: 使用了 DSL 规范中未定义或未实现的语法。

| 文件 | 错误位置 | 问题语句 |
|------|---------|---------|
| `00019.json` | line 1 | `line(a)` - 使用小写字母作为线名 |
| `00050.json` | line 11 | `ray(B, P)` - ray 语法未实现 |
| `00056.json` | line 12 | `foot(D, A, axis_x)` - `axis_x` 不是有效的线段名 |

**示例**:
```json
// 00019.json 错误写法
"line(a)"

// 正确写法 (使用大写字母或下划线命名)
"line(l1)"  或  "line(line_a)"
```

```json
// 00050.json 错误写法
"ray(B, P)"
"on(E, ray_BP)"

// 建议替代写法
"segment(BP)"
"on(E, BP_extended)"
```

```json
// 00056.json 错误写法
"foot(D, A, axis_x)"

// 正确写法
"point_coord(D, 1, 0)"  // 直接指定 D 的坐标
"perpendicular(AD, OX)"
```

**修复建议**:
1. 按照现有规范重写
2. 或扩展 DSL 规范添加 `ray`, `axis_x`, `axis_y` 支持

---

### 类型 5: 三角形命名问题 (1 个文件)

**问题描述**: 使用带数字后缀的点名组成三角形名，Parser 解析错误。

| 文件 | 错误位置 | 问题语句 |
|------|---------|---------|
| `00043.json` | line 5 | `similar(ABC, A1B1C1)` - `A1B1C1` 被解析为 6 个字符 |

**示例**:
```json
// 错误写法
"triangle(A1, B1, C1)"
"similar(ABC, A1B1C1)"

// 正确写法 (使用单字母或下划线分隔)
"triangle(D, E, F)"
"similar(ABC, DEF)"

// 或使用引号/括号明确分隔
"similar(ABC, (A1, B1, C1))"  // 需要扩展 Parser
```

**修复建议**:
1. 使用不同的单字母点名
2. 或扩展 Parser 支持带数字后缀的点名组合

---

## 📋 失败文件完整列表

| 文件 | 错误类型 | 错误信息 |
|------|---------|---------|
| `00004.json` | 参数格式 | `Invalid segment name: C` |
| `00007.json` | 关键字参数 | `Expected RPAREN, got EQUALS` |
| `00009.json` | 关键字参数 | `Expected RPAREN, got EQUALS` |
| `00013.json` | 参数格式 | `Invalid segment name: B` |
| `00015.json` | 变量参数 | `Expected number` |
| `00019.json` | 未实现语法 | `Invalid line specification: a` |
| `00026.json` | 变量参数 | `Expected number` |
| `00037.json` | 变量参数 | `Expected number` |
| `00040.json` | 变量参数 | `Expected number` |
| `00041.json` | 变量参数 | `Expected number` |
| `00043.json` | 点名解析 | `Triangle name must be 3 points` |
| `00045.json` | 变量参数 | `Expected number` |
| `00046.json` | 变量参数 | `Expected number` |
| `00050.json` | 未实现语法 | `Unexpected token: ray` |
| `00056.json` | 未实现语法 | `Invalid segment name: axis_x` |
| `00057.json` | 变量参数 | `Expected number` |
| `00059.json` | 变量参数 | `Expected number` |
| `00060.json` | 变量参数 | `Expected number` |
| `00062.json` | 变量参数 | `Expected number` |
| `00064.json` | 变量参数 | `Expected number` |

---

## 🔧 修复优先级建议

### 高优先级 (立即修复)

1. **变量参数问题** (12 个文件)
   - 影响最大，占失败文件的 60%
   - 修复方法：将变量替换为具体数值
   - 工作量：中等

2. **关键字参数语法** (2 个文件)
   - 修复方法：拆分为基本语句
   - 工作量：低

### 中优先级 (短期修复)

3. **参数格式错误** (2 个文件)
   - 修复方法：按规范调整参数顺序
   - 工作量：低

4. **三角形命名问题** (1 个文件)
   - 修复方法：使用单字母点名
   - 工作量：低

### 低优先级 (考虑扩展 DSL)

5. **未实现语法** (3 个文件)
   - 需要扩展 DSL 规范和 Parser
   - 涉及：`ray()`, `line(小写)`, `axis_x/axis_y`
   - 工作量：高

---

## 📈 修复后预期效果

| 修复阶段 | 预期成功率 | 新增成功文件 |
|---------|-----------|-------------|
| 当前状态 | 70.1% | 47/67 |
| 修复高优先级 | 91.0% | +14 |
| 修复中优先级 | 95.5% | +3 |
| 修复低优先级 | 100% | +3 |

---

## 🛠️ 推荐修复步骤

### 步骤 1: 批量修复变量参数问题

对于函数声明中使用变量的情况，有两种策略：

**策略 A: 使用占位数值**
```json
// 原始
"parabola(p, a, b, c)"

// 修复 (使用典型值)
"parabola(p, 1, 0, 0)"  // y = x²
```

**策略 B: 扩展 Parser 支持符号变量**
```python
# 在 parser.py 中添加
def parse_symbolic_param(self):
    if self.current_token.type == TokenType.NUMBER:
        return self.parse_number()
    elif self.current_token.type == TokenType.IDENTIFIER:
        return SymbolicParam(self.current_token.value)
```

### 步骤 2: 修复语法格式问题

```bash
# 运行验证工具检查每个文件
python tools/annotation_validator.py dataset/annotations/00007.json --verbose
```

### 步骤 3: 扩展 DSL 规范 (可选)

如果需要支持 `ray`, `axis_x` 等新语法，需要：
1. 更新 `dataset/dsl_spec.md`
2. 添加新的 Token 和 AST 节点
3. 实现 Parser 解析方法
4. 添加约束类和 Extractor 方法
5. 编写测试用例

---

## 📝 附录: 成功文件统计

成功解析的 47 个文件的语句和约束统计：

| 指标 | 最小值 | 最大值 | 平均值 |
|------|--------|--------|--------|
| 语句数 | 10 | 30 | 15.4 |
| 约束数 | 2 | 26 | 6.4 |

---

*报告生成完毕。如需详细修复某个文件，请运行：*
```bash
python tools/validate_all_annotations.py --verbose
```

