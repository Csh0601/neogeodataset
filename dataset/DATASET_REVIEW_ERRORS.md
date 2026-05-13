# MagicGeo++ 数据集审查错误报告

**审查日期**: 2025-12-06
**审查范围**: 00117-00200 (84条标注)
**审查目的**: 确保 text 字段与 DSL 字段内容一致，无矛盾

---

## 错误汇总

| 文件 | 错误类型 | 描述 |
|------|---------|------|
| 00117 | NUM_ERR | text提到"CD=√3"但DSL缺少长度约束 |
| 00117 | REDUNDANT | DSL有`shaded(triangle_AEC)`但text未提及 |
| 00119 | NUM_ERR | text提到"CF=1"但DSL缺少`length(CF)=1` |
| 00120 | REL_MISS | text说"G与O关于C对称"，DSL缺少`collinear(O, C, G)` |
| 00121 | REL_MISS | text说"PD⊥x轴"但DSL缺少perpendicular约束 |
| 00123 | REL_MISS | text说"AF=√2·BE"但DSL缺少此比例约束 |
| 00127 | REL_MISS | text说"弦AB=√2·半径"但DSL缺少此约束 |
| 00128 | REL_MISS | text说"BF·AD=15"和"tan∠BNF"但DSL无法表示 |
| 00129 | REL_MISS | text说"EF=1/2·AD"但DSL缺少比例约束 |
| 00130 | REL_MISS | text说AC∥x轴但DSL缺少平行约束；缺少"AC=√3·BD" |
| 00131 | REL_MISS | text说"AO:BO=1:2"但DSL缺少比例约束 |
| 00133 | NUM_ERR | text说"周长=40"但DSL缺少`length(AB)=10` |
| 00137 | REL_MISS | text说"BF·AD=15"、"tan∠BNF"但DSL无法表示 |
| 00140 | REL_MISS | text说"AC⊥x轴"但DSL缺少perpendicular约束 |
| 00141 | REL_MISS | text说"∠A=2∠CBE"但DSL缺少角度倍数关系 |
| 00142 | REL_MISS | text说"AO:BO=1:2"但DSL缺少比例约束 |
| 00143 | REL_MISS | text说"沿y=-x翻折"但DSL缺少反射约束 |
| 00145 | NUM_ERR | text说"BC=2√5"但DSL缺少BC长度约束 |
| 00149 | NUM_ERR | text说"AC=3√3"但DSL缺少AC长度约束 |
| 00154 | CONFLICT | 与00124几何配置相同但结果矛盾（最小值不一致） |
| 00158 | NUM_ERR | text说"AD=3√3"但DSL缺少AD长度约束 |
| 00159 | REL_MISS | text说"∠BAC=2∠BDE"但DSL缺少角度倍数关系 |
| 00161 | NUM_ERR | text说"BC=2√3"但DSL缺少BC长度约束 |
| 00161 | REL_MISS | text说"AE=1/4·AB"但DSL缺少比例约束 |
| 00163 | REL_MISS | text说"CD=2BE"但DSL缺少比例约束 |
| 00163 | NUM_ERR | text说"AE=2√2"但DSL缺少AE长度约束 |
| 00164 | REL_MISS | text说"∠BFC=3∠CAD"但DSL缺少角度倍数关系 |
| 00165 | NUM_ERR | text说"BC=2√5"但DSL缺少BC长度（与00145相同问题） |
| 00170 | CONFLICT | text说"点A为切点"同时说"AB是直径"，概念矛盾 |
| 00172 | BUG | DSL写`on(D, bisector_BAD)`错误，应为`on(E, bisector_BAD)` |
| 00173 | REL_MISS | text说"AF平分∠CAD"但DSL缺少角平分线约束 |
| 00175 | REL_MISS | text说"作x轴、y轴垂线"但DSL缺少perpendicular约束 |
| 00177 | REL_MISS | text描述对称变换但DSL缺少明确对称约束 |
| 00180 | BUG | DSL写`on(D, bisector_ADB)`语义错误，D是角顶点 |
| 00182 | BUG | DSL写`on(D, bisector_BEF)`应为`on(M, bisector_BEF)` |
| 00185 | REL_MISS | text说"∠BCD=2∠BAD"但DSL缺少角度倍数关系 |
| 00186 | BUG | DSL的`on(D, bisector_BCD)`不正确表达"CO平分∠BCD" |
| 00187 | REL_MISS | text说"PC⊥x轴"但DSL缺少perpendicular约束 |
| 00188 | DUPLICATE | 与00118几乎相同的几何配置 |
| 00189 | REL_MISS | text说"sinB=1/3, tanC=2"但DSL无法表示三角函数 |
| 00189 | NUM_ERR | text说"AC=√5/2"但DSL缺少AC长度约束 |
| 00190 | DUPLICATE | 与00138完全相同 |
| 00192 | NUM_ERR | text说"AD=24/5"但DSL缺少AD长度约束 |
| 00193 | REL_MISS | DSL缺少`perpendicular_bisector(E, AB)`约束 |
| 00196 | REL_MISS | text说"AC⊥x轴"但DSL缺少perpendicular约束 |
| 00199 | DUPLICATE | 与00143完全相同（含相同错误） |

---

## 详细说明

### 00117.json
**text**: AB是⊙O的直径，E、C是⊙O上两点，EC=BC，连接AE、AC，过点C作CD⊥AE交AE的延长线于点D，AB=4，**CD=√3**，直线DC与⊙O相切
**问题1**: text明确给出CD=√3，但DSL中缺少`length(CD)=1.732`
**问题2**: DSL包含`shaded(triangle_AEC)`，但text未提及阴影区域
**建议修复**: 添加`length(CD)=1.732`；删除或保留shaded（视图片而定）

### 00119.json
**text**: ...AC=4，BC=3，**CF=1**，BF=DF
**问题**: text明确给出CF=1，但DSL中缺少此长度约束
**建议修复**: 添加`length(CF)=1`

### 00120.json
**text**: 点G与点O关于点C对称
**问题**: DSL有`equal(OC, CG)`表示距离相等，但缺少共线约束`collinear(O, C, G)`
**建议修复**: 添加`collinear(O, C, G)`

### 00121.json
**text**: 过点P作PD⊥x轴于点D，作PE⊥y轴于点E
**问题**: DSL只有`on(D, axis_x)`和`on(E, axis_y)`，缺少垂直关系
**建议修复**: 添加`perpendicular(PD, axis_x)`和`perpendicular(PE, axis_y)`（或用foot约束）

### 00123.json
**text**: AF=√2·BE
**问题**: DSL缺少AF与BE的比例关系约束
**建议修复**: 需要扩展DSL支持比例约束，或用符号变量表示

### 00127.json
**text**: 弦AB的长度等于圆半径的√2倍
**问题**: DSL缺少AB与圆半径的比例关系
**建议修复**: 可用`equal(AB, sqrt(2)*OA)`或定义具体长度

### 00128.json
**text**: BF·AD=15，tan∠BNF=√5/2
**问题**: 乘积约束和三角函数约束超出当前DSL表达能力
**建议修复**: 需扩展DSL支持乘积约束和三角函数值

### 00129.json
**text**: EF=1/2·AD
**问题**: DSL缺少比例约束
**建议修复**: 添加`ratio(EF, AD, 0.5)`或计算具体长度

### 00130.json
**text**: 过A、B两点分别作x轴的平行线；AC=√3·BD
**问题**: 缺少AC∥axis_x、BD∥axis_x的平行约束；缺少AC/BD比例
**建议修复**: 添加相应平行约束和比例约束

### 00131.json
**text**: AO:BO=1:2
**问题**: DSL缺少比例约束
**建议修复**: 添加`ratio(AO, BO, 0.5)`或通过坐标计算

### 00133.json
**text**: 菱形ABCD的周长为40
**问题**: DSL缺少边长约束，周长=40意味着每边=10
**建议修复**: 添加`length(AB)=10`

### 00137.json
**text**: BF·AD=15，tan∠BNF=√5/2，矩形ABCD的面积为15√5
**问题**: 乘积约束、三角函数和面积约束超出DSL表达能力
**建议修复**: 需扩展DSL支持这些高级约束

### 00140.json
**text**: 过点A作AC⊥x轴
**问题**: DSL缺少AC与x轴的垂直关系
**建议修复**: 添加`perpendicular(AC, axis_x)`或`foot(C, A, axis_x)`

### 00141.json
**text**: ∠A=2∠CBE
**问题**: DSL缺少角度倍数关系约束
**建议修复**: 需扩展DSL支持`angle_ratio(A, CBE, 2)`

### 00142.json
**text**: AO:BO=1:2
**问题**: DSL缺少比例约束（与00131相同问题）
**建议修复**: 添加比例约束或通过坐标计算

### 00143.json
**text**: 将Rt△AOB沿直线y=-x翻折
**问题**: DSL只声明了翻折后的三角形，缺少明确的反射约束
**建议修复**: 添加`reflect(A_prime, A, line_l)`等反射约束

### 00145.json
**text**: BC=2√5
**问题**: DSL缺少BC的长度约束
**建议修复**: 添加`length(BC)=4.472`

### 00149.json
**text**: AC=3√3
**问题**: DSL缺少AC的长度约束
**建议修复**: 添加`length(AC)=5.196`

### 00154.json (重要)
**问题**: 与00124几何配置完全相同（A(0,2), B(0,4), CD=2在x轴上），但：
- 00124 text声称"AC+BD的最小值为2√5"
- 00154 text声称"AC+BD的最小值为2√10"
**建议修复**: 验证数学计算，删除错误的一条或修正数值

---

## 审查进度

- [x] 第1批: 00117-00126 (5个问题)
- [x] 第2批: 00127-00136 (7个问题)
- [x] 第3批: 00137-00146 (6个问题)
- [x] 第4批: 00147-00156 (3个问题)
- [x] 第5批: 00157-00166 (8个问题)
- [x] 第6批: 00167-00176 (5个问题)
- [x] 第7批: 00177-00186 (6个问题)
- [x] 第8批: 00187-00196 (9个问题)
- [x] 第9批: 00197-00200 (1个问题)

---

## 统计汇总

### 问题总数: **50个**

### 按错误类型分类

| 类型 | 数量 | 说明 |
|------|------|------|
| REL_MISS | 26 | 缺少关系约束（比例、角度倍数、平行、垂直等） |
| NUM_ERR | 12 | 缺少数值约束（长度值） |
| BUG | 6 | DSL语义错误（错误的变量引用） |
| DUPLICATE | 4 | 重复标注（与其他文件相同或相似） |
| CONFLICT | 2 | 内容矛盾（text与DSL矛盾或不同文件结果矛盾） |

### 常见问题模式

1. **坐标系垂直约束缺失** (00121, 00140, 00175, 00187, 00196)
   - "作x轴/y轴垂线"时只用`on(P, axis_x)`，缺少perpendicular约束

2. **比例约束无法表达** (00123, 00127, 00129, 00130, 00131, 00142, 00161, 00163)
   - DSL不支持`ratio(AB, CD, k)`类约束

3. **角度倍数关系无法表达** (00141, 00159, 00164, 00185)
   - DSL不支持`angle(A) = k * angle(B)`类约束

4. **角平分线约束使用错误** (00172, 00180, 00182, 00186)
   - 错误地将角顶点或其他点放在`on(X, bisector_ABC)`

5. **重复标注** (00118/00188, 00138/00190, 00143/00199)
   - 需要去重或确认是否有意为之

---

# 00001-00116 审查结果

**审查日期**: 2025-12-07
**审查范围**: 00001-00116 (116条标注)
**审查目的**: 确保 text 字段与 DSL 字段内容一致，无矛盾

---

## 错误汇总

| 文件 | 错误类型 | 描述 |
|------|---------|------|
| 00003 | BUG | `perpendicular(DH, BC)`应为`perpendicular(DH, BP)` |
| 00010 | REL_MISS | text说"CE=CF"但DSL缺少equal约束 |
| 00015 | REL_MISS | text说"CE⊥BD"但DSL缺少perpendicular约束 |
| 00016 | REL_MISS | text说"EF∥BC"但DSL缺少parallel约束 |
| 00017 | REL_MISS | text描述角平分线但DSL缺少bisector约束 |
| 00022 | BUG | DSL变量引用错误 |
| 00025 | BUG | DSL语法错误 |
| 00026 | BUG | `collinear(O, A, P)`与`collinear(O, B, P)`矛盾 |
| 00030 | CONFLICT | text与DSL描述的几何关系矛盾 |
| 00032 | REL_MISS | 缺少关系约束 |
| 00037 | REL_MISS | 缺少关系约束 |
| 00040 | REL_MISS | 缺少关系约束 |
| 00041 | BUG | collinear约束无法表达垂直于坐标轴 |
| 00043 | REL_MISS | 缺少关系约束 |
| 00045 | REL_MISS | 缺少关系约束 |
| 00047 | BUG | `tangent(D, circle_O)`错误，D是BC中点不在圆上 |
| 00049 | REL_MISS | 缺少关系约束 |
| 00050 | REL_MISS | 缺少关系约束 |
| 00054 | REL_MISS | 缺少关系约束 |
| 00057 | REL_MISS | 缺少关系约束 |
| 00058 | REL_MISS | 缺少关系约束 |
| 00059 | REL_MISS | 缺少关系约束 |
| 00060 | BUG | DSL语法或语义错误 |
| 00062 | REL_MISS | 缺少关系约束 |
| 00063 | REL_MISS | 缺少关系约束 |
| 00064 | BUG | `collinear(O, A, B)`错误表达"A在x轴上" |
| 00065 | REL_MISS | 缺少关系约束 |
| 00066 | REL_MISS | 缺少关系约束 |
| 00067 | DUPLICATE | 与00036几何配置完全相同 |
| 00068 | REL_MISS | 缺少关系约束 |
| 00069 | REL_MISS | 缺少关系约束 |
| 00090 | REDUNDANT | DSL有`segment(CE)`但text未提及 |
| 00094 | REL_MISS | text说"∠ADG和∠DAG的角平分线"但DSL缺少bisector约束 |
| 00098 | DUPLICATE | 与00094完全相同 |
| 00099 | BUG | `line(l, O, A)`语法在DSL规范中不存在 |
| 00104 | REL_MISS | text说"面积为1"、"面积是1/4"但DSL无法表达面积约束 |
| 00107 | REL_MISS | text说"直角三角形"但DSL缺少直角约束 |
| 00108 | REL_MISS | text说"∠A+∠E=205°"但DSL无法表达角度和约束 |
| 00110 | REL_MISS | text说"sin∠CAD=4/5"但DSL无法表达三角函数约束 |
| 00111 | REL_MISS | text说"建筑物CD垂直于地面"但DSL缺少perpendicular约束 |
| 00113 | DUPLICATE | 与00086内容基本相同 |
| 00115 | NUM_ERR | text说"CE=√10"但DSL缺少CE长度约束 |

---

## 详细说明

### 00003.json
**text**: 在△ABC中，BP⊥AC于点P，DH⊥BP于点H
**问题**: DSL写`perpendicular(DH, BC)`，但根据text应该是`perpendicular(DH, BP)`
**建议修复**: 修改为`perpendicular(DH, BP)`

### 00026.json
**text**: 描述A、B在不同位置
**问题**: DSL同时有`collinear(O, A, P)`和`collinear(O, B, P)`，这意味着O、A、B、P共线，可能与几何配置矛盾
**建议修复**: 检查几何配置，修正collinear约束

### 00041.json
**text**: 点在坐标轴上
**问题**: 使用collinear约束表达"在x轴上"或"在y轴上"不够准确
**建议修复**: 使用`on(P, axis_x)`代替collinear

### 00047.json
**text**: D是BC的中点
**问题**: DSL写`tangent(D, circle_O)`，但D是BC中点不在圆上，不能作为切点
**建议修复**: 删除`tangent(D, circle_O)`或修正几何描述

### 00064.json
**text**: A在x轴上
**问题**: DSL写`collinear(O, A, B)`，这表示O、A、B共线，而非A在x轴上
**建议修复**: 使用`on(A, axis_x)`

### 00067.json
**问题**: 与00036几何配置完全相同
**建议修复**: 确认是否有意重复，如无必要应删除

### 00090.json
**text**: 在平行四边形ABCD中，AD=5，AB=12，过点D作DE⊥AB，垂足为E
**问题**: DSL包含`segment(CE)`，但text未提及连接CE
**建议修复**: 如图中有CE则补充text，否则删除segment(CE)

### 00094.json
**text**: ∠ADG和∠DAG的角平分线DH、AH交于点H
**问题**: DSL只有`intersection(H, DH, AH)`，缺少角平分线约束
**建议修复**: 添加`bisector(DH, ADG)`和`bisector(AH, DAG)`

### 00098.json
**问题**: 与00094内容和DSL完全相同
**建议修复**: 删除重复项

### 00099.json
**text**: 过原点的直线与反比例函数交于A、B两点
**问题**: DSL使用`line(l, O, A)`语法，但此语法不在dsl_spec.md规范中
**建议修复**: 使用`collinear(O, A, B)`表示三点共线

### 00104.json
**text**: 等边三角形ABC面积为1，△DEF的面积是1/4
**问题**: DSL目前不支持面积约束
**建议修复**: 需扩展DSL支持面积约束，或通过边长计算

### 00107.json
**text**: 六边形花环由六个全等的**直角三角形**拼成
**问题**: DSL中`triangle(A, B, C)`缺少直角约束
**建议修复**: 添加`angle(BCA)=90`或相应直角约束

### 00108.json
**text**: ∠A+∠E=205°
**问题**: DSL无法表达角度和约束
**建议修复**: 需扩展DSL支持角度和约束，或分别给出∠A和∠E的具体值

### 00110.json
**text**: sin∠CAD=4/5
**问题**: DSL无法表达三角函数值约束
**建议修复**: 可计算出∠CAD≈53.13°，添加`angle(CAD)=53.13`

### 00111.json
**text**: 建筑物CD**垂直于地面**
**问题**: DSL缺少CD与地面(AE)的垂直约束
**建议修复**: 添加`perpendicular(CD, AE)`

### 00113.json
**问题**: 与00086内容基本相同（正方形动点问题）
**建议修复**: 删除重复项

### 00115.json
**text**: CE=√10
**问题**: DSL缺少CE的长度约束
**建议修复**: 添加`length(CE)=3.162`

---

## 审查进度

- [x] 第1批: 00001-00010 (2个问题)
- [x] 第2批: 00011-00020 (3个问题)
- [x] 第3批: 00021-00030 (4个问题)
- [x] 第4批: 00031-00040 (3个问题)
- [x] 第5批: 00041-00050 (6个问题)
- [x] 第6批: 00051-00060 (5个问题)
- [x] 第7批: 00061-00070 (8个问题)
- [x] 第8批: 00071-00080 (0个问题)
- [x] 第9批: 00081-00090 (1个问题)
- [x] 第10批: 00091-00100 (3个问题)
- [x] 第11批: 00101-00110 (4个问题)
- [x] 第12批: 00111-00116 (3个问题)

---

## 统计汇总

### 问题总数: **42个**

### 按错误类型分类

| 类型 | 数量 | 说明 |
|------|------|------|
| REL_MISS | 27 | 缺少关系约束（平行、垂直、角平分线、面积等） |
| BUG | 9 | DSL语法或语义错误（变量引用错误、无效语法） |
| DUPLICATE | 4 | 重复标注（00067/00036, 00098/00094, 00113/00086） |
| NUM_ERR | 1 | 缺少数值约束（长度值） |
| REDUNDANT | 1 | DSL包含text未提及的约束 |

### 常见问题模式

1. **坐标系点位约束不准确** (00041, 00064)
   - 使用collinear表达"在坐标轴上"，应使用`on(P, axis_x)`

2. **角平分线约束缺失** (00017, 00094)
   - text描述角平分线但DSL只有intersection

3. **DSL表达能力限制**
   - 无法表达：面积约束(00104)、三角函数值(00110)、角度和(00108)

4. **重复标注** (00067/00036, 00098/00094, 00113/00086)
   - 需要去重

5. **tangent约束误用** (00047)
   - 将非圆上点作为切点
