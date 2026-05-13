# NeoGeo Phase IV 训练指南：Self-Evolving DSL 微调

> 日期：2026-03-05
> 前置依赖：Phase III 训练完成（LoRA checkpoint 在 `outputs/lora_phase3/final`）

---

## 一、为什么需要 Phase IV

Phase III 训练的模型只会输出纯原子 DSL。自演化系统要求模型能输出 CALL + DSL 混合格式——当 prompt 中注入了 Skill 定义时，模型应该优先使用 CALL 调用 Skill，覆盖不到的部分用原子 DSL 补全。

Phase IV 在干净的 gold annotation 基础上，加入混合格式样本和语义增强数据，完全替代 Phase III 的脏数据。

---

## 二、训练数据说明

### 2.1 数据文件

| 文件 | 位置 | 样本数 | 用途 |
|------|------|--------|------|
| `train_split.json` | `dataset/` | 4354 | 训练集 |
| `val_split.json` | `dataset/` | 472 | 验证/测试集 |

**不再使用的文件**（历史产物，不要用于训练）：
- ~~`train_phase3.json`~~（Phase III 脏数据，已废弃）
- ~~`train_final.json`~~（未划分的合并数据，已被 train_split/val_split 替代）
- ~~`train_phase4_mixed.json`~~（中间产物，已合并进 train_split）
- ~~`train_augmented.json`~~（中间产物，已合并进 train_split）

### 2.2 数据组成

训练集（4354 条）和验证集（472 条）均来自 1277 条 gold annotation 的分层抽样（10% 验证集），包含三种类型：

| 类型 | 训练集 | 验证集 | assistant 输出格式 | system prompt |
|------|--------|--------|-------------------|---------------|
| 混合格式（CALL + DSL） | 100 | 14 | `CALL skill(...)\n原子DSL` | 含 Skill 定义段 |
| 纯 DSL（含 Skill 定义） | ~1063 | ~115 | 纯原子 DSL | 含 Skill 定义段 |
| 语义增强（改写） | 3191 | 343 | 纯原子 DSL | 短版 system prompt |

### 2.3 防数据泄漏

训练集和验证集按 annotation_id 整体划分：同一条标注的所有变体（原始、CALL 混合、style_A/B/C 改写）全部在同一个 split 中，不存在泄漏。每条样本都有 `annotation_id` 字段标识来源。

### 2.4 验证集分层覆盖

| 维度 | 分布 |
|------|------|
| 难度 | easy=13, medium=64, hard=51 |
| 对象类型 | triangle=37, coordinate=34, circle=27, quadrilateral=13, transform=11, other=6 |
| CALL 样本 | 14 条（可验证模型是否学会 CALL 格式） |

### 2.5 数据格式

每条样本是 Qwen2.5 chat 格式（system + user + assistant 三段式）：

```json
{
  "messages": [
    {"role": "system", "content": "你是一个专业的几何DSL生成器...\n\n你还可以使用以下已积累的技能（Skill）..."},
    {"role": "user", "content": "将以下中文几何描述转换为DSL程序：\n\n在三角形ABC中..."},
    {"role": "assistant", "content": "CALL circle_points(O=O, points=A,B,C)\ntriangle(A, B, C)\nangle(ACB)=60"}
  ],
  "annotation_id": "00042",
  "augmentation": {"type": "mixed_skill_format", "coverage": 0.24, "skills_used": ["circle_points"]}
}
```

### 2.6 三种场景的学习目标

模型需要学会根据 system prompt 的内容条件性地选择输出格式：

1. **system 有 Skill 定义 + 题目匹配 Skill** -> 输出 CALL + 剩余原子 DSL（100 条训练样本教的）
2. **system 有 Skill 定义 + 题目不匹配** -> 输出纯原子 DSL（~1063 条训练样本教的）
3. **system 无 Skill 定义** -> 输出纯原子 DSL（3191 条语义增强样本教的）

### 2.7 CALL 格式规范

训练数据、system prompt 示例、推理时解析，三者使用统一格式：

```
CALL skill_name(param1=value1, param2=value2)
```

---

## 三、服务器操作步骤

### 3.1 上传数据

```bash
# 从本地上传训练集和验证集
scp dataset/train_split.json cshcsh@172.31.179.101:/home/cshcsh/magicgeo/data/processed/
scp dataset/val_split.json cshcsh@172.31.179.101:/home/cshcsh/magicgeo/data/processed/
```

不需要合并操作。`train_split.json` 和 `val_split.json` 是最终文件，直接使用。

### 3.2 创建 Phase IV 训练配置

```bash
ssh cshcsh@172.31.179.101
cd /home/cshcsh/magicgeo

cp server_scripts/training_config_phase3.yaml server_scripts/training_config_phase4.yaml
```

修改 `training_config_phase4.yaml`：

```yaml
model:
  base_model: "Qwen/Qwen2.5-14B-Instruct"
  lora_checkpoint: "outputs/lora_phase3/final"
  load_lora_weights: true
  fresh_start: true

data:
  train_file: "data/processed/train_split.json"
  val_file: "data/processed/val_split.json"
  max_seq_length: 1536

training:
  num_epochs: 3
  learning_rate: 2.0e-5
  warmup_ratio: 0.15

output:
  output_dir: "outputs/lora_phase4"
  logging_dir: "outputs/lora_phase4/logs"

wandb:
  project: "magicgeo-phase4"
  name: "mixed-skill-format-v1"
  tags: ["phase4", "skill-call-format", "from-phase3"]
```

### 3.3 关键参数说明

| 参数 | Phase III | Phase IV | 原因 |
|------|-----------|----------|------|
| `train_file` | train_phase3.json | **train_split.json** | 干净数据，不再用 Phase III 脏数据 |
| `val_file` | val.json | **val_split.json** | 分层抽样的验证集，含 CALL 样本 |
| `max_seq_length` | 1024 | **1536** | system prompt 加了 Skill 定义段（约 500 token） |
| `num_epochs` | 5 | **3** | 数据量更大（4354 vs ~2000），3 轮足够 |
| `learning_rate` | 5e-5 | **2e-5** | 在 Phase III 基础上微调，避免遗忘 |

### 3.4 开始训练

```bash
cd /home/cshcsh/magicgeo

PYTHONPATH=/home/cshcsh/magicgeo:$PYTHONPATH \
CUDA_VISIBLE_DEVICES=0,1 \
nohup python server_scripts/train_lora_phase3.py \
    --config server_scripts/training_config_phase4.yaml \
    --output_dir outputs/lora_phase4 \
    > logs/train_phase4.log 2>&1 &

tail -f logs/train_phase4.log
```

### 3.5 训练后验证

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import PeftModel

tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-14B-Instruct")
base = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-14B-Instruct", device_map="auto")
model = PeftModel.from_pretrained(base, "outputs/lora_phase4/final")

# 测试：system prompt 含 Skill 定义时，模型是否输出 CALL
messages = [
    {"role": "system", "content": """你是一个专业的几何DSL生成器...

你还可以使用以下已积累的技能（Skill），直接调用而非逐行写 DSL：

SKILL circle_points(O, points):
  声明圆O及其上的多个点
  调用示例: CALL circle_points(O=O, points=A,B,C)

使用规则：
1. 如果题目的构造完全匹配某个 Skill，优先使用 CALL
2. Skill 覆盖不到的部分，继续用原子 DSL 补全
3. CALL 和原子 DSL 可以混写
4. 如果没有合适的 Skill，直接输出完整的原子 DSL"""},
    {"role": "user", "content": "将以下中文几何描述转换为DSL程序：\n\n点A、B、C是圆O上的点，角AOB=70度"}
]

text = tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
inputs = tokenizer(text, return_tensors="pt").to(model.device)
outputs = model.generate(**inputs, max_new_tokens=256, temperature=0.1)
response = tokenizer.decode(outputs[0][inputs["input_ids"].shape[1]:], skip_special_tokens=True)
print(response)

# 期望输出：
# CALL circle_points(O=O, points=A,B,C)
# segment(OA)
# segment(OB)
# angle(AOB)=70
```

如果模型仍然输出纯原子 DSL（没有 CALL），检查：
- `max_seq_length` 是否足够（Skill 定义段是否被截断）
- 混合样本比例是否太低（可增加到 15-20% 重新生成）

---

## 四、自演化系统核心架构

### 4.1 整体闭环

```
题库 -> Curriculum 选题 -> LLM 生成 (CALL + DSL) -> Feedback -> 成功?
                                                              |-- 是 -> LLM 提炼 Skill -> Gatekeeper -> 入库
                                                              +-- 否 -> 记录失败经验 (仅 Hard Gate 触发 Backward Construction)
```

### 4.2 关键设计决策

| 决策 | 选择 | 原因 |
|------|------|------|
| Skill 定义注入位置 | system prompt | 与推理时一致，user prompt 不变 |
| 演化方式 | 上下文记忆，不重训 | 逐题实时，Voyager 范式 |
| Skill 产生者 | LLM 自主提炼 | 非人工设计 |
| Skill 审核 | Gatekeeper 程序化 | 人工干预极少 |
| Solver 验证位置 | Stage 2 实战验证 | Skill 片段欠约束，孤立跑 Solver 无意义 |
| 失败经验处理 | 仅 Hard Gate 失败触发 | Solver 失败原因不可信，会污染 prompt |

### 4.3 Gatekeeper 两阶段

**Stage 1（CANDIDATE -> STAGING）：静态结构检查**
- 展开行数 >= 2
- DSLParser 逐行可解析
- 参数引用一致
- 与已有 Skill 去重 (相似度 < 0.8)
- 不跑 Solver（Skill 片段是欠约束的）

**Stage 2（STAGING -> PRODUCTION）：实战验证**
- 触发次数 >= 10
- 实战成功率 >= 60%（完整题目上下文中 Solver 成功）
- 贡献度剥离测试
- Anchor 结构对齐

---

## 五、已实现的代码

```
solver/skills/                        自演化 Skill 系统框架
  base.py                             SkillDefinition 基类
  evaluator.py                        Feedback Loop（语义 F1 + 内联验证）
  anchor.py                           ConstraintProblem 结构化指纹 Anchor
  gatekeeper.py                       三阶段程序化审核
  registry.py                         Skill 注册/检索/统计
  miner.py                            冷启动用 PatternMiner
  postprocessor.py                    辅助工具（压缩潜力评估）
  bank/composite_skills.py            10 个种子 Skill

server_scripts/
  generate_mixed_training_data.py     Phase IV 混合数据生成（支持 --diagnose）
  patch_annotation_id.py              annotation_id 补丁工具
  split_train_val.py                  分层 train/val 划分（防泄漏）

dataset/
  train_split.json                    训练集 (4354 条) -- 上传到服务器用这个
  val_split.json                      验证集 (472 条) -- 上传到服务器用这个
```

### 待实现（Phase IV 训练完成后）

1. Skill 提炼 prompt + 迭代修正逻辑
2. Skill 增强 prompt 构造（推理时注入 Skill Memory）
3. 混合输出解析器（解析 CALL + DSL）
4. Curriculum 选题模块
5. 推理端点 `/infer_with_evolution` 集成
6. Backward Construction（仅 Hard Gate 失败）

---

## 六、参考文档

| 文档 | 位置 | 说明 |
|------|------|------|
| 自演化设计方案 | `paperforreference/seplan.md` | v6.2，完整技术方案 |
| DSL 规范 | `dataset/dsl.md` | v1.8.8 |
| Phase III 训练报告 | `TRAINING_REPORT.md` | Phase I-III 三阶段对比 |
| 数据生成脚本 | `server_scripts/generate_mixed_training_data.py` | 支持 --diagnose 模式 |
| 数据增强 prompt | `dataset/CLAUDE_AUGMENTATION_PROMPT.md` | Claude API 语义增强指南 |
