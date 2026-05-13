# NeoGeo中文几何描述 DSL 规范 v1.8.3

**作者：Barry Chen**

---

## 目录

1. [前言](#1-前言)
2. [DSL 设计原则](#2-dsl-设计原则)
3. [形式化语法定义（EBNF）](#3-形式化语法定义ebnf)
4. [基础元素（Entities）](#4-基础元素entities)
5. [几何关系（Relations）](#5-几何关系relations)
6. [操作型构造指令（Operations）](#6-操作型构造指令operations)
7. [图形结构（Shapes）](#7-图形结构shapes)
8. [宏指令与展开规则（Macros）](#8-宏指令与展开规则macros)
9. [圆与弧（Circles &amp; Arcs）](#9-圆与弧circles--arcs)
10. [角度与长度表达](#10-角度与长度表达)
11. [点的位置关系（高级）](#11-点的位置关系高级)
12. [几何变换与折叠操作](#12-几何变换与折叠操作)
13. [坐标系与函数图像（Coordinate &amp; Functions）](#13-坐标系与函数图像coordinate--functions)
14. [模糊语义消歧指令（Disambiguation）](#14-模糊语义消歧指令disambiguation)
15. [中文表达对照表](#15-中文表达对照表)
16. [合法性规则（Validity Rules）](#16-合法性规则validity-rules)
17. [标准化与规范化（Canonicalization）](#17-标准化与规范化canonicalizationv16)
18. [标注示例](#18-标注示例)
19. [版本管理与变更日志](#19-版本管理与变更日志)

---

## 1. 前言

本 DSL（Domain-Specific Language）用于将中文几何绘图指令转换为结构化、可解析的形式逻辑表示，用于：

- 中文 → DSL 模型训练（NL → DSL）
- DSL → 图像生成（DSL → Diagram）
- DSL → 几何求解（Symbolic / Numeric Solver）
- 几何图像理解（Diagram Parsing）
- 多轮几何编辑（Geometric Editing Agent）

DSL 必须具备：

- **语义明确**：每条指令对应一个明确的几何概念、操作或者skills
- **可解析性**：可被solver自动解析、验证和执行
- **可扩展性**：支持通过宏指令扩展高层语义
- **与实际几何一致**：所有约束在几何上可满足
- **适合 LLM 生成**：格式简洁，便于语言模型学习和生成
- **贴合中文习惯**：支持中文几何题中常见的"操作型"、"隐式性"表达

**本规范定义所有合法 token，任何人员和模型必须严格遵循，不可进行随意篡改和未经允许的调整。**

---

## 2. DSL 设计原则

### 2.1 一条 DSL = 一个基本几何事实（Atomic Fact）

每条 DSL 指令表达一个原子几何事实，不能同时表达多个独立事实。

```text
parallel(AB, CD)      ✓ 表达一个平行关系
angle(AOB)=40         ✓ 表达一个角度值
on(C, circle_O)       ✓ 表达一个点在圆上
```

### 2.2 所有关键字必须小写

```text
circle(O)   ✓
Circle(O)   ✗
TRIANGLE    ✗
```

### 2.3 点名必须为大写字母（或带 prime 后缀）

合法点名格式：

```text
A, B, C, D            # 单个大写字母
A1, B2, C3            # 大写字母 + 数字
O, P, Q               # 常用于圆心或特殊点
B_prime, C_prime      # 带 _prime 后缀（表示变换后的点）
```

### 2.4 线段用两个点名连写表示

```text
segment(AB)           # 线段 AB
parallel(AB, CD)      # AB 平行于 CD
equal(AB, AC)         # AB 等于 AC
```

### 2.5 与中文描述结构对齐

DSL 设计尽量与中文题目的句子结构保持对应关系：

| 中文表达      | DSL 对应              |
| ------------- | --------------------- |
| 在△ABC中     | `triangle(A, B, C)` |
| 点 D 在 BC 上 | `on(D, BC)`         |
| 连接 AD       | `connect(A, D)`     |
| AB = AC       | `equal(AB, AC)`     |

### 2.6 简单核心 + 可扩展宏

- 核心层只保留基本图元和基本约束
- 高层概念（如中线、高、角平分线）通过宏指令定义，由解释器展开为基础约束

---

## 3. 形式化语法定义（EBNF）

以下使用扩展巴科斯-诺尔范式（EBNF）定义 DSL 的完整语法规则：

```ebnf


(* 程序结构 *)
program        ::= statement*
statement      ::= decl_stmt | constraint_stmt | op_stmt | macro_stmt | assign_stmt
                 | coord_stmt | function_stmt | function_transform | function_geom
                 | disambig_stmt

(* 声明语句 *)
decl_stmt      ::= point_decl | line_decl | segment_decl | ray_decl
                 | circle_decl | arc_decl | sector_decl | shape_decl

point_decl     ::= "point" "(" POINT ")"
                 | "points" "(" POINT ("," POINT)* ")"

line_decl      ::= "line" "(" SEGMENT ")"
                 | "line" "(" ID "," POINT "," POINT ")"

segment_decl   ::= "segment" "(" SEGMENT ")"

ray_decl       ::= "ray" "(" POINT "," POINT ")"

circle_decl    ::= "circle" "(" POINT ")"
                 | "circle" "(" POINT "," "r" "=" NUMBER ")"
                 | "circle" "(" POINT "," "radius" "=" SEGMENT ")"

arc_decl       ::= "arc" "(" POINT "," POINT "," CIRCLE_REF ")"

sector_decl    ::= "sector" "(" POINT "," POINT "," POINT ")"
                 | "sector" "(" POINT "," POINT "," POINT "," "angle" "=" NUMBER ")"
                 | "sector" "(" POINT "," POINT "," POINT "," "r" "=" NUMBER ")"

shape_decl     ::= "triangle" "(" POINT "," POINT "," POINT ")"
                 | "quadrilateral" "(" POINT "," POINT "," POINT "," POINT ")"
                 | "polygon" "(" POINT ("," POINT)+ ")"
                 | "rectangle" "(" POINT "," POINT "," POINT "," POINT ")"
                 | "parallelogram" "(" POINT "," POINT "," POINT "," POINT ")"
                 | "square" "(" POINT "," POINT "," POINT "," POINT ")"
                 | "trapezoid" "(" POINT "," POINT "," POINT "," POINT ")"  (* v1.8.3: 梯形 *)

(* 约束语句 *)
constraint_stmt ::= relation_constraint | position_constraint | measure_constraint

relation_constraint ::= "parallel" "(" LINE "," LINE ")"
                      | "perpendicular" "(" LINE "," LINE ")"
                      | "equal" "(" SEGMENT "," SEGMENT ")"
                      | "equal_angle" "(" ANGLE "," ANGLE ")"
                      | "collinear" "(" POINT "," POINT "," POINT ("," POINT)* ")"
                      | "concyclic" "(" POINT "," POINT "," POINT "," POINT ")"
                      | "congruent" "(" TRIANGLE_ID "," TRIANGLE_ID ")"
                      | "similar" "(" TRIANGLE_ID "," TRIANGLE_ID ")"
                      | "tangent" "(" POINT "," CIRCLE_REF ")"
                      | "tangent" "(" LINE "," CIRCLE_REF ")"
                      | "bisector" "(" LINE "," ANGLE ")"
                      | "inscribed" "(" TRIANGLE_ID "," CIRCLE_REF ")"
                      | "circumscribed" "(" CIRCLE_REF "," TRIANGLE_ID ")"
                      | "circumcircle" "(" POINT "," POINT "," POINT "," POINT ")"

position_constraint ::= "on" "(" POINT "," LOCATION ")"
                      | "midpoint" "(" POINT "," SEGMENT ")"
                      | "arc_midpoint" "(" POINT "," SEGMENT "," CIRCLE_REF ("," "arc_type" "=" STRING)? ")"
                      | "foot" "(" POINT "," POINT "," FOOT_LINE ")"
                      | "intersection" "(" POINT "," LINE_REF "," LINE_REF ")"  (* v1.8.1: 支持延长线 *)
                      | "intersection" "(" POINT "," LINE_REF "," CIRCLE_REF ("," "index" "=" NUMBER)? ")"
                      | "intersection" "(" POINT "," CIRCLE_REF "," CIRCLE_REF ("," "index" "=" NUMBER)? ")"
                      | "trisection" "(" POINT "," SEGMENT "," "near" "=" POINT ")"
                      | "perpendicular_bisector" "(" POINT "," SEGMENT ")"
                      | "perpendicular_bisector" "(" SEGMENT "," SEGMENT ")"  (* v1.8.2: 线段是中垂线 *)

measure_constraint ::= "angle" "(" ANGLE ")" "=" NUMBER
                     | "angle" "(" ANGLE ")" "=" NUMBER "*" "angle" "(" ANGLE ")"  (* v1.6.7: angle ratio *)
                     | "length" "(" SEGMENT ")" "=" NUMBER
                     | "ratio" "(" SEGMENT "," SEGMENT ")" "=" NUMBER
                     | "right_angle" "(" ANGLE ")"
                     | "sin" "(" ANGLE ")" "=" NUMBER    (* v1.6.4: angle sine value *)
                     | "cos" "(" ANGLE ")" "=" NUMBER    (* v1.6.4: angle cosine value *)
                     | "tan" "(" ANGLE ")" "=" NUMBER    (* v1.6.4: angle tangent value *)

(* 操作型构造语句 - 核心创新 *)
op_stmt        ::= "connect" "(" POINT "," POINT ")"
                 | "extend" "(" SEGMENT "," POINT ("," "length" "=" NUMBER)? ")"
                 | "draw_perpendicular" "(" POINT "," LINE "," ID ")"
                 | "draw_parallel" "(" POINT "," LINE "," ID ")"
                 | "bisect_angle" "(" POINT "," POINT "," POINT "," ID ")"
                 | "draw_altitude" "(" POINT "," POINT "," POINT "," POINT ")"
                 | "draw_median" "(" POINT "," POINT "," POINT "," POINT ")"

(* 宏语句 *)
macro_stmt     ::= "isosceles" "(" POINT "," POINT "," POINT ")"
                 | "isosceles_right" "(" POINT "," POINT "," POINT ")"
                 | "equilateral" "(" POINT "," POINT "," POINT ")"
                 | "right_triangle" "(" POINT "," POINT "," POINT "," "right_at" "=" POINT ")"
                 | "median" "(" POINT "," POINT "," POINT "," POINT ")"
                 | "altitude" "(" POINT "," POINT "," POINT "," POINT ")"
                 | "angle_bisector" "(" POINT "," POINT "," POINT "," POINT ")"

(* 赋值语句 *)
assign_stmt    ::= "angle" "(" ANGLE ")" "=" "angle" "(" ANGLE ")"
                 | "length" "(" SEGMENT ")" "=" "length" "(" SEGMENT ")"

(* 坐标系与函数语句 *)
coord_stmt     ::= "coordinate_system" "(" POINT ")"
                 | "axis_x" "(" POINT ")"
                 | "axis_y" "(" POINT ")"
                 | "point_coord" "(" POINT "," NUMBER "," NUMBER ")"

function_stmt  ::= basic_function | power_function | exp_log_function 
                 | trig_function | inverse_trig_function | function_expr

basic_function ::= "line_function" "(" ID "," VALUE "," VALUE ")"
                 | "parabola" "(" ID "," VALUE "," VALUE "," VALUE ")"
                 | "hyperbola" "(" ID "," VALUE ")"

power_function ::= "power_function" "(" ID "," VALUE ")"

exp_log_function ::= "exp_function" "(" ID "," (VALUE | "e") ")"
                   | "log_function" "(" ID "," (VALUE | "e") ")"

trig_function  ::= "sin_function" "(" ID "," VALUE "," VALUE "," VALUE ")"
                 | "cos_function" "(" ID "," VALUE "," VALUE "," VALUE ")"
                 | "tan_function" "(" ID ")"
                 | "cot_function" "(" ID ")"
                 | "sec_function" "(" ID ")"
                 | "csc_function" "(" ID ")"

inverse_trig_function ::= "arcsin_function" "(" ID ")"
                        | "arccos_function" "(" ID ")"
                        | "arctan_function" "(" ID ")"
                        | "arccot_function" "(" ID ")"

function_expr  ::= "function_expr" "(" ID "," STRING ")"

(* 函数变换语句 *)
function_transform ::= "translate_function" "(" ID "," ID "," NUMBER "," NUMBER ")"
                     | "scale_function" "(" ID "," ID "," NUMBER "," NUMBER ")"
                     | "reflect_function" "(" ID "," ID "," AXIS ")"

(* 函数与几何交互 *)
function_geom  ::= "function_point" "(" POINT "," ID ")"
                 | "function_intersection" "(" POINT "," ID "," ID ")"
                 | "axis_intersection" "(" POINT "," ID "," AXIS ")"
                 | "tangent_point" "(" POINT "," ID "," ID ")"
                 | "tangent_line" "(" ID "," ID "," POINT ")"
                 | "asymptote" "(" ID "," ID ")"

(* 模糊语义消歧语句 - v1.4 核心创新 *)
disambig_stmt  ::= direction_stmt | region_stmt | bisector_on_stmt | choose_stmt

direction_stmt ::= "direction" "(" POINT "," SEGMENT "," "towards" "=" POINT ")"
                 | "on_extended" "(" POINT "," SEGMENT "," "towards" "=" POINT ")"

region_stmt    ::= "inside" "(" POINT "," REGION_REF ")"
                 | "outside" "(" POINT "," REGION_REF ")"
                 | "sector" "(" POINT "," POINT "," POINT "," POINT ")"  (* sector(P, O, A, B) - P在以O为圆心、从OA到OB的扇形内 *)
                 | "half_plane" "(" POINT "," SEGMENT "," "side" "=" SIDE ")"
                 | "shaded" "(" REGION_REF ")"

bisector_on_stmt ::= "on" "(" POINT "," "bisector_" ANGLE "," "near" "=" POINT ")"

choose_stmt    ::= "choose" "(" POINT "," "on" "=" LOCATION ")"
                 | "choose" "(" POINT "," "in" "=" REGION_REF ")"

(* 终结符定义 *)
POINT          ::= [A-Z] [0-9]* | [A-Z] [0-9]* "_prime"
SEGMENT        ::= POINT POINT
LINE           ::= SEGMENT | ID
ANGLE          ::= POINT POINT POINT      (* 角标识，如 ABC 表示 ∠ABC；连写用于约束如 angle(ABC)=60，逗号分隔用于函数参数如 bisect_angle(A,B,C,l) *)
TRIANGLE_ID    ::= POINT POINT POINT
CIRCLE_REF     ::= "circle_" POINT
RAY_REF        ::= "ray_" POINT POINT                (* 射线引用，如 ray_AB *)
BISECTOR_REF   ::= "bisector_" POINT POINT POINT     (* 角平分线引用，如 bisector_ABC *)
TRIANGLE_REF       ::= "triangle_" POINT POINT POINT
SECTOR_REF         ::= "sector_" POINT POINT POINT       (* 扇形引用，如 sector_OAB，连写 *)
RECTANGLE_REF      ::= "rectangle_" POINT POINT POINT POINT     (* 矩形引用，如 rectangle_ABCD *)
PARALLELOGRAM_REF  ::= "parallelogram_" POINT POINT POINT POINT (* 平行四边形引用 *)
SQUARE_REF         ::= "square_" POINT POINT POINT POINT        (* 正方形引用 *)
QUADRILATERAL_REF  ::= "quadrilateral_" POINT POINT POINT POINT (* 四边形引用 *)
TRAPEZOID_REF      ::= "trapezoid_" POINT POINT POINT POINT     (* v1.8.3: 梯形引用 *)
POLYGON_REF        ::= "polygon_" POINT+                        (* 多边形引用，如 polygon_ABCDEF *)
REGION_REF     ::= TRIANGLE_REF | CIRCLE_REF | SECTOR_REF
               | RECTANGLE_REF | PARALLELOGRAM_REF | SQUARE_REF
               | QUADRILATERAL_REF | TRAPEZOID_REF | POLYGON_REF
LOCATION       ::= SEGMENT | SEGMENT "_extended" | RAY_REF | CIRCLE_REF | FUNC_REF
FOOT_LINE      ::= SEGMENT | SEGMENT "_extended"   (* v1.7.1: foot约束支持延长线 *)
LINE_REF       ::= LINE | LINE "_extended"         (* v1.8.1: intersection支持延长线 *)
FUNC_REF       ::= ID                           (* 函数引用 *)
ID             ::= [a-z] [a-z0-9]*
SIGNED_NUMBER  ::= "-"? [0-9]+ ("." [0-9]+)?    (* 支持负数，v1.6 *)
VALUE          ::= SIGNED_NUMBER | SYMBOL       (* 数值或符号变量，v1.6 *)
SYMBOL         ::= [a-z] [a-z0-9]*              (* 符号变量：a, b, k1 等 *)
NUMBER         ::= SIGNED_NUMBER                (* 向后兼容别名 *)
STRING         ::= '"' [^"]* '"'                (* 函数表达式字符串 *)
AXIS           ::= "x" | "y" | "origin"         (* 坐标轴或原点 *)
SIDE           ::= "left" | "right"             (* 半平面侧向 *)
```

---

## 4. 基础元素（Entities）

### 4.1 点的声明

```text
point(A)                    # 声明单个点 A
points(A, B, C, D)          # 批量声明多个点（推荐用于多点场景）
```

### 4.2 线的声明

```text
line(AB)                    # 直线 AB（经过点 A 和 B 的直线）
line(l, A, B)               # 命名直线 l，经过点 A 和 B（便于后续引用）
segment(AB)                 # 线段 AB
ray(A, B)                   # 射线 AB（从 A 出发经过 B）
```

### 4.3 圆的声明

```text
circle(O)                   # 圆心为 O 的圆
circle(O, r=5)              # 圆心为 O，半径为 5
circle(O, radius=AB)        # 圆心为 O，半径等于线段 AB 的长度
```

### 4.4 基本图形声明

```text
triangle(A, B, C)           # △ABC
quadrilateral(A, B, C, D)   # 四边形 ABCD（按顺序连接）
polygon(A, B, C, D, E)      # 多边形（顶点按顺序）
rectangle(A, B, C, D)       # 矩形 ABCD
parallelogram(A, B, C, D)   # 平行四边形 ABCD
square(A, B, C, D)          # 正方形 ABCD
```

**注意**：

- `triangle(A, B, C)` 隐式引入三条边 AB、BC、CA
- 四边形类图形的顶点必须按顺序（顺时针或逆时针）排列

---

## 5. 几何关系（Relations）

### 5.1 平行与垂直

```text
parallel(AB, CD)            # AB ∥ CD
perpendicular(AB, CD)       # AB ⟂ CD
```

### 5.2 等长关系

```text
equal(AB, AC)               # AB = AC
ratio(AB, AC)=2             # AB : AC = 2 : 1
```

### 5.3 角度关系

```text
equal_angle(ABC, DEF)       # ∠ABC = ∠DEF
angle(ABC)=angle(DEF)       # 等价写法
```

### 5.4 共线与共圆

```text
collinear(A, B, C)          # 三点共线
collinear(A, B, C, D)       # 四点共线（可扩展）
concyclic(A, B, C, D)       # 四点共圆
```

### 5.5 全等与相似

```text
congruent(ABC, DEF)         # △ABC ≌ △DEF
similar(ABC, DEF)           # △ABC ∼ △DEF
```

### 5.6 点的位置关系

```text
on(P, AB)                   # 点 P 在线段 AB 上
on(P, AB_extended)          # 点 P 在线段 AB 的延长线上
on(P, ray_AB)               # 点 P 在射线 AB 上
on(P, circle_O)             # 点 P 在圆 O 上
on(P, axis_x)               # 点 P 在 x 轴上（P.y = 0）
on(P, axis_y)               # 点 P 在 y 轴上（P.x = 0）
```

**坐标轴约束说明**：

- `on(P, axis_x)` 表示点 P 在 x 轴上，等价于约束 P.y = 0
- `on(P, axis_y)` 表示点 P 在 y 轴上，等价于约束 P.x = 0
- 通常与 `coordinate_system(O)` 配合使用（见第 13 节）
- 也可独立使用，此时以默认原点 (0, 0) 为参考

### 5.7 切线关系

```text
tangent(A, circle_O)        # A 是圆 O 上的切点（隐含存在一条过 A 的切线）
tangent(l, circle_O)        # 直线 l 与圆 O 相切
```

**语义说明**：

- `tangent(A, circle_O)`：声明点 A 是圆 O 的一个切点。这意味着存在一条经过 A 的直线与圆 O 相切于 A 点。通常与 `on(A, circle_O)` 配合使用。
- `tangent(l, circle_O)`：声明直线 l 与圆 O 相切。切点位置由其他约束确定。

**对应中文**：

- 「过点 A 作圆 O 的切线」→ 需要组合多个约束
- 「直线 l 与圆 O 相切于点 A」→ `tangent(l, circle_O)` + `on(A, l)` + `on(A, circle_O)`

### 5.8 角平分线关系

```text
bisector(l, ABC)            # 直线 l 是 ∠ABC 的平分线
```

**蕴含约束**：`bisector(l, ABC)` 隐含 `on(B, l)`（角平分线必过顶点 B）

---

## 6. 操作型构造指令（Operations）

**这是本项目的核心创新点**，用于直接承接中文几何题中的"动词式操作"描述。

### 6.1 连接

```text
connect(A, D)               # 连接 AD
```

**语义展开**：隐式引入 `segment(AD)`

**对应中文**：「连接 AD」「连接点 A 与点 D」

### 6.2 延长

```text
extend(AB, D)               # 延长 AB 到点 D
extend(AB, D, length=3)     # 延长 AB 到点 D，使 BD = 3
```

**语义展开**：生成 `on(D, AB_extended)` + 可选的长度约束

**对应中文**：「延长 AB 至点 D」「将 AB 延长到 D」

### 6.3 作垂线

```text
draw_perpendicular(P, AB, l)     # 过点 P 作直线 AB 的垂线，命名为 l
draw_perpendicular(P, l1, l2)    # 过点 P 作直线 l1 的垂线 l2
```

**语义展开**：生成 `line(l, ...)` + `perpendicular(l, AB)` + `on(P, l)`

**对应中文**：「过点 P 作 AB 的垂线」「从 P 向 AB 作垂线」

### 6.4 作平行线

```text
draw_parallel(P, AB, l)          # 过点 P 作直线 AB 的平行线，命名为 l
```

**语义展开**：生成 `line(l, ...)` + `parallel(l, AB)` + `on(P, l)`

**对应中文**：「过点 P 作 AB 的平行线」「经过 P 作一条直线平行于 AB」

### 6.5 作角平分线

```text
bisect_angle(A, B, C, l)         # 作 ∠ABC 的平分线，命名为 l
```

**语义展开**：生成角平分线约束 `bisector(l, ABC)`

**bisector 语义说明**：

- `bisector(l, ABC)` 完整表达"l 是 ∠ABC 的角平分线"
- **蕴含约束**：`on(B, l)`（角平分线必过顶点 B）
- **几何性质**：l 平分 ∠ABC，即 ∠ABL = ∠LBC（L 为 l 上任意点，仅作语义描述，不进入可执行展开）
- 如需显式使用角平分线上的点，应使用 `angle_bisector(P, A, B, C)` 宏

**对应中文**：「作 ∠ABC 的平分线」「作角 B 的平分线」

### 6.6 作高

```text
draw_altitude(A, B, C, H)        # 从顶点 A 向对边 BC 作高，垂足为 H
```

**语义展开**：生成 `foot(H, A, BC)` + `segment(AH)` + `perpendicular(AH, BC)`

**对应中文**：「从 A 向 BC 作高」「作 △ABC 中 BC 边上的高」

### 6.7 作中线

```text
draw_median(A, B, C, M)          # 从顶点 A 向对边 BC 的中点 M 作中线
```

**语义展开**：生成 `midpoint(M, BC)` + `segment(AM)`

**对应中文**：「作 △ABC 的中线 AM」「连接 A 与 BC 的中点」

---

## 7. 图形结构（Shapes）

### 7.1 基本多边形

```text
triangle(A, B, C)           # 三角形
quadrilateral(A, B, C, D)   # 四边形
polygon(A, B, C, D, E)      # 多边形（至少 3 个顶点）
```

### 7.2 特殊三角形（宏指令）

#### 7.2.1 等腰三角形

```text
isosceles(A, B, C)          # 等腰三角形，AB = AC
```

**隐含约束**：`triangle(A, B, C)` + `equal(AB, AC)`

#### 7.2.2 等腰直角三角形

```text
isosceles_right(A, B, C)    # 等腰直角三角形，AB = AC，∠BAC = 90°
```

**隐含约束**：`triangle(A, B, C)` + `equal(AB, AC)` + `right_angle(BAC)`

#### 7.2.3 等边三角形

```text
equilateral(A, B, C)        # 等边三角形
```

**隐含约束**：`triangle(A, B, C)` + `equal(AB, BC)` + `equal(BC, AC)`

#### 7.2.4 直角三角形

```text
right_triangle(A, B, C, right_at=B)   # 直角三角形，直角在 B 处
```

**隐含约束**：`triangle(A, B, C)` + `perpendicular(AB, CB)`

### 7.3 特殊四边形

#### 7.3.1 矩形

```text
rectangle(A, B, C, D)       # 矩形 ABCD
```

**隐含约束**：

- `parallel(AB, CD)` + `parallel(BC, AD)`
- `perpendicular(AB, BC)`
- `equal(AB, CD)` + `equal(BC, AD)`

#### 7.3.2 平行四边形

```text
parallelogram(A, B, C, D)   # 平行四边形 ABCD
```

**隐含约束**：

- `parallel(AB, CD)` + `parallel(BC, AD)`
- `equal(AB, CD)` + `equal(BC, AD)`

#### 7.3.3 梯形 (v1.8.3)

```text
trapezoid(A, B, C, D)       # 梯形 ABCD
```

**隐含约束**：

- `parallel(AB, CD)` - AB 和 CD 为两条平行的底边
- BC 和 AD 为两条腰（不要求平行）

**注意**：与平行四边形的区别是只有一对对边平行，不添加 `parallel(BC, AD)` 约束。

**对应中文**：「梯形 ABCD，其中 AB // CD」

#### 7.3.4 正方形

```text
square(A, B, C, D)          # 正方形 ABCD
```

**隐含约束**：`rectangle(A, B, C, D)` + `equal(AB, BC)`

---

## 8. 宏指令与展开规则（Macros）

宏指令是为简化表达和提升可读性而设计的高阶构造，在解释执行时会自动展开为基础指令集合。

### 8.0 命名与参数设计原则

本 DSL 区分两类高阶指令，它们的参数顺序设计遵循不同的原则：

| 类型     | 前缀       | 参数顺序               | 语义风格                    |
| -------- | ---------- | ---------------------- | --------------------------- |
| 操作指令 | `draw_*` | 操作主体在前，结果在后 | "从 A 向 BC 作高，垂足为 H" |
| 宏指令   | 无前缀     | 结果点在前，依赖点在后 | "H 是 A 到 BC 的垂足"       |

**对比示例**：

| 操作指令                      | 宏指令                   | 语义                            |
| ----------------------------- | ------------------------ | ------------------------------- |
| `draw_altitude(A, B, C, H)` | `altitude(H, A, B, C)` | 从 A 向 BC 作高，垂足为 H       |
| `draw_median(A, B, C, M)`   | `median(M, A, B, C)`   | 从 A 向 BC 中点作中线，中点为 M |

两者语义等价，展开后的约束相同。选择哪种取决于原始中文表达的风格：

- **操作式表达**（如"从 A 向 BC 作高"）→ 使用 `draw_*` 指令
- **声明式表达**（如"H 是 A 到 BC 的垂足"）→ 使用宏指令

### 8.1 中线

```text
median(M, A, B, C)
```

**展开为**：

```text
midpoint(M, BC)
segment(AM)
```

**语义**：M 是 BC 的中点，AM 是从顶点 A 到对边中点的中线

### 8.2 高

```text
altitude(H, A, B, C)
```

**展开为**：

```text
foot(H, A, BC)
segment(AH)
perpendicular(AH, BC)
```

**语义**：H 是从 A 到 BC 的垂足，AH 是高

### 8.3 角平分线

```text
angle_bisector(P, A, B, C)
```

**展开为**：

```text
on(P, bisector_ABC)
equal_angle(ABP, PBC)
```

**语义**：P 在 ∠ABC 的平分线上

### 8.4 垂直平分线约束

#### 8.4.1 点在垂直平分线上

```text
perpendicular_bisector(P, AB)
```

**语义**：点 P 在线段 AB 的垂直平分线上

**约束条件**：|PA| = |PB|（P 到 A、B 的距离相等）

**对应中文**：「点 P 在 AB 的垂直平分线上」

#### 8.4.2 线段是中垂线 (v1.8.2)

```text
perpendicular_bisector(AB, CD)
```

**语义**：线段 AB 是 CD 的垂直平分线（AB 垂直平分 CD）

**约束条件**：
1. CD 的中点 M 在直线 AB 上（A、B、M 共线）
2. AB ⟂ CD（向量点积为 0）

**对应中文**：「AB 是 CD 的中垂线」「AB 垂直平分 CD」

**示例**：
```text
triangle(A, B, C)
perpendicular_bisector(DE, BC)   # DE 是 BC 的中垂线
```

### 8.5 特殊三角形宏展开表

| 宏指令                                  | 展开结果                                                         |
| --------------------------------------- | ---------------------------------------------------------------- |
| `isosceles(A, B, C)`                  | `triangle(A, B, C)` + `equal(AB, AC)`                        |
| `equilateral(A, B, C)`                | `triangle(A, B, C)` + `equal(AB, BC)` + `equal(BC, CA)`    |
| `isosceles_right(A, B, C)`            | `triangle(A, B, C)` + `equal(AB, AC)` + `right_angle(BAC)` |
| `right_triangle(A, B, C, right_at=B)` | `triangle(A, B, C)` + `perpendicular(AB, CB)`                |

---

## 9. 圆与弧（Circles & Arcs）

### 9.1 圆的声明

```text
circle(O)                   # 圆心为 O 的圆
circle(O, r=5)              # 指定半径
circle(O, radius=AB)        # 半径等于线段 AB
```

### 9.2 点与圆的关系

```text
on(A, circle_O)             # 点 A 在圆 O 上
tangent(A, circle_O)        # 点 A 是圆 O 的切点
```

### 9.3 弧

```text
arc(A, B, circle_O)         # 圆 O 上从 A 到 B 的弧（声明）
on(E, arc_AB)               # 点 E 在弧 AB 上（需先声明 arc(A, B, circle_O)）
```

**弧上点约束说明**：

- `arc_AB` 引用格式表示从 A 到 B 的弧（逆时针方向的劣弧）
- 使用前必须先声明 `arc(A, B, circle_O)` 以关联圆心
- 约束等价于：点在圆上 + 角度在弧范围内

### 9.4 扇形声明 (v1.8.0)

```text
sector(O, A, B)             # 以 O 为圆心，从 OA 到 OB 的扇形
sector(O, A, B, angle=120)  # 指定圆心角 120°
sector(O, A, B, r=5)        # 指定半径 5
```

**语义**：声明一个扇形区域，用于渲染和约束。

**隐含约束**：

| 语法形式 | 生成的约束 |
|---------|-----------|
| `sector(O, A, B)` | `equal(OA, OB)` |
| `sector(O, A, B, angle=120)` | `equal(OA, OB)` + `angle(AOB)=120` |
| `sector(O, A, B, r=5)` | `length(OA)=5` + `length(OB)=5` |

**对应中文**：「以 O 为圆心、OA 和 OB 为边的扇形」「扇形 OAB」

**与消歧语法的区别**：

| 语法 | 参数数量 | 用途 | 示例 |
|-----|---------|------|------|
| `sector(O, A, B, ...)` | 3个点 + 可选参数 | 扇形声明（用于渲染） | `sector(O, A, B, angle=60)` |
| `sector(P, O, A, B)` | 4个点 | 消歧约束（P在扇形内） | `sector(P, O, A, B)` |

**注意**：
- 扇形声明 `sector(O, A, B)` 会自动引入 `|OA| = |OB|` 约束
- 扇形引用使用 `sector_OAB` 格式（如 `shaded(sector_OAB)`）
- 消歧语法 `sector(P, O, A, B)` 见第 14.3.3 节

### 9.5 内接与外接

```text
inscribed(ABC, circle_O)    # △ABC 内接于圆 O
circumscribed(circle_O, ABC) # 圆 O 外接于 △ABC
```

### 9.6 半圆

```text
semicircle(O, A, B)         # 以O为圆心、AB为直径的半圆
```

**语义**: 声明一个以 O 为圆心、AB 为直径的半圆

**隐含约束**:

- `midpoint(O, AB)` - O 是 AB 的中点
- `circle(O, radius=OA)` - 创建以 O 为圆心、OA 为半径的圆
- `on(A, circle_O)` + `on(B, circle_O)` - A、B 在圆上

**对应中文**: 「以点 O 为圆心、AB 为直径的半圆」

**使用场景**: 当几何题目明确提到"半圆"且给出直径端点时使用

**示例**:

```text
triangle(A, B, C)
semicircle(O, A, B)         # AB 为直径的半圆
on(D, circle_O)             # 点 D 在半圆上
```

**注意**:

- 半圆在约束求解时等同于完整的圆，区别仅在于语义表达
- 如需限制点只在半圆弧上（而非完整圆），需额外添加角度或位置约束

---

## 10. 角度与长度表达

### 10.1 角度

```text
angle(ABC)=40               # ∠ABC = 40°
angle(ABC)=90               # ∠ABC = 90°
right_angle(ABC)            # ∠ABC 是直角（等价于 angle(ABC)=90）
```

### 10.2 线段长度

```text
length(AB)=5                # AB 的长度为 5
length(AB)=length(CD)       # AB 和 CD 等长（等价于 equal(AB, CD)）
```

### 10.2.1 数学表达式支持 (v1.6.9)

DSL 支持在数值参数位置使用数学表达式，Parser 阶段自动求值为小数。

#### 支持的表达式类型

| 类型 | 示例 | 求值结果 |
|------|------|---------|
| 分数 | `1/2`, `-7/3`, `5/4` | `0.5`, `-2.333...`, `1.25` |
| 根号 | `√3`, `sqrt(5)`, `2√5` | `1.732...`, `2.236...`, `4.472...` |
| 混合表达式 | `(1+√3)/2`, `2√5/3` | `1.366...`, `1.490...` |
| 基础运算 | `2+3`, `5-1`, `2*3` | `5`, `4`, `6` |

#### 使用示例

```text
# 分数长度
length(AB)=1/2              # AB 长度为 0.5
length(CD)=-7/3             # CD 长度为 -2.333...

# 根号长度
length(AB)=√3               # AB 长度为 1.732...
length(CD)=2√5              # CD 长度为 4.472...

# 混合表达式
length(AB)=(1+√3)/2         # AB 长度为 1.366...
point_coord(A, 2√5/3, 0)    # A 坐标为 (1.490..., 0)

# 角度表达式
angle(ABC)=180/3            # ∠ABC = 60°
angle(DEF)=√3*30            # ∠DEF ≈ 51.96°

# 比例表达式
ratio(AB, CD)=√2            # AB:CD = √2:1 ≈ 1.414:1
```

#### 适用范围

数学表达式可用于**所有数值参数**位置：

- 长度约束：`length(AB)=√3`
- 角度约束：`angle(ABC)=180/3`
- 坐标值：`point_coord(A, 1/2, √3)`
- 比例系数：`ratio(AB, CD)=√2`
- 圆半径：`circle(O, radius=2√5)` (当radius为数值时)
- 旋转角度：`rotate(D, source=C, center=O, angle=180/3)`

#### 注意事项

1. **求值时机**：表达式在 Parser 阶段立即求值为 `float`，不保留符号形式
2. **求值精度**：使用 SymPy 求值，精度约 15 位小数
3. **Unicode 支持**：同时支持 `√`（Unicode U+221A）和 `sqrt()` 函数
4. **向后兼容**：原有纯数值 DSL（`3.14`, `0.5`）完全不受影响
5. **错误处理**：无效表达式（如 `1/0`）会触发 `ParseError`
6. **运算符优先级**：遵循标准数学优先级（括号 > 乘除 > 加减）

#### 符号约定

| 符号 | 含义 | 示例 |
|------|------|------|
| `+` | 加法 | `1+2` → `3` |
| `-` | 减法/负号 | `5-1` → `4`, `-3` → `-3` |
| `*` | 乘法 | `2*3` → `6`, `2√5` → `4.472...` |
| `/` | 除法 | `1/2` → `0.5`, `2√5/3` → `1.490...` |
| `√` | 平方根 | `√3` → `1.732...` |
| `sqrt()` | 平方根函数 | `sqrt(5)` → `2.236...` |
| `()` | 分组 | `(1+√3)/2` → `1.366...` |

#### 对应中文表达

| 中文 | DSL（v1.6.9+，推荐） | DSL（旧式，仍支持） |
|------|---------------------|-------------------|
| AB 长为二分之一 | `length(AB)=1/2` | `length(AB)=0.5` |
| AB 长为根号 3 | `length(AB)=√3` | `length(AB)=1.732` |
| AB 长为 2 倍根号 5 | `length(AB)=2√5` | `length(AB)=4.472` |
| 点 B 横坐标为负三分之七 | `point_coord(B, -7/3, y)` | `point_coord(B, -2.333, y)` |
| 半径为根号 2 | `circle(O, radius=√2)` | `circle(O, radius=1.414)` |

#### 实现细节

**表达式求值流程**：

```
DSL 输入: length(AB)=2√5/3
    ↓
Lexer: [LENGTH, LPAREN, IDENTIFIER(AB), RPAREN, EQUALS, EXPR("2√5/3")]
    ↓
Parser.parse_number(): 
    → evaluate_expression("2√5/3")
    → SymPy: "2√5/3" → "2*sqrt(5)/3" → 1.4907119849998598
    ↓
AST: LengthConstraint(seg=('A','B'), value=1.4907119849998598)
    ↓
Solver: 使用 float 值进行约束求解
```

**与角度比例约束的区别**：

| 语法 | 用途 | 求值时机 | 示例 |
|------|------|---------|------|
| `angle(AOB)=2*angle(BOC)` | 角度比例关系 | 求解时动态计算 | 角度间存在倍数关系 |
| `angle(AOB)=2*30` | 数学表达式 | 解析时求值为 `60.0` | 等价于 `angle(AOB)=60` |

**注意**：`2*angle(BOC)` 中的 `*` 表示"角度比例"（运行时关系），而 `2*30` 中的 `*` 表示"算术乘法"（编译时常量）。

### 10.3 比例关系

```text
ratio(AB, CD)=2             # AB : CD = 2 : 1
ratio(AB, CD)=0.5           # AB : CD = 1 : 2
```

### 10.3.1 角度比例约束 (v1.6.7)

```text
angle(AOB)=0.3333*angle(BOC)     # ∠AOB = (1/3) * ∠BOC
angle(ABC)=2*angle(DEF)          # ∠ABC = 2 * ∠DEF
angle(P1)=0.5*angle(P2)          # ∠P1 = 0.5 * ∠P2
```

**语义**：一个角度等于另一个角度的若干倍。

**约束**：∠AOB = k × ∠BOC (k为比例系数)

**对应中文**：「∠AOB=(1/3)∠BOC」「∠ABC是∠DEF的2倍」

**注意**:

- 比例系数k必须是数值字面量（不支持符号变量）
- v1.6.9+ 支持分数表达式：`(1/3)` 或 `0.3333` 均可
- 比例可以是任意正实数

### 10.4 角度单位约定（v1.6）

本 DSL 中角度参数的单位约定如下：

| 上下文                         | 单位                          | 示例                             |
| ------------------------------ | ----------------------------- | -------------------------------- |
| `angle(ABC)=N`               | **度** (degree)         | `angle(ABC)=60` 表示 60°      |
| `rotate(..., angle=N)`       | **度** (degree)         | `angle=90` 表示逆时针旋转 90° |
| `sin_function(f, A, w, phi)` | phi 为**弧度** (radian) | `phi=1.5708` ≈ π/2           |
| `cos_function(f, A, w, phi)` | phi 为**弧度** (radian) | 同上                             |

**规则**：几何约束中的角度统一使用**度**，三角函数相位参数使用**弧度**。

### 10.5 角度的三角函数值约束 (v1.6.4)

```text
sin(ABC)=0.5          # ∠ABC 的正弦值为 0.5
cos(ABC)=0.866        # ∠ABC 的余弦值为 0.866（≈ √3/2）
tan(ABC)=1            # ∠ABC 的正切值为 1（即 45°）
```

**语义**：约束角度的三角函数值，而非角度本身。适用于题目直接给出三角函数值的情况。

**参数范围**：

- `sin` 和 `cos`：值必须在 [-1, 1] 范围内
- `tan`：值可以是任意实数（除接近 ±90° 时）

**对应中文**：

- 「tan∠ABC = 1/2」→ `tan(ABC)=0.5`
- 「cos∠ABC = √3/2」→ `cos(ABC)=0.866`
- 「sin∠ABC = 1/2」→ `sin(ABC)=0.5`

**与 angle(ABC)=N 的区别**：

| 语法              | 含义                     | 示例                               |
| ----------------- | ------------------------ | ---------------------------------- |
| `angle(ABC)=60` | 直接约束角度为 60°      | ∠ABC = 60°                       |
| `cos(ABC)=0.5`  | 约束角度使其余弦值为 0.5 | cos∠ABC = 0.5（即 60°）          |
| `sin(ABC)=0.5`  | 约束角度使其正弦值为 0.5 | sin∠ABC = 0.5（即 30° 或 150°） |
| `tan(ABC)=1`    | 约束角度使其正切值为 1   | tan∠ABC = 1（即 45°）            |

---

## 11. 点的位置关系（高级）

### 11.1 中点

```text
midpoint(M, AB)             # M 是 AB 的中点
```

### 11.2 三等分点

```text
trisection(C, AB, near=A)   # C 是 AB 的三等分点，靠近 A
trisection(D, AB, near=B)   # D 是 AB 的三等分点，靠近 B
```

### 11.3 垂足

```text
foot(H, A, BC)              # H 是从点 A 到直线 BC 的垂足
foot(H, A, BC_extended)     # H 是从点 A 到 BC 延长线的垂足（v1.7.1）
```

**语义说明**：

- `foot(H, A, BC)`: BC 被解释为"过 B、C 两点的直线"（与第 16 节规则一致）
- `foot(H, A, BC_extended)`: 语义等价，但明确标注为延长线（提高可读性）
- 约束求解行为完全相同（line 始终视为无限直线）

**对应中文**：

- 「H 是 A 到 BC 的垂足」→ `foot(H, A, BC)`
- 「H 是 A 到 BC 延长线的垂足」→ `foot(H, A, BC_extended)`

### 11.4 交点

```text
intersection(P, AB, CD)              # P 是直线 AB 与 CD 的交点
intersection(P, l1, l2)              # P 是直线 l1 与 l2 的交点
intersection(P, AB, circle_O)        # P 是直线 AB 与圆 O 的交点（v1.6）
intersection(P, AB, circle_O, index=0)  # 多交点时指定第一个交点
intersection(P, circle_O1, circle_O2)   # P 是两圆的交点（v1.6）
intersection(P, AB, CD_extended)     # P 是 AB 与 CD 延长线的交点（v1.8.1）
intersection(P, AB_extended, CD)     # P 是 AB 延长线与 CD 的交点（v1.8.1）
intersection(P, AB_extended, CD_extended)  # 两条延长线的交点（v1.8.1）
```

**注意**：
- 直线与圆、圆与圆可能有 0、1 或 2 个交点。使用 `index=0` 或 `index=1` 指定所需交点。
- `_extended` 后缀表示线段的延长线（即过两点的直线），语义上表明交点可能在线段本身之外。

### 11.5 弧中点 (v1.6.6)

```text
arc_midpoint(D, BC, circle_O)                    # D是圆O上劣弧BC的中点
arc_midpoint(D, BC, circle_O, arc_type='minor')  # D是劣弧BC的中点（显式）
arc_midpoint(D, BC, circle_O, arc_type='major')  # D是优弧BC的中点
```

**语义**：点D是圆O上连接B、C两点的弧的中点，即圆心角∠BOD = ∠COD。

**约束说明**：

- `arc_type='minor'`（默认）：劣弧（小于180°的弧）
- `arc_type='major'`：优弧（大于180°的弧）
- 等价约束：`on(D, circle_O)` + `equal_angle(BOD, COD)` + 弧方向约束

**对应中文**：「点D是劣弧BC的中点」「D平分弧BC」

### 11.6 延长线上的点

```text
on(E, AB_extended)          # E 在 AB 的延长线上（即 B 在 A、E 之间）
on(F, BA_extended)          # F 在 BA 的延长线上（即 A 在 B、F 之间）
```

---

## 12. 几何变换与折叠操作

### 12.1 基本几何变换

```text
rotate(D, source=C, center=O, angle=60)      # D 是 C 绕 O 逆时针旋转 60° 的结果
translate(P, dx=2, dy=3)                     # 点 P 坐标沿x轴方向平移2，沿y轴方向平移3
scale(P, center=O, factor=2)                 # 点 P 以 O 为中心放大 2 倍
reflect(D, C, AB)                            # D 是 C 关于直线 AB 的反射点（v1.5.2）
```

**rotate 语法说明**：

- `rotate(D, source=C, center=O, angle=N)` - **唯一有效格式** - D 是 C 绕 O 逆时针旋转 N° 的结果
- 旋转角度为正时表示逆时针方向，为负时表示顺时针方向
- **已移除**：`rotate(P, center=O, angle=N)` 格式（v1.5.3 起不再支持）

**reflect 语法说明** (v1.5.2 新增)：

- `reflect(P', P, AB)` - P' 是点 P 关于直线 AB 的反射点
- 反射满足：P 和 P' 到直线 AB 的距离相等，PP' 垂直于 AB
- 也可用于表示翻折操作后的点位置

### 12.2 折叠操作

折叠是中国几何题中常见的操作类型，用于表示纸张折叠后的点位置变换：

```text
fold(P, P_prime, line)           # 点 P 沿折痕 line 折叠后到达 P_prime 位置
```

**示例**：

```text
fold(C, C_prime, EF)             # 点 C 沿折痕 EF 折叠到 C'
fold(B, B_prime, EF)             # 点 B 沿折痕 EF 折叠到 B'
```

**隐含约束**：

- P 和 P_prime 关于折痕 line 对称
- `perpendicular(PP_prime, line)`
- P 到 line 的距离等于 P_prime 到 line 的距离

**与 reflect 的关系**：

`fold` 和 `reflect` 在几何上等价，但参数顺序不同：

| 指令                              | 参数顺序    | 语义风格                         |
| --------------------------------- | ----------- | -------------------------------- |
| `fold(源点, 目标点, 折痕)`      | P, P', line | 强调操作过程（从哪里折到哪里）   |
| `reflect(目标点, 源点, 对称轴)` | P', P, line | 强调结果声明（目标点是源点的像） |

**等价关系**：`fold(C, C_prime, EF)` ≡ `reflect(C_prime, C, EF)`

**推荐使用场景**：

- 描述折纸、翻折操作时使用 `fold`
- 描述几何变换、对称关系时使用 `reflect`

---

## 13. 坐标系与函数图像（Coordinate & Functions）

本节定义坐标系和函数图像相关的 DSL 指令，用于支持解析几何类题目的标注。

### 13.1 坐标系声明

```text
coordinate_system(O)              # 以 O 为原点的平面直角坐标系
```

**说明**：

- `coordinate_system(O)` 隐式声明原点 O 的坐标为 (0, 0)
- 隐式建立 x 轴和 y 轴（无需显式声明）
- 坐标轴上的点可通过 `point_coord` 或共线约束表示
- 原点 O 通常用大写字母 O 表示

**坐标轴上的点约束**：

若需约束点在坐标轴上，可使用 `on(P, axis_x)` 或 `on(P, axis_y)` 语法（见第 5.6 节）：

```text
on(A, axis_x)                     # 点 A 在 x 轴上（A.y = 0）
on(B, axis_y)                     # 点 B 在 y 轴上（B.x = 0）
```

**对应中文**：

- 「点 A 在 x 轴上」→ `on(A, axis_x)`
- 「点 B 在 y 轴上」→ `on(B, axis_y)`
- 「直线 l 与 x 轴交于点 A」→ `on(A, axis_x)` + `on(A, l)` 或 `axis_intersection(A, l, x)`

**辅助指令**（通常不需要显式使用）：

```text
axis_x(O)                         # x 轴（经过原点 O 的水平轴）
axis_y(O)                         # y 轴（经过原点 O 的垂直轴）
```

**注意**：`axis_x(O)` 和 `axis_y(O)` 为辅助指令，大多数情况下直接使用 `coordinate_system(O)` 即可，无需显式声明坐标轴。

### 13.2 坐标点声明

```text
point_coord(A, 2, 3)              # 点 A 的坐标为 (2, 3)
point_coord(B, -1, 0)             # 点 B 的坐标为 (-1, 0)
point_coord(O, 0, 0)              # 原点 O 的坐标为 (0, 0)
```

**对应中文**：「点 A 的坐标为 (2, 3)」「A(2, 3)」

### 13.3 基础函数图像声明

#### 13.3.1 一次函数与二次函数

```text
line_function(l, k, b)            # 直线函数 y = kx + b，命名为 l
line_function(l, -2, 8)           # 直线 y = -2x + 8

parabola(p, a, b, c)              # 抛物线 y = ax² + bx + c，命名为 p
parabola(p, 1, 0, 0)              # 抛物线 y = x²
parabola(p, -1, 2, 3)             # 抛物线 y = -x² + 2x + 3
```

#### 13.3.2 幂函数（Power Functions）

```text
power_function(f, n)              # y = x^n
power_function(f, 2)              # y = x² (等价于 parabola(f, 1, 0, 0))
power_function(f, 3)              # y = x³
power_function(f, 0.5)            # y = x^(1/2) = √x
power_function(f, -1)             # y = x^(-1) = 1/x (等价于 hyperbola)
power_function(f, -2)             # y = x^(-2) = 1/x²
```

**对应中文**：

- 「幂函数 y = x³」→ `power_function(f, 3)`
- 「根号函数 y = √x」→ `power_function(f, 0.5)`

#### 13.3.3 反比例函数

```text
hyperbola(h, k)                   # 反比例函数 y = k/x，命名为 h
hyperbola(h, 8)                   # 反比例函数 y = 8/x
hyperbola(h, -2)                  # 反比例函数 y = -2/x
```

#### 13.3.4 指数函数（Exponential Functions）

```text
exp_function(f, a)                # y = a^x (a > 0, a ≠ 1)
exp_function(f, e)                # y = e^x (自然指数，e ≈ 2.718)
exp_function(f, 2)                # y = 2^x
exp_function(f, 0.5)              # y = (1/2)^x = 0.5^x
```

**对应中文**：

- 「指数函数 y = e^x」→ `exp_function(f, e)`
- 「指数函数 y = 2^x」→ `exp_function(f, 2)`

#### 13.3.5 对数函数（Logarithmic Functions）

```text
log_function(f, a)                # y = log_a(x) (a > 0, a ≠ 1)
log_function(f, e)                # y = ln(x) (自然对数)
log_function(f, 10)               # y = lg(x) (常用对数)
log_function(f, 2)                # y = log₂(x)
```

**对应中文**：

- 「对数函数 y = ln(x)」→ `log_function(f, e)`
- 「对数函数 y = lg(x)」→ `log_function(f, 10)`

#### 13.3.6 三角函数（Trigonometric Functions）

```text
sin_function(f, A, w, phi)        # y = A·sin(ωx + φ)
sin_function(f, 1, 1, 0)          # y = sin(x)
sin_function(f, 2, 1, 0)          # y = 2sin(x)
sin_function(f, 1, 2, 0)          # y = sin(2x)
sin_function(f, 1, 1, 1.57)       # y = sin(x + π/2)

cos_function(f, A, w, phi)        # y = A·cos(ωx + φ)
cos_function(f, 1, 1, 0)          # y = cos(x)

tan_function(f)                   # y = tan(x)
cot_function(f)                   # y = cot(x)
sec_function(f)                   # y = sec(x) = 1/cos(x)
csc_function(f)                   # y = csc(x) = 1/sin(x)
```

**对应中文**：

- 「正弦函数 y = sin(x)」→ `sin_function(f, 1, 1, 0)`
- 「余弦函数 y = cos(x)」→ `cos_function(f, 1, 1, 0)`
- 「正切函数 y = tan(x)」→ `tan_function(f)`
- 「y = 2sin(2x + π/3)」→ `sin_function(f, 2, 2, 1.047)`

#### 13.3.7 反三角函数（Inverse Trigonometric Functions）

```text
arcsin_function(f)                # y = arcsin(x)，定义域 [-1, 1]
arccos_function(f)                # y = arccos(x)，定义域 [-1, 1]
arctan_function(f)                # y = arctan(x)
arccot_function(f)                # y = arccot(x)
```

**对应中文**：

- 「反正弦函数 y = arcsin(x)」→ `arcsin_function(f)`
- 「反正切函数 y = arctan(x)」→ `arctan_function(f)`

### 13.4 通用函数表达式

用于表示复合函数和不常见的函数形式：

```text
function_expr(f, "x*ln(x)")       # y = x·ln(x)
function_expr(f, "ln(x)/x")       # y = ln(x)/x
function_expr(f, "e^x + e^(-x)")  # y = e^x + e^(-x) (双曲余弦相关)
function_expr(f, "x^2 * sin(x)")  # y = x²·sin(x)
function_expr(f, "sqrt(1-x^2)")   # y = √(1-x²) (上半圆)
function_expr(f, "x^3 - 3*x")     # y = x³ - 3x
```

**表达式语法规则**：

- **变量**：`x`
- **运算符**：`+`, `-`, `*`, `/`, `^`（幂运算）
- **内置函数**：`sin`, `cos`, `tan`, `cot`, `sec`, `csc`, `arcsin`, `arccos`, `arctan`, `ln`, `log`, `exp`, `sqrt`, `abs`, `floor`, `ceil`
- **常数**：`e`（自然常数）, `pi`（圆周率）, 数字

**对应中文**：

- 「y = x·ln(x)」→ `function_expr(f, "x*ln(x)")`
- 「y = ln(x)/x」→ `function_expr(f, "ln(x)/x")`

### 13.5 函数变换

支持函数的平移、伸缩、对称变换：

```text
translate_function(f, g, dx, dy)   # g 是 f 沿 x 方向平移 dx，y 方向平移 dy
translate_function(f, g, 2, 0)     # g = f(x-2)，向右平移 2 个单位
translate_function(f, g, 0, 3)     # g = f(x) + 3，向上平移 3 个单位

scale_function(f, g, sx, sy)       # g 是 f 沿 x 方向伸缩 sx 倍，y 方向伸缩 sy 倍
scale_function(f, g, 2, 1)         # g = f(x/2)，横向拉伸 2 倍
scale_function(f, g, 1, 2)         # g = 2f(x)，纵向拉伸 2 倍

reflect_function(f, g, axis)       # g 是 f 关于 axis 的对称图像
reflect_function(f, g, x)          # g = -f(x)，关于 x 轴对称
reflect_function(f, g, y)          # g = f(-x)，关于 y 轴对称
reflect_function(f, g, origin)     # g = -f(-x)，关于原点对称
```

**对应中文**：

- 「将 f(x) 向右平移 2 个单位」→ `translate_function(f, g, 2, 0)`
- 「将 f(x) 关于 y 轴对称」→ `reflect_function(f, g, y)`

### 13.6 函数与几何的交互

```text
function_point(P, f)              # 点 P 在函数 f 的图像上
function_intersection(P, f1, f2)  # P 是函数 f1 与 f2 的交点
axis_intersection(A, f, x)        # A 是函数 f 与 x 轴的交点
axis_intersection(B, f, y)        # B 是函数 f 与 y 轴的交点
tangent_point(C, l, f)            # C 是直线 l 与曲线 f 的切点
tangent_line(l, f, P)             # l 是曲线 f 在点 P 处的切线
asymptote(l, f)                   # l 是函数 f 的渐近线
```

**对应中文**：

- 「直线 l 与 x 轴交于点 A」→ `axis_intersection(A, l, x)`
- 「直线 l 与反比例函数图像相切于点 C」→ `tangent_point(C, l, h)`
- 「曲线 f 在点 P 处的切线」→ `tangent_line(l, f, P)`

**参数顺序设计说明**：

| 指令                       | 参数顺序       | 设计原则                               |
| -------------------------- | -------------- | -------------------------------------- |
| `tangent_point(C, l, f)` | 点, 直线, 曲线 | 结果（切点）在前                       |
| `tangent_line(l, f, P)`  | 直线, 曲线, 点 | 结果（切线）在前，参考位置（切点）在后 |

### 13.7 坐标系中的几何图形

在坐标系中，可以结合坐标点与几何图形：

```text
coordinate_system(O)
point_coord(A, 0, 4)
point_coord(B, 4, 0)
point_coord(C, 1, 2)
triangle(O, A, B)
on(C, AB)
```

### 13.8 中文表达对照（坐标系与函数相关）

| 中文表达                   | DSL 转换                             |
| -------------------------- | ------------------------------------ |
| 在平面直角坐标系中         | `coordinate_system(O)`             |
| 点 A 的坐标为 (2, 3)       | `point_coord(A, 2, 3)`             |
| **基础函数**         |                                      |
| 一次函数 y = kx + b        | `line_function(l, k, b)`           |
| 二次函数 y = ax² + bx + c | `parabola(p, a, b, c)`             |
| 反比例函数 y = k/x         | `hyperbola(h, k)`                  |
| **幂函数**           |                                      |
| 幂函数 y = x³             | `power_function(f, 3)`             |
| 根号函数 y = √x           | `power_function(f, 0.5)`           |
| **指数与对数函数**   |                                      |
| 指数函数 y = e^x           | `exp_function(f, e)`               |
| 指数函数 y = 2^x           | `exp_function(f, 2)`               |
| 对数函数 y = ln(x)         | `log_function(f, e)`               |
| 对数函数 y = lg(x)         | `log_function(f, 10)`              |
| **三角函数**         |                                      |
| 正弦函数 y = sin(x)        | `sin_function(f, 1, 1, 0)`         |
| 余弦函数 y = cos(x)        | `cos_function(f, 1, 1, 0)`         |
| 正切函数 y = tan(x)        | `tan_function(f)`                  |
| y = 2sin(2x + π/3)        | `sin_function(f, 2, 2, 1.047)`     |
| **反三角函数**       |                                      |
| 反正弦函数 y = arcsin(x)   | `arcsin_function(f)`               |
| 反余弦函数 y = arccos(x)   | `arccos_function(f)`               |
| 反正切函数 y = arctan(x)   | `arctan_function(f)`               |
| **复合函数**         |                                      |
| y = x·ln(x)               | `function_expr(f, "x*ln(x)")`      |
| y = ln(x)/x                | `function_expr(f, "ln(x)/x")`      |
| y = e^x + e^(-x)           | `function_expr(f, "e^x + e^(-x)")` |
| **函数变换**         |                                      |
| 将 f(x) 向右平移 2 个单位  | `translate_function(f, g, 2, 0)`   |
| 将 f(x) 向上平移 3 个单位  | `translate_function(f, g, 0, 3)`   |
| 将 f(x) 关于 y 轴对称      | `reflect_function(f, g, y)`        |
| **函数与几何交互**   |                                      |
| 与 x 轴交于点 A            | `axis_intersection(A, f, x)`       |
| 与 y 轴交于点 B            | `axis_intersection(B, f, y)`       |
| 图像经过点 P               | `function_point(P, f)`             |
| 两函数图像相交于点 Q       | `function_intersection(Q, f1, f2)` |
| 曲线 f 在点 P 处的切线     | `tangent_line(l, f, P)`            |

#### 13.8.1 函数点坐标约束 (v1.7.0)

```text
function_point_constrained(P, f, x=val)  # 点 P 在函数 f 上，且 P.x = val
function_point_constrained(P, f, y=val)  # 点 P 在函数 f 上，且 P.y = val
```

**语义**：点在函数图像上，且某一坐标（x 或 y）的值已知。

**适用场景**：
- 「点 P 在抛物线上，纵坐标为 -0.5」→ `function_point_constrained(P, p, y=-0.5)`
- 「点 Q 在双曲线上，横坐标为 2」→ `function_point_constrained(Q, h, x=2)`
- 「第三象限内抛物线上的点，纵坐标为 -3」→ `function_point_constrained(P, p, y=-3)` + `inside(P, half_plane_...)`

**与 function_point 的区别**：

| 约束类型 | 语法 | 语义 | 初始化 |
|---------|------|------|--------|
| `function_point` | `function_point(P, f)` | P 在 f 上，位置完全未知 | 需要其他约束确定位置 |
| `function_point_constrained` | `function_point_constrained(P, f, y=-0.5)` | P 在 f 上，y 坐标已知 | 自动求解 f(x) = -0.5 |

**约束逻辑**：

```
function_point_constrained(P, f, y=y0):
  1. 强约束：P.y = y0
  2. 强约束：P.y = f(P.x)
  3. 初始化：求解 f(x) = y0，得到 P.x 的初始值

function_point_constrained(P, f, x=x0):
  1. 强约束：P.x = x0
  2. 强约束：P.y = f(P.x)
  3. 初始化：直接计算 P.y = f(x0)
```

**注意事项**：

1. **多解情况**：如果 f(x) = y0 有多个解（如抛物线），Solver 会选择最接近 0 的解
   - 可通过额外约束（如 `inside()`, `x > 0` 等）消除歧义
   
2. **无解情况**：如果 f(x) = y0 无解，初始化会失败，求解可能不收敛
   - 确保 y0 在函数值域内

3. **数值表达式**：坐标值支持数学表达式（v1.6.9+）
   ```text
   function_point_constrained(P, p, y=-1/2)     # 分数
   function_point_constrained(Q, f, x=√3)       # 根号
   function_point_constrained(R, h, y=2√5/3)    # 混合表达式
   ```

4. **向后兼容**：现有 `function_point(P, f)` 语法保持不变，无需迁移

**中文表达对照**：

| 中文 | DSL (v1.7.0+) |
|------|---------------|
| P 在抛物线上，纵坐标为 -0.5 | `function_point_constrained(P, p, y=-0.5)` |
| Q 在双曲线上，横坐标为 2 | `function_point_constrained(Q, h, x=2)` |
| 第三象限内抛物线上一点 P，y=-3 | `function_point_constrained(P, p, y=-3)` |
| P 在直线上，过点 (2, f(2)) | `function_point_constrained(P, l, x=2)` |

---

## 14. 模糊语义消歧指令（Disambiguation）

本节定义用于消除中文几何题中常见模糊表达的 DSL 指令。这是 v1.4 版本的核心创新，旨在解决 LLM 从自然语言到 DSL 解析时遇到的歧义问题。

### 14.1 设计动机

中文几何题中存在大量模糊表达，例如：

- 「点 P 在 AB 的延长线上」→ 延长线方向不明确（向 A 还是向 B）
- 「在△ABC 内取一点 P」→ 点的具体位置不明确
- 「在角平分线上取一点 Q」→ 靠近哪个端点不明确
- 「在 AB 上任取一点」→ 需要生成默认位置

这些模糊表达会导致：

1. LLM 无法可靠地进行语义解析
2. Solver 无法确定初始点位置
3. 渲染结果可能与题意不符

因此需要引入**消歧指令**，将模糊语义转化为明确的几何约束。

### 14.2 线段方向与位置指令

#### 14.2.1 方向指示 `direction`

```text
direction(P, AB, towards=A)      # P 位于 AB 上，且靠近 A 端
direction(P, AB, towards=B)      # P 位于 AB 上，且靠近 B 端
```

**语义**：用于消歧点在线段上的相对位置。

- `towards=A` 表示点 P 在 AB 的前半部分（距离 A 更近）
- `towards=B` 表示点 P 在 AB 的后半部分（距离 B 更近）

**对应中文**：

- 「P 在 AB 上靠近 A 的位置」→ `direction(P, AB, towards=A)`
- 「P 在 AB 上靠近端点 B」→ `direction(P, AB, towards=B)`

#### 14.2.2 延长线方向 `on_extended`

```text
on_extended(P, AB, towards=B)    # P 在 AB 延长线上（过 B 向外延伸）
on_extended(P, AB, towards=A)    # P 在 BA 延长线上（过 A 向外延伸）
```

**语义**：明确指定延长线的方向。

- `towards=B` 表示从 A 经过 B 继续延伸，P 在 B 的外侧
- `towards=A` 表示从 B 经过 A 继续延伸，P 在 A 的外侧

**对应中文**：

- 「延长 AB 至点 P」→ `on_extended(P, AB, towards=B)`
- 「延长 BA 至点 P」→ `on_extended(P, AB, towards=A)`
- 「P 在 AB 的延长线上」→ 需要根据上下文判断 `towards=A` 或 `towards=B`

**与 `on(P, AB_extended)` 的区别**：

- `on(P, AB_extended)` 不明确延长方向（旧版语法，保留兼容）
- `on_extended(P, AB, towards=B)` 明确指定延长方向（推荐使用）

#### 14.2.3 射线声明 `ray`

```text
ray(A, B)                        # 从点 A 出发，经过点 B 的射线
```

**语义**：声明一条射线，起点为 A，方向由 B 确定。

**对应中文**：

- 「射线 AB」→ `ray(A, B)`
- 「从 A 出发经过 B 的射线」→ `ray(A, B)`

### 14.3 区域内外关系

#### 14.3.1 内部 `inside`

```text
inside(P, triangle_ABC)          # 点 P 在 △ABC 内部
inside(P, circle_O)              # 点 P 在 ⊙O 内部
inside(P, sector_AOB)            # 点 P 在扇形 AOB 内部
inside(P, rectangle_ABCD)        # 点 P 在矩形 ABCD 内部 (v1.6.3)
inside(P, parallelogram_ABCD)    # 点 P 在平行四边形 ABCD 内部 (v1.6.3)
inside(P, square_ABCD)           # 点 P 在正方形 ABCD 内部 (v1.6.3)
inside(P, quadrilateral_ABCD)    # 点 P 在四边形 ABCD 内部 (v1.6.3)
inside(P, polygon_ABCDEF)        # 点 P 在多边形 ABCDEF 内部 (v1.6.3)
```

**语义**：声明点在某个区域的内部。Extractor 会据此推断初始点坐标。

**对应中文**：

- 「在 △ABC 内取一点 P」→ `inside(P, triangle_ABC)`
- 「点 P 在圆 O 内」→ `inside(P, circle_O)`
- 「点 P 在矩形 ABCD 内」→ `inside(P, rectangle_ABCD)`
- 「点 P 在四边形 ABCD 内」→ `inside(P, quadrilateral_ABCD)`

**Solver 处理**：

- 不作为硬约束
- 用于推断初始点位置（如三角形重心附近）
- 可选：添加 soft constraint 验证

#### 14.3.2 外部 `outside`

```text
outside(P, triangle_ABC)         # 点 P 在 △ABC 外部
outside(P, circle_O)             # 点 P 在 ⊙O 外部
outside(P, rectangle_ABCD)       # 点 P 在矩形 ABCD 外部 (v1.6.3)
outside(P, parallelogram_ABCD)   # 点 P 在平行四边形 ABCD 外部 (v1.6.3)
outside(P, square_ABCD)          # 点 P 在正方形 ABCD 外部 (v1.6.3)
outside(P, quadrilateral_ABCD)   # 点 P 在四边形 ABCD 外部 (v1.6.3)
outside(P, polygon_ABCDEF)       # 点 P 在多边形 ABCDEF 外部 (v1.6.3)
```

**语义**：声明点在某个区域的外部。

**对应中文**：

- 「点 P 在 △ABC 外」→ `outside(P, triangle_ABC)`
- 「圆外一点 P」→ `outside(P, circle_O)`
- 「点 P 在矩形 ABCD 外」→ `outside(P, rectangle_ABCD)`
- 「点 P 在四边形 ABCD 外」→ `outside(P, quadrilateral_ABCD)`

#### 14.3.3 扇形区域

扇形区域约束使用 `sector(P, O, A, B)` 定义，表示点 P 在以 O 为圆心、从 OA 到 OB 的扇形区域内。

```text
sector(P, O, A, B)               # 点 P 在扇形 OAB 内部
```

**语义**：声明点在扇形区域内。角度按逆时针方向从 OA 到 OB 计算。

**对应中文**：

- 「点 P 在扇形 AOB 内」→ `sector(P, O, A, B)`

#### 14.3.4 半平面 `half_plane`

```text
half_plane(P, AB, side=left)     # 点 P 在直线 AB 的左侧半平面
half_plane(P, AB, side=right)    # 点 P 在直线 AB 的右侧半平面
```

**语义**：声明点在直线划分的半平面内。

- `left`：沿 A→B 方向看，点在左侧
- `right`：沿 A→B 方向看，点在右侧

**对应中文**：

- 「P 在直线 AB 的一侧」→ 需要根据上下文判断 `side=left` 或 `side=right`
- 「P 与 C 在直线 AB 的同侧」→ 需要先确定 C 的位置

#### 14.3.5 阴影区域标记 `shaded`

```text
shaded(triangle_ABC)             # 标记 △ABC 为阴影区域
shaded(sector_OAB)               # 标记扇形 OAB 为阴影区域
shaded(circle_O)                 # 标记 ⊙O 为阴影区域
```

**语义**：标记某个区域为阴影/填充区域，通常用于表示题目中需要强调或计算面积的部分。

**对应中文**：

- 「图中阴影部分」→ `shaded(region_ref)`
- 「求阴影部分的面积」→ 通过 `shaded` 标记目标区域

**Solver/Renderer 处理**：

- 渲染时对该区域进行填充显示
- 可用于面积计算的目标区域标识

### 14.4 角平分线消歧 `on(P, bisector, near=...)`

```text
on(P, bisector_ABC, near=B)      # P 在 ∠ABC 的平分线上，靠近顶点 B
on(P, bisector_ABC, near=C)      # P 在 ∠ABC 的平分线上，远离顶点 B，靠近边 BC 方向
```

**语义**：当在角平分线上取点时，消歧点的位置。

**参数说明**：

- `near=B`（顶点）：点靠近角的顶点
- `near=A` 或 `near=C`（边端点）：点远离顶点，靠近该边方向

**对应中文**：

- 「在 ∠ABC 的平分线上取一点 P，P 靠近 B」→ `on(P, bisector_ABC, near=B)`
- 「角平分线与 BC 边交于点 D」→ `on(D, bisector_ABC, near=C)` + `on(D, BC)`

### 14.5 任选点 `choose`

#### 14.5.1 线上任选点

```text
choose(P, on=AB)                 # 在线段 AB 上任选一点 P
choose(P, on=circle_O)           # 在圆 O 上任选一点 P
choose(P, on=ray_AB)             # 在射线 AB 上任选一点 P
```

**语义**：在指定位置随机/默认生成一个点。

**Solver 处理**：

- 在线段上：默认取中点或黄金分割点
- 在圆上：默认取特定角度位置（如 45°）
- 在射线上：默认取距起点一定距离的点

**对应中文**：

- 「在 AB 上任取一点 P」→ `choose(P, on=AB)`
- 「在 ⊙O 上取一点 Q」→ `choose(Q, on=circle_O)`

#### 14.5.2 区域内任选点

```text
choose(P, in=triangle_ABC)       # 在 △ABC 内任选一点 P
choose(P, in=circle_O)           # 在 ⊙O 内任选一点 P
choose(P, in=sector_AOB)         # 在扇形 AOB 内任选一点 P
```

**语义**：在指定区域内随机/默认生成一个点。

**Solver 处理**：

- 在三角形内：默认取重心
- 在圆内：默认取圆心附近
- 在扇形内：默认取中心角对应的位置

**对应中文**：

- 「在 △ABC 内任取一点 P」→ `choose(P, in=triangle_ABC)`

### 14.6 消歧指令的 Extractor 处理

ConstraintExtractor 对消歧指令的处理策略：

| 指令类型                      | 处理方式                                    |
| ----------------------------- | ------------------------------------------- |
| `direction`                 | 添加 soft constraint，影响初始值生成        |
| `on_extended`               | 生成 `on(P, segment_extended)` + 方向约束 |
| `inside/outside`            | 推断初始坐标，不添加硬约束                  |
| `sector`                    | 推断初始坐标，可选添加角度范围约束          |
| `half_plane`                | 推断初始坐标，可选添加位置验证              |
| `bisector_on` with `near` | 生成角平分线约束 + 位置提示                 |
| `choose`                    | 生成默认坐标，标记为可自由移动              |

### 14.7 中文表达对照（消歧相关）

| 中文表达             | DSL 转换                                        |
| -------------------- | ----------------------------------------------- |
| 延长 AB 至点 P       | `on_extended(P, AB, towards=B)`               |
| 延长 BA 至点 P       | `on_extended(P, AB, towards=A)`               |
| P 在 AB 上靠近 A     | `on(P, AB)` + `direction(P, AB, towards=A)` |
| P 在 △ABC 内部      | `inside(P, triangle_ABC)`                     |
| P 在 ⊙O 外部        | `outside(P, circle_O)`                        |
| P 在扇形 AOB 内      | `inside(P, sector_OAB)`                       |
| P 在矩形 ABCD 内     | `inside(P, rectangle_ABCD)`                   |
| P 在矩形 ABCD 外     | `outside(P, rectangle_ABCD)`                  |
| P 在四边形 ABCD 内   | `inside(P, quadrilateral_ABCD)`               |
| P 在四边形 ABCD 外   | `outside(P, quadrilateral_ABCD)`              |
| P 在多边形内         | `inside(P, polygon_ABCDEF)`                   |
| P 在多边形外         | `outside(P, polygon_ABCDEF)`                  |
| P 在直线 AB 左侧     | `half_plane(P, AB, side=left)`                |
| 在 AB 上任取一点 P   | `choose(P, on=AB)`                            |
| 在 △ABC 内取一点 P  | `choose(P, in=triangle_ABC)`                  |
| P 在角平分线上靠近 B | `on(P, bisector_ABC, near=B)`                 |

---

## 15. 中文表达对照表

本节提供系统的中文几何表达与 DSL 的映射关系，用于指导标注和模型训练。

### 15.1 基础图形描述

| 中文表达                | DSL 转换                      |
| ----------------------- | ----------------------------- |
| 在△ABC中 / 如图，△ABC | `triangle(A, B, C)`         |
| 四边形 ABCD             | `quadrilateral(A, B, C, D)` |
| 矩形 ABCD               | `rectangle(A, B, C, D)`     |
| 平行四边形 ABCD         | `parallelogram(A, B, C, D)` |
| 正方形 ABCD             | `square(A, B, C, D)`        |
| ⊙O / 圆 O              | `circle(O)`                 |

### 15.2 点与位置关系

| 中文表达              | DSL 转换                    |
| --------------------- | --------------------------- |
| 点 D 在 BC 上         | `on(D, BC)`               |
| 点 E 在 AB 的延长线上 | `on(E, AB_extended)`      |
| 点 P 在圆 O 上        | `on(P, circle_O)`         |
| M 是 AB 的中点        | `midpoint(M, AB)`         |
| H 是从 A 到 BC 的垂足 | `foot(H, A, BC)`          |
| P 是 AB 与 CD 的交点  | `intersection(P, AB, CD)` |
| A、B、C 三点共线      | `collinear(A, B, C)`      |
| A、B、C、D 四点共圆   | `concyclic(A, B, C, D)`   |

### 15.3 线段与角度关系

| 中文表达                | DSL 转换                  |
| ----------------------- | ------------------------- |
| AB = AC                 | `equal(AB, AC)`         |
| AB ∥ CD / AB 平行于 CD | `parallel(AB, CD)`      |
| AB ⟂ CD / AB 垂直于 CD | `perpendicular(AB, CD)` |
| ∠ABC = 40°            | `angle(ABC)=40`         |
| ∠ABC = ∠DEF           | `equal_angle(ABC, DEF)` |
| ∠ABC 是直角            | `right_angle(ABC)`      |
| AB 的长度为 5           | `length(AB)=5`          |
| AB : CD = 2 : 1         | `ratio(AB, CD)=2`       |

### 15.4 操作型表达

| 中文表达              | DSL 转换                         |
| --------------------- | -------------------------------- |
| 连接 AD               | `connect(A, D)`                |
| 延长 AB 至点 D        | `extend(AB, D)`                |
| 过点 P 作 BC 的垂线   | `draw_perpendicular(P, BC, l)` |
| 过点 P 作 BC 的平行线 | `draw_parallel(P, BC, l)`      |
| 作 ∠ABC 的平分线     | `bisect_angle(A, B, C, l)`     |
| 从 A 向 BC 作高       | `draw_altitude(A, B, C, H)`    |
| 作 BC 边上的中线      | `draw_median(A, B, C, M)`      |

### 15.5 特殊三角形

| 中文表达                        | DSL 转换                                |
| ------------------------------- | --------------------------------------- |
| 等腰三角形 ABC，AB = AC         | `isosceles(A, B, C)`                  |
| 等边三角形 ABC                  | `equilateral(A, B, C)`                |
| 等腰直角三角形，AB=AC，∠A=90° | `isosceles_right(A, B, C)`            |
| 直角三角形，∠B = 90°          | `right_triangle(A, B, C, right_at=B)` |
| △ABC ≌ △DEF                  | `congruent(ABC, DEF)`                 |
| △ABC ∼ △DEF                  | `similar(ABC, DEF)`                   |

### 15.6 可忽略的表达

以下中文表达通常作为题目引导语，不需要转换为 DSL：

| 中文表达        | 处理方式                 |
| --------------- | ------------------------ |
| 如图所示 / 如图 | 忽略（图形描述起始标记） |
| 已知 / 设       | 忽略（条件引导词）       |
| 求证 / 证明     | 忽略（问题类型标记）     |
| 求 / 计算       | 忽略（问题类型标记）     |

---

## 16. 合法性规则（Validity Rules）

所有 DSL 必须满足以下规则：

| 规则编号 | 规则描述                                              | 示例                                                               |
| -------- | ----------------------------------------------------- | ------------------------------------------------------------------ |
| V1       | Token 必须出现在本规范定义的列表中                    | `circle(O)` ✓, `oval(O)` ✗                                   |
| V2       | 点名必须是大写字母或大写字母+数字或带 `_prime` 后缀 | `A`, `B1`, `C_prime` ✓                                      |
| V3       | 所有关键字必须小写                                    | `triangle` ✓, `Triangle` ✗                                   |
| V4       | 几何度量角度（内角）必须在 (0, 180] 范围内            | `angle(ABC)=40` ✓, `angle(ABC)=-10` ✗, `angle(ABC)=200` ✗ |
| V4a      | 变换操作的角度允许负值（单位：度），负值表示顺时针    | `rotate(..., angle=-30)` ✓ 表示顺时针旋转 30°                  |
| V5       | 多边形顶点数必须 ≥ 3                                 | `polygon(A, B, C)` ✓, `polygon(A, B)` ✗                      |
| V6       | 圆的引用必须使用 `circle_O` 格式                    | `on(A, circle_O)` ✓, `on(A, O)` ✗                            |
| V7       | 每个 DSL 函数的参数数量必须匹配定义                   | `triangle(A, B, C)` ✓, `triangle(A, B)` ✗                    |
| V8       | 线段标识必须由两个点名组成                            | `segment(AB)` ✓, `segment(A)` ✗                              |
| V9       | 引用的几何对象必须先声明（推荐）                      | 先 `triangle(A, B, C)` 再 `equal(AB, AC)`                      |
| V10      | 一条 DSL 只能表达一个原子事实                         | 不能合并多个独立约束                                               |
| V11      | `tangent(A, circle_O)` 隐含 `on(A, circle_O)`     | 切点必须在圆上                                                     |

**V11 实现行为**：

- **Validator（strict 模式）**：若存在 `tangent(A, circle_O)` 但缺少 `on(A, circle_O)`，报 error
- **Validator（autocorrect 模式）**：自动补全 `on(A, circle_O)` 并记录 warning
- **Canonicalization**：归一化时自动补全 `on(A, circle_O)`，确保评估一致性

**角度语义说明**：

- `angle(ABC)=N` 表示 ∠ABC 的**内角**度数，N ∈ (0, 180]
- `rotate(..., angle=N)` 的 N 为旋转角度（单位：度），正值逆时针，负值顺时针
- 本 DSL 默认使用**内角**表示，不支持外角或有向角（如需扩展，另行定义）

### 类型检查规则

| 参数位置                       | 期望类型 | 说明                         |
| ------------------------------ | -------- | ---------------------------- |
| `equal(X, Y)` 的 X, Y        | SEGMENT  | 必须是线段标识符             |
| `angle(X)` 的 X              | ANGLE    | 必须是三个点组成的角         |
| `on(P, L)` 的 P              | POINT    | 必须是点                     |
| `on(P, L)` 的 L              | LOCATION | 可以是线段、延长线、射线或圆 |
| `parallel(L1, L2)` 的 L1, L2 | LINE     | 必须是线段或命名直线         |

### 线段与直线的语义区分

| 上下文                    | SEGMENT 的语义                                          | 示例                     |
| ------------------------- | ------------------------------------------------------- | ------------------------ |
| `on(P, AB)`             | **线段** AB（P 在 A、B 之间，含端点）             | 点 P 在线段 AB 上        |
| `on(P, AB_extended)`    | 线段 AB 的**延长线**（已在 EBNF LOCATION 中定义） | 点 P 在 AB 延长线上      |
| `collinear(A, P, B)`    | 三点**共线**（P 在直线 AB 上，不限于线段内）      | 点 P 在直线 AB 上        |
| `parallel(AB, CD)`      | 过 A、B 两点的**直线**                            | 直线 AB 平行于直线 CD    |
| `perpendicular(AB, CD)` | 过 A、B 两点的**直线**                            | 直线 AB 垂直于直线 CD    |
| `bisector(l, ABC)`      | l 是**直线**                                      | 直线 l 是 ∠ABC 的平分线 |
| `tangent(l, circle_O)`  | l 是**直线**                                      | 直线 l 与圆 O 相切       |

**规则总结**：

1. **线段语义**：`on(P, AB)` 表示 P 在线段 AB 上（含端点，即 A、P、B 共线且 P 在 A、B 之间或与端点重合）
2. **直线语义**：在直线关系约束（parallel, perpendicular, bisector, tangent）中，SEGMENT 一律解释为"过两点的直线"
3. **直线上的点**：若需表达"P 在直线 AB 上（不限于线段）"，应使用 `collinear(A, P, B)` 或 `collinear(P, A, B)`

**关于 AB_extended 的说明**：

- `AB_extended` 已在 EBNF 的 LOCATION 终结符中定义：`LOCATION ::= SEGMENT | SEGMENT "_extended" | RAY_REF | ...`
- 因此 `on(P, AB_extended)` 是合法语法，表示"P 在线段 AB 的延长线上"
- 使用 `on_extended(P, AB, towards=B)` 可以更精确地指定延长方向（见第 14.2.2 节）

---

## 17. 标准化与规范化（Canonicalization）（v1.6）

本节定义 DSL 程序的标准化规则，用于训练数据一致性和评估指标准确性。

### 17.1 DSL 语句拓扑排序

DSL 语句必须按照**定义-引用顺序**排列，即对象必须先声明再使用。推荐的 9 层优先级如下：

| 层级 | 语句类型       | 示例                                                                                                                                                             |
| ---- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | 点声明、坐标系 | `point(A)`, `coordinate_system(O)`                                                                                                                           |
| 2    | 图形声明       | `circle(O)`, `triangle(A, B, C)`, `line_function(l, 2, 1)`                                                                                                 |
| 3    | 线段/射线/直线 | `segment(AB)`, `ray(A, B)`, `line(l, A, B)`                                                                                                                |
| 4    | 位置约束       | `on(D, BC)`, `midpoint(M, AB)`, `arc_midpoint(D, BC, circle_O)`, `foot(H, A, BC)`, `intersection(...)`                                                 |
| 5    | 构造操作       | `extend(AB, D)`, `draw_perpendicular(...)`                                                                                                                   |
| 6    | 变换操作       | `rotate(...)`, `reflect(...)`, `fold(...)`                                                                                                                 |
| 7    | 几何关系       | `equal(AB, CD)`, `parallel(...)`, `perpendicular(...)`, `inscribed(...)`, `circumscribed(...)`, `similar(...)`, `congruent(...)`, `tangent(...)` |
| 8    | 度量约束       | `angle(ABC)=60`, `length(AB)=5`, `ratio(...)=2`                                                                                                            |
| 9    | 消歧/辅助      | `inside(...)`, `shaded(...)`, `direction(...)`                                                                                                             |

#### 17.1.1 Layer 2（图形声明）vs Layer 7（几何关系）的区别

这是一个容易混淆的关键点，需要特别注意：

**Layer 2 - 图形声明**：声明几何对象的**存在**

- `circle(O)` - 声明存在一个圆心为O的圆
- `triangle(A, B, C)` - 声明存在一个三角形ABC
- `rectangle(A, B, C, D)` - 声明存在一个矩形ABCD
- `line_function(l, 2, 1)` - 声明存在一个函数线 y=2x+1

**Layer 7 - 几何关系**：描述已声明对象之间的**关系**

- `inscribed(ABC, circle_O)` - 三角形ABC **内接于** 圆O
- `circumscribed(ABC, circle_O)` - 三角形ABC **外接于** 圆O
- `similar(ABC, DEF)` - 三角形ABC **相似于** 三角形DEF
- `congruent(ABC, DEF)` - 三角形ABC **全等于** 三角形DEF
- `tangent(A, circle_O)` - 点A处 **切于** 圆O

**关键原则**：

1. **声明在前，关系在后**：必须先声明对象（Layer 2），才能描述对象间关系（Layer 7）
2. **依赖性检查**：`inscribed(ABC, circle_O)` 依赖于 `triangle(A,B,C)` 和 `circle(O)` 都已声明
3. **语义清晰度**：
   - ✅ 正确：`triangle(A,B,C)` → `circle(O)` → ... → `inscribed(ABC, circle_O)`
   - ❌ 错误：`inscribed(ABC, circle_O)` → `triangle(A,B,C)` (违反依赖顺序)

**常见错误示例**：

```text
# 错误：inscribed 在 triangle 之前
inscribed(ABC, circle_O)  # Layer 7
triangle(A, B, C)         # Layer 2 - 但ABC还未定义！

# 正确：先声明，后关系
circle(O)                 # Layer 2
triangle(A, B, C)         # Layer 2
on(A, circle_O)           # Layer 4
on(B, circle_O)           # Layer 4
on(C, circle_O)           # Layer 4
inscribed(ABC, circle_O)  # Layer 7 - ABC和O都已定义
```

**macros 的特殊性**：

某些 macro 语句虽然看起来像声明，但实际上会展开为声明+关系的组合：

- `isosceles(A, B, C)` → `triangle(A,B,C)` (Layer 2) + `equal(AB, AC)` (Layer 7)
- `circumcircle(O, A, B, C)` → `circle(O)` (Layer 2) + `on(A,circle_O)` (Layer 4) + `on(B,circle_O)` (Layer 4) + `on(C,circle_O)` (Layer 4)

这些 macro 可以放在 Layer 2，因为它们的主要作用是声明图形，附带的约束会自动展开到正确的层级。

### 17.2 Operations 与 Macros 的展开

Operations（如 `connect`, `extend`）和 Macros（如 `isosceles`）是**语法糖**，可以无损展开为基础语句。

| 原始形式                      | 展开形式                                                         |
| ----------------------------- | ---------------------------------------------------------------- |
| `connect(A, D)`             | `segment(AD)`                                                  |
| `isosceles(A, B, C)`        | `triangle(A, B, C)` + `equal(AB, AC)`                        |
| `draw_altitude(A, B, C, H)` | `foot(H, A, BC)` + `segment(AH)` + `perpendicular(AH, BC)` |
| `fold(C, C', EF)`           | `reflect(C', C, EF)`                                           |

**训练建议**：评估时可以先将预测结果和标准答案都展开为 canonical form，再计算匹配度。

### 17.2.1 等价写法归一化

以下等价写法在评估时应归一化为 canonical form：

| 输入形式（兼容）                                  | Canonical Form               | 说明               |
| ------------------------------------------------- | ---------------------------- | ------------------ |
| `angle(ABC)=angle(DEF)`                         | `equal_angle(ABC, DEF)`    | 角相等（无向内角） |
| `length(AB)=length(CD)`                         | `equal(AB, CD)`            | 长度相等           |
| `connect(A, D)`                                 | `segment(AD)`              | 连接两点           |
| `fold(C, C', EF)`                               | `reflect(C', C, EF)`       | 折叠等价于反射     |
| `tangent(A, circle_O)` 缺少 `on(A, circle_O)` | 自动补全 `on(A, circle_O)` | V11 隐含约束       |

**equal_angle 语义定义**：

- `equal_angle(ABC, DEF)` 表示 ∠ABC 与 ∠DEF 的**无向内角**相等
- 与 V4 规则一致，均使用内角语义
- 不涉及有向角或角的方向性

**归一化原则**：模型输出和标准答案在评估前都应先归一化为 canonical form，再计算匹配度。

### 17.3 点名标准化

对于自动生成的辅助点，建议使用以下命名规则：

- 中点：`M`, `M1`, `M2`...
- 垂足：`H`, `H1`, `H2`...
- 交点：`P`, `Q`, `P1`...
- 变换后的点：`A_prime`, `B_prime`...

---

## 18. 标注示例

以下是完整的标注示例，展示从中文描述到 DSL 的转换过程。

### 18.1 示例 1：圆周角定理

**中文描述**：点 A、B、C 是⊙O 上的点，∠AOB=70°, ∠ACB=35°

**DSL 标注**：

```text
circle(O)
on(A, circle_O)
on(B, circle_O)
on(C, circle_O)
segment(OA)
segment(OB)
segment(AB)
segment(CA)
segment(CB)
angle(AOB)=70
angle(ACB)=35
```

**完整 JSON 格式**：

```json
{
  "id": "00001",
  "text_cn": "点 A、B、C是⊙O 上的点，∠AOB=70°, ∠ACB=35°",
  "dsl": [
    "circle(O)",
    "on(A, circle_O)",
    "on(B, circle_O)",
    "on(C, circle_O)",
    "segment(OA)",
    "segment(OB)",
    "segment(AB)",
    "segment(CA)",
    "segment(CB)",
    "angle(AOB)=70",
    "angle(ACB)=35"
  ],
  "meta": {
    "source": "圆周角定理基础题",
    "difficulty": "easy",
    "tags": ["circle", "inscribed_angle", "central_angle"]
  }
}
```

### 18.2 示例 2：矩形折叠问题

**中文描述**：折叠矩形ABCD，点C位于AD的中点C'，AB=9, BC=6

**DSL 标注**：

```text
rectangle(A, B, C, D)
point(E)
point(F)
point(C_prime)
point(B_prime)
midpoint(C_prime, AD)
on(E, BC)
on(F, CD)
segment(EF)
segment(EC_prime)
segment(FC_prime)
segment(EB_prime)
segment(FB_prime)
fold(C, C_prime, EF)
fold(B, B_prime, EF)
length(AB)=9
length(BC)=6
```

### 18.3 示例 3：平行四边形与垂足

**中文描述**：平行四边形ABDC中，P是BC上一点，Q是BC延长线上一点，H是从D到BP的垂足

**DSL 标注**：

```text
parallelogram(A, B, D, C)
point(P)
point(Q)
point(H)
on(P, BC)
on(Q, BC_extended)
segment(AD)
segment(AB)
segment(BD)
segment(DC)
segment(DH)
segment(DP)
segment(DQ)
foot(H, D, BP)
perpendicular(DH, BC)
```

### 18.4 示例 4：等腰三角形与中线

**中文描述**：在△ABC中，AB=AC，M是BC的中点，连接AM

**DSL 标注**：

```text
isosceles(A, B, C)
midpoint(M, BC)
connect(A, M)
```

**或等价的展开形式**：

```text
triangle(A, B, C)
equal(AB, AC)
midpoint(M, BC)
segment(AM)
```

### 18.5 示例 5：作图操作

**中文描述**：在△ABC中，过点A作BC的垂线，垂足为H，作∠BAC的平分线交BC于点D

**DSL 标注**：

```text
triangle(A, B, C)
draw_altitude(A, B, C, H)
bisect_angle(B, A, C, l)
intersection(D, l, BC)
on(D, BC)
```

---

## 19. 版本管理与变更日志

### 19.1 版本号规则

本规范采用语义化版本号（Semantic Versioning）：

- **主版本号（MAJOR）**：不兼容的 API 变更
- **次版本号（MINOR）**：向后兼容的功能性新增
- **修订号（PATCH）**：向后兼容的问题修正

### 19.2 向后兼容性承诺

- **新增指令**：旧数据集仍然有效，新指令为可选
- **废弃指令**：保留解析支持至少一个主版本周期，验证器给出 warning
- **语义重大改动**：提供迁移脚本 `migrate_dsl_vX_to_vY.py`

### 19.3 变更日志

#### v1.8.3 (2026-01-28)

**trapezoid 梯形语法 (新增)**：

- **新增 `trapezoid(A, B, C, D)` 语法**：声明一个梯形，其中 AB // CD 为两条平行的底边
  - 隐含约束：`parallel(AB, CD)`
  - BC 和 AD 为两条腰，不添加平行约束
  - 与平行四边形的区别：只有一对对边平行
  - 支持区域引用：`trapezoid_ABCD`
  - 支持 `inside(P, trapezoid_ABCD)` 和 `outside(P, trapezoid_ABCD)` 约束

#### v1.8.2 (2026-01-28)

**perpendicular_bisector 线段语法 (新增)**：

- **新增线段-线段语法**：`perpendicular_bisector(AB, CD)` 表示线段 AB 是 CD 的垂直平分线
  - 约束 1：CD 的中点在直线 AB 上
  - 约束 2：AB ⟂ CD
  - 与现有 `perpendicular_bisector(P, AB)`（点在中垂线上）语法兼容共存
  - 通过第一个参数长度区分：单字符为点版本，多字符为线段版本

#### v1.8.1 (2026-01-28)

**intersection 延长线语法 (新增)**：

- **新增 `_extended` 后缀支持**：`intersection` 语句支持 `AB_extended` 语法表示线段延长线
  - `intersection(P, AB, CD_extended)` - P 是 AB 与 CD 延长线的交点
  - `intersection(P, AB_extended, CD)` - P 是 AB 延长线与 CD 的交点
  - `intersection(P, AB_extended, CD_extended)` - 两条延长线的交点
  - 新增 EBNF 终结符 `LINE_REF ::= LINE | LINE "_extended"`
  - 新增 `IntersectionConstraint.obj1_extended` 和 `obj2_extended` 字段
  - 向后兼容：现有 `intersection(P, AB, CD)` 语法不变

#### v1.8.0 (2026-01-28)

**扇形声明语法 (新增)**：

- **新增 `sector(O, A, B, ...)` 声明语法**：声明扇形区域，用于渲染和约束
  - `sector(O, A, B)` - 基本形式，隐含 `equal(OA, OB)`
  - `sector(O, A, B, angle=120)` - 指定圆心角，隐含角度约束
  - `sector(O, A, B, r=5)` - 指定半径，隐含长度约束
  - 新增 EBNF 产生式 `sector_decl`
  - 新增 `SectorDecl` AST 节点
  - 新增 `visit_sector_decl` 约束提取方法

**与消歧语法的区别**：

| 语法 | 参数 | 用途 |
|-----|------|------|
| `sector(O, A, B, ...)` | 3点+可选 | 扇形声明 |
| `sector(P, O, A, B)` | 4点 | 消歧（P在扇形内） |

**实现状态**：

- Lexer: 复用现有 `SECTOR` token
- Parser: 根据参数数量区分声明和消歧
- AST: 新增 `SectorDecl` 节点
- Extractor: 新增 `visit_sector_decl` 方法
- GeometryModel: 新增 `SectorInfo` 数据类
- 渲染: SVG/TikZ 支持扇形渲染

#### v1.7.1 (2026-01-25)

**foot 约束语法增强**：

- **支持延长线语法**：`foot(H, A, BC_extended)`
  - 新增 EBNF 终结符 `FOOT_LINE ::= SEGMENT | SEGMENT "_extended"`
  - 新增 `FootConstraint.is_extended` 字段（用于 AST 往返）
  - 与 `on()` 约束保持一致的行为
  - 向后兼容：`foot(H, A, BC)` 仍然有效

**语义澄清**：

- `foot(H, A, BC)`: BC 被解释为"过 B、C 两点的直线"（与规范第 16 节一致）
- `foot(H, A, BC_extended)`: 语义等价，但明确标注为延长线（提高可读性）
- 约束求解行为完全相同（line 始终视为无限直线）

**实现状态**：

- Parser: 增强 `parse_foot_constraint()` 支持 `_extended` 后缀
- AST: `FootConstraint` 新增 `is_extended: bool` 字段
- Extractor: 文档化 `_extended` 处理逻辑
- 测试用例: 完整覆盖语法解析和约束求解

#### v1.7.0 (2026-01-24)

**核心功能增强**：

- **函数点坐标约束**：新增 `function_point_constrained` 约束类型
  - 语法：`function_point_constrained(P, f, x=val)` 或 `function_point_constrained(P, f, y=val)`
  - 语义：点 P 在函数 f 上，且某一坐标（x 或 y）值已知
  - 新增 `FUNCTION_POINT_CONSTRAINED` token 类型
  - 新增 `FunctionPointConstrainedConstraint` AST 节点和约束类
  - Parser 支持命名参数语法：`x=val` 或 `y=val`
  - Extractor 智能初始化：自动求解 f(x) = y0 或直接计算 y = f(x0)

**解决的问题**：

- 之前无法表达"函数上某坐标已知的点"：
  - `point_coord(P, m, -0.5)` ← 错误：m 是符号变量，无法解析
  - `function_point(P, p)` ← 不完整：需要额外约束确定 P 的位置
- 新语法直接表达：`function_point_constrained(P, p, y=-0.5)`

**适用场景**：

- 「点 P 在抛物线上，纵坐标为 -0.5」
- 「第三象限内函数图像上的点，y 坐标为 -3」
- 「双曲线上横坐标为 2 的点」

**向后兼容性**：

- 保留 `function_point(P, f)` 语法，无需迁移现有数据集
- 新约束作为补充，用于更精确的坐标控制

**实现状态**：

- 新增 `solver/parser/lexer.py` FUNCTION_POINT_CONSTRAINED token
- 新增 `solver/parser/ast_nodes.py` FunctionPointConstrainedConstraint 节点
- 新增 `solver/parser/parser.py` parse_function_point_constrained_constraint 方法
- 新增 `solver/constraints/relations.py` FunctionPointConstrainedConstraint 约束类
- 新增 `solver/constraints/extractor.py` visit_function_point_constrained_constraint 方法
- 新增 `solver/constraints/extractor.py` _solve_function_for_x 辅助方法
- 文档完整（第 13.8.1 节）
- 测试用例覆盖（test_function_point_constrained.py）

#### v1.6.9 (2026-01-24)

**核心功能增强**：

- **数学表达式支持**：在所有数值参数位置支持数学表达式
  - 新增 `TokenType.EXPR` token 类型
  - 新增 `solver/parser/expr_eval.py` 表达式求值模块（基于 SymPy）
  - Lexer 增强：`read_expression()` 方法识别分数、根号、混合表达式
  - Parser 增强：`parse_number()` 支持 EXPR token 并自动求值
  - 支持分数：`1/2`, `-7/3`, `5/4`
  - 支持根号：`√3`, `sqrt(5)`, `2√5`（Unicode 和 ASCII 两种写法）
  - 支持混合表达式：`(1+√3)/2`, `2√5/3`
  - 支持基础运算：`+`, `-`, `*`, `/`, 括号优先级

**适用范围**：

- 长度约束：`length(AB)=√3`
- 角度约束：`angle(ABC)=180/3`
- 坐标值：`point_coord(A, 1/2, √3)`
- 比例系数：`ratio(AB, CD)=√2`
- 圆半径：`circle(O, radius=2√5)` （数值半径）
- 旋转角度：`rotate(..., angle=180/3)`

**求值策略**：

- 表达式在 **Parser 阶段立即求值为 float**
- 不保留符号形式（`√3` → `1.7320508...`）
- 使用 SymPy 求值，精度约 15 位小数
- 向后兼容：纯数值 DSL（`3.14`）行为不变

**实现状态**：

- 新增 `solver/parser/expr_eval.py`（172 行）
- 修改 `solver/parser/lexer.py`（新增 EXPR token，read_expression 方法）
- 修改 `solver/parser/parser.py`（增强 parse_number 方法）
- 文档完整（第 10.2.1 节）
- 测试用例覆盖（test_expr_eval.py）

**注意**：v1.6.7 的角度比例约束说明已更新，明确"分数需预先计算为小数"的限制已被解除。

#### v1.6.8 (2026-01-24)

**文档补充**：

- **半圆语法文档补充**：在第 9.5 节添加 `semicircle(O, A, B)` 语法说明
  - `semicircle` 语法已在 solver 代码中完整实现，但 DSL 规范文档缺失
  - 语义：声明以 O 为圆心、AB 为直径的半圆
  - 隐含约束展开：O 是 AB 中点 + 创建圆 O（半径=|OA|）+ A、B 在圆上
  - 补充使用场景、示例代码和注意事项

**实现状态**：

- Lexer/Parser/AST/Extractor 均已实现（v1.5.x 之前）
- 测试用例完整覆盖（test_parser_complete.py, test_extractor_complete.py）
- 本次仅补充文档说明

#### v1.6.7 (2026-01-23)

**新增约束实现**：

- **角度比例约束**：支持 `angle(AOB)=k*angle(BOC)` 语法
  - 新增 `STAR` token 类型用于乘法运算符
  - 新增 `AngleRatioConstraint` AST 节点和约束类
  - Parser 扩展：`parse_angle_constraint()` 支持解析 `NUMBER*angle(...)` 表达式
  - Constraint 实现：`compute_error()` 基于角度值比例计算误差
  - 适用场景：表达角度间的倍数关系（如 ∠AOB = (1/3)∠BOC）

**使用说明**：

- 比例系数 k 必须是数值字面量（不支持符号变量）
- 分数需预先计算为小数：`(1/3)` → `0.3333`
- 示例：`angle(AOB)=2*angle(BOC)`, `angle(CAD)=0.5*angle(BAC)`

#### v1.6.6 (2026-01-23)

**新增约束实现**：

- **弧中点约束**：`arc_midpoint(D, BC, circle_O, arc_type='minor'|'major')`
  - 支持劣弧/优弧中点的精确表达
  - 约束条件：|OD| = |OB| 且 ∠BOD = ∠COD（圆心角相等）
  - 新增 `ARC_MIDPOINT` token、`ArcMidpointConstraint` AST节点和约束类
  - 智能初始化：`_infer_from_arc_midpoint()` 根据弦中点和垂直方向自动推断初始位置
  - 默认 `arc_type='minor'`（劣弧），可选 `'major'`（优弧）

**拓扑排序规范增强**：

- **Layer 2 vs Layer 7 区分**：在 17.1.1 节添加详细说明
  - Layer 2（图形声明）：声明几何对象的**存在性**（如 `circle(O)`, `triangle(A,B,C)`）
  - Layer 7（几何关系）：描述已存在对象间的**关系**（如 `inscribed(...)`, `tangent(...)`）
  - 关键原则：先声明对象，再描述关系

**语义展开简化**：

- **bisect_angle** 展开简化为仅 `bisector(l, ABC)`，去掉匿名点 P
- **bisector** 语义明确蕴含 `on(B, l)`（角平分线必过顶点）

**类型语义澄清**：

- 新增"线段与直线的语义区分"表，明确 `on(P, AB)` 为线段语义
- 明确"直线上的点"应使用 `collinear(A, P, B)`
- 明确 `parallel/perpendicular` 中 SEGMENT 解释为"过两点的直线"

**Canonicalization 增强**：

- 新增 17.2.1 节"等价写法归一化"规则表
- 明确 `equal_angle` 为"无向内角相等"语义
- V11 隐含约束纳入归一化流程

#### v1.6.6 (2026-01-20)

**坐标轴约束语法文档化**：

- **文档化 `on(P, axis_x)` 和 `on(P, axis_y)`**：将已实现的坐标轴约束语法添加到 DSL 规范

**语法**：

```text
on(P, axis_x)               # 点 P 在 x 轴上（P.y = 0）
on(P, axis_y)               # 点 P 在 y 轴上（P.x = 0）
```

**约束说明**：

- `on(P, axis_x)` 约束点 P 的 y 坐标为 0
- `on(P, axis_y)` 约束点 P 的 x 坐标为 0
- 通常与 `coordinate_system(O)` 配合使用
- 也可独立使用，以默认原点 (0, 0) 为参考

**Solver 实现**：

- `PointOnAxisConstraint` - 坐标轴上点约束（已在 v1.5.0 实现）
- Parser 识别 `axis_x` / `axis_y` 关键字
- Extractor 生成 `PointOnAxisConstraint`

**对应中文**：

- 「点 A 在 x 轴上」→ `on(A, axis_x)`
- 「点 B 在 y 轴上」→ `on(B, axis_y)`

#### v1.6.5 (2026-01-22)

**弧上点约束 (新增)**：

- **新增 `on(E, arc_BC)`**：约束点 E 在弧 BC 上

**语法**：

```text
arc(B, C, circle_O)     # 声明弧 BC（圆 O 上从 B 到 C 的弧）
on(E, arc_BC)           # 点 E 在弧 BC 上
```

**约束说明**：

- `arc_BC` 引用格式表示从 B 到 C 的弧（逆时针方向的劣弧）
- 使用前必须先声明 `arc(B, C, circle_O)` 以关联圆心
- 约束等价于：点在圆上 + 角度在弧范围内

**Solver 实现**：

- 新增 `PointOnArcConstraint` - 弧上点约束
- Parser 识别 `arc_` 前缀的 location_type
- Extractor 跟踪弧声明并生成相应约束
- NumericSolver 支持弧上点的智能初始化

#### v1.6.4 (2026-01-21)

**角度三角函数值约束 (新增)**：

- **新增 `sin(ABC)=value`**：约束角度 ABC 的正弦值
- **新增 `cos(ABC)=value`**：约束角度 ABC 的余弦值
- **新增 `tan(ABC)=value`**：约束角度 ABC 的正切值

**语法**：

```text
sin(ABC)=0.5          # ∠ABC 的正弦值为 0.5
cos(ABC)=0.866        # ∠ABC 的余弦值为 √3/2
tan(ABC)=1            # ∠ABC 的正切值为 1
```

**Solver 实现**：

- 新增 `AngleSinConstraint` - 正弦值约束
- 新增 `AngleCosConstraint` - 余弦值约束
- 新增 `AngleTanConstraint` - 正切值约束

#### v1.6.3 (2026-01-20)

**REGION_REF 扩展**：

- **新增 RECTANGLE_REF**：支持 `rectangle_ABCD` 格式的矩形区域引用
- **新增 PARALLELOGRAM_REF**：支持 `parallelogram_ABCD` 格式的平行四边形区域引用
- **新增 SQUARE_REF**：支持 `square_ABCD` 格式的正方形区域引用
- **新增 QUADRILATERAL_REF**：支持 `quadrilateral_ABCD` 格式的四边形区域引用
- **新增 POLYGON_REF**：支持 `polygon_ABC...` 格式的任意多边形区域引用

**inside/outside 约束增强**：

- `inside(P, rectangle_ABCD)` - 点 P 在矩形 ABCD 内部
- `outside(P, rectangle_ABCD)` - 点 P 在矩形 ABCD 外部
- 支持所有四边形类型（rectangle、parallelogram、square、quadrilateral）
- 支持任意多边形（polygon）

**Solver 实现**：

- 新增 `InsideQuadrilateralConstraint` - 四边形内部软约束
- 新增 `OutsideQuadrilateralConstraint` - 四边形外部软约束
- 新增 `InsidePolygonConstraint` - 多边形内部软约束
- 新增 `OutsidePolygonConstraint` - 多边形外部软约束
- 新增 `OutsideTriangleConstraint` - 三角形外部软约束（补充缺失）

#### v1.6.2 (2026-01-16)

**Validity Rules 完善**：

- **V4 细化**：几何度量角度（内角）必须在 (0, 180] 范围内，明确"内角"语义
- **V4a 新增**：变换操作角度允许负值（单位：度），负值表示顺时针
- **V11 新增**：`tangent(A, circle_O)` 隐含 `on(A, circle_O)`，validator 支持 strict/autocorrect 双模式

#### v1.6.1 (2026-01-16)

**EBNF 完整性修复**：

- **补充 inscribed/circumscribed/circumcircle**：在 `relation_constraint` 中添加内接、外接圆关系的产生式
- **补充 arc_decl**：在 `decl_stmt` 中添加弧声明 `arc(POINT, POINT, CIRCLE_REF)`
- **修正 POINT 终结符**：将 `[A-Z] [A-Z0-9]*` 改为 `[A-Z] [0-9]*`，避免与 SEGMENT 产生歧义
- **新增 RAY_REF 终结符**：定义射线引用格式 `ray_POINT POINT`
- **新增 BISECTOR_REF 终结符**：定义角平分线引用格式 `bisector_ANGLE`
- **新增 SECTOR_REF 终结符**：定义扇形引用格式 `sector_POINT POINT POINT`
- **更新 REGION_REF**：使用新定义的 SECTOR_REF
- **更新 LOCATION**：使用新定义的 RAY_REF

**语法一致性修复**：

- **统一 sector 语法**：将 14.3.3 节的 `sector(P, O, A, B)` 改为 `inside(P, sector_OAB)`，与 EBNF 3 参数定义一致
- **更新对照表**：修正 14.7 节中扇形相关的 DSL 转换

**语义清晰度改进**：

- **明确 fold 与 reflect 关系**：在 12.2 节添加两者参数顺序差异和等价关系说明
- **添加操作/宏命名规范**：在第 8 章添加 8.0 节，说明 `draw_*` 操作指令与宏指令的参数顺序设计原则
- **重写 tangent 语义**：在 5.7 节明确 `tangent(POINT, CIRCLE)` 表示"该点是切点"
- **添加 tangent_point/tangent_line 参数说明**：在 13.6 节说明参数顺序设计原则

**文档细节修正**：

- **明确 axis_x/axis_y 状态**：在 13.1 节标注为辅助指令，说明通常无需显式使用
- **完善 ANGLE 注释**：在 EBNF 中添加 ANGLE 在连写和逗号分隔两种上下文的说明
- **修正 isosceles_right 描述**：将 15.5 节"直角在 A"改为"AB=AC，∠A=90°"

#### v1.6.0 (2025-12-28)

**EBNF 语法一致性修复**：

- **NUMBER 支持负数**：`NUMBER` 更新为 `SIGNED_NUMBER ::= "-"? [0-9]+ ("." [0-9]+)?`
- **新增 VALUE 类型**：`VALUE ::= SIGNED_NUMBER | SYMBOL`，统一函数参数的数值/符号表示
- **统一 ray 格式**：移除 `ray(AB)` 格式，仅保留 `ray(A, B)` 作为唯一有效格式
- **扩展 intersection**：新增 `intersection(P, LINE, CIRCLE_REF)` 和 `intersection(P, CIRCLE_REF, CIRCLE_REF)` 支持
- **扩展 tangent**：新增 `tangent(LINE, CIRCLE_REF)` 表示直线与圆相切
- **移除 circle_coord**：删除未实现的 `circle_coord` 描述，改用 `circle(O, r=N)` + `point_coord(O, x, y)` 组合

**新增章节**：

- **第 10.4 节 - 角度单位约定**：明确几何约束使用度（degree），三角函数相位使用弧度（radian）
- **第 17 章 - 标准化与规范化（Canonicalization）**：定义 DSL 语句拓扑排序规则、宏展开规则、点名标准化

**章节编号调整**：

- 原第 17 章（标注示例）→ 第 18 章
- 原第 18 章（版本管理）→ 第 19 章

#### v1.5.3 (2025-12-07)

**语法一致性强化**：

- **移除** 旧版 `rotate(P, center=O, angle=N)` 语法（无 `source=` 参数）
- **唯一有效格式**：`rotate(D, source=C, center=O, angle=N)`
- `fold(C, C', EF)` 现在等价于 `reflect(C', C, EF)`，不再产生无效的 SegmentDecl

**新增约束实现**：

- `ConcyclicConstraint` - 四点共圆约束（4x4 行列式条件）
- `EqualAngleConstraint` - 等角约束（余弦相等条件）
- `SegmentLengthRatioConstraint` - 线段长度比约束（`ratio(AB,CD)=2`）

**修复缺失的访问方法**：

- 新增 `visit_ray_decl` - 射线声明处理
- 新增 `parse_ray_decl` - 射线解析方法

**保留关键字完善**：

- 新增 ~20 个缺失的保留关键字到 `RESERVED_KEYWORDS`
- 包括：`ray`, `polygon`, `rectangle`, `parallelogram`, `square`, `concyclic`, `fold`, `isosceles` 等

#### v1.5.0 (2025-12-06)

**新增符号参数支持**：

- 函数声明支持符号变量作为参数：`parabola(p, a, b, c)`、`hyperbola(h, k)`、`line_function(l, k1, b)`
- 新增 AST 节点类型：`ValueNode`（基类）、`NumberValue`（数值）、`SymbolicValue`（符号变量）
- Parser 新增 `parse_value()` 方法，支持解析数字或符号变量
- Extractor 新增 `_resolve_value()` 方法，自动解析符号值为默认数值（用于可视化）
- 支持符号上下文设置：`extractor.symbol_context["k"] = 5.0`

**新增几何约束**：

- `trisection(P, AB, near=A)` - 三等分点约束
- `perpendicular_bisector(M, AB)` - 垂直平分线约束
- `median(M, A, BC)` - 中线约束
- `similar(ABC, DEF)` - 相似三角形约束

**新增函数几何约束**：

- `coordinate_system(O)` - 坐标系声明，自动设置原点 O=(0,0)
- `function_point(P, f)` - 点在函数图像上
- `axis_intersection(A, f, x/y)` - 函数与坐标轴交点
- `function_intersection(P, f, g)` - 两函数交点
- `tangent_point(C, l, f)` - 切点约束
- `tangent(A, circle_O)` - 过点的圆切线

**C++ 后端扩展**：

- 新增 `TANGENT_POINT_CIRCLE` 约束实现
- 新增 `TRISECTION` 约束实现
- 新增 `PERPENDICULAR_BISECTOR` 约束实现
- 新增 `SIMILAR_TRIANGLES` 约束实现

**AST Round-trip 支持**：

- 所有 AST 节点实现 `to_dsl()` 方法，支持 AST → DSL 序列化
- `NumberValue.to_dsl()` 对整数输出整数格式（如 `2` 而非 `2.0`）
- `SymbolicValue.to_dsl()` 输出符号名称

**数据集修复**：

- 修复 8 个语法错误的标注文件
- 修复 4 个使用 `axis_x`/`axis_y` 的文件（改用共线约束）
- 解析成功率：70.1% → 100%
- 约束提取成功率：70.1% → 100%

**测试覆盖**：

- 新增 11 个符号参数测试用例
- 总测试数：151 个（全部通过）

#### v1.4.1 (2025-12-06)

**新增阴影区域标记**：

- 新增 `shaded(REGION_REF)` 语法，用于标记阴影/填充区域
- 支持三角形、扇形、圆等区域的阴影标记
- 在 region_stmt 产生式中添加相应定义

**扩展旋转变换语法**：

- 扩展 `rotate` 语法，新增 `source` 参数
- 支持 `rotate(D, source=C, center=O, angle=60)` 格式
- 明确指定旋转源点和结果点的对应关系

#### v1.4.0 (2025-12-03)

**新增模糊语义消歧指令**：

- 线段方向消歧：`direction(P, AB, towards=A/B)`
- 延长线方向消歧：`on_extended(P, AB, towards=A/B)`
- 区域内部/外部：`inside(P, region)`, `outside(P, region)`
- 扇形区域：`sector(P, O, A, B)`
- 半平面：`half_plane(P, AB, side=left/right)`
- 角平分线消歧：`on(P, bisector_ABC, near=B)`
- 任选点：`choose(P, on=AB)`, `choose(P, in=triangle_ABC)`

**新增终结符**：

- `TRIANGLE_REF`：三角形区域引用
- `REGION_REF`：区域引用（三角形/圆/扇形）
- `SIDE`：半平面侧向（left/right）

**语法更新**：

- EBNF 语法定义新增 `disambig_stmt` 产生式
- 新增第 14 章：模糊语义消歧指令

**设计目的**：

- 解决 LLM 从中文到 DSL 解析时的歧义问题
- 为 Solver 提供明确的初始点位置提示
- 支持区域约束的 soft constraint 处理

#### v1.3.0 (2025-12-02)

**新增函数类型**：

- 新增幂函数：`power_function(f, n)` → y = x^n
- 新增指数函数：`exp_function(f, a)` → y = a^x
- 新增对数函数：`log_function(f, a)` → y = log_a(x)
- 新增三角函数：`sin_function`, `cos_function`, `tan_function`, `cot_function`, `sec_function`, `csc_function`
- 新增反三角函数：`arcsin_function`, `arccos_function`, `arctan_function`, `arccot_function`
- 新增通用函数表达式：`function_expr(f, "expression")` 支持复合函数

**新增函数变换**：

- 平移变换：`translate_function(f, g, dx, dy)`
- 伸缩变换：`scale_function(f, g, sx, sy)`
- 对称变换：`reflect_function(f, g, axis)`

**新增函数与几何交互**：

- 切线：`tangent_line(l, f, P)`
- 渐近线：`asymptote(l, f)`

**语法更新**：

- EBNF 语法定义新增函数相关产生式
- 新增终结符：`STRING`, `AXIS`, `FUNC_REF`

#### v1.2.0 (2025-11-13)

**新增功能**：

- 新增坐标系与函数图像章节（第 13 节）
- 新增坐标系声明：`coordinate_system`, `axis_x`, `axis_y`
- 新增坐标点声明：`point_coord`
- 新增函数图像声明：`line_function`, `hyperbola`, `parabola`
- 新增函数与几何交互：`function_point`, `function_intersection`, `axis_intersection`, `tangent_point`
- 新增坐标系中的圆：`circle_coord`

**结构调整**：

- 章节编号顺延，原 13-16 节变为 14-17 节

#### v1.1.0 (2025-10-25)

**新增功能**：

- 新增操作型构造指令章节（`connect`, `extend`, `draw_perpendicular`, `draw_parallel`, `bisect_angle`, `draw_altitude`, `draw_median`）
- 新增宏指令展开规则（`median`, `altitude`, `angle_bisector`, `perpendicular_bisector`）
- 新增几何关系：`collinear`, `concyclic`, `congruent`, `similar`, `bisector`
- 新增中文表达对照表章节
- 新增 EBNF 形式化语法定义
- 新增完整标注示例
- 新增版本管理机制

**增强功能**：

- 支持多点声明语法 `points(A, B, C, D)`
- 支持直线命名 `line(l, A, B)`
- 支持圆半径声明 `circle(O, r=5)` 和 `circle(O, radius=AB)`
- 增强特殊三角形宏的展开规则文档

**结构调整**：

- 重组章节顺序，使结构更清晰
- 将图形结构与宏指令分离为独立章节

#### v1.0.0 (2025-10-20)

- 初始版本发布
- 定义基础元素、几何关系、图形结构
- 定义圆与弧、角度与长度表达
- 定义点的位置关系、几何变换与折叠操作
- 定义合法性规则

---

> **注意**：本规范为 NeoGeo 项目的正式 DSL 定义文档，所有标注员和模型训练过程必须严格遵循本规范。如有疑问或发现规范缺陷，请联系项目维护者陈铄涵chenshuohan60@gmail.com。
