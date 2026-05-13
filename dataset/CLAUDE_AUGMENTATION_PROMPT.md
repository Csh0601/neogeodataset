# Claude Code Prompt: Phase III Semantic Augmentation Data Regeneration

## Task

You need to regenerate the Phase III semantic augmentation training data that was lost. The task is: for each of the 1292 gold annotations, rewrite the Chinese geometry description in 3 different styles while keeping the DSL output unchanged.

## Context

- Project: NeoGeo, Chinese geometry text to DSL conversion
- The original augmentation was done by `server_scripts/augment_semantic.py` using Qwen2.5-14B on the GPU server
- Since the data is lost and re-running on server would take hours, you will do the rewriting directly using your language abilities
- You are MUCH better at Chinese rewriting than Qwen2.5-14B, so quality will be higher

## Input

Read all JSON files from `dataset/annotations/`. Each file looks like:

```json
{
  "id": "00001",
  "text": "点 A、B、C是半径为4的圆O 上的点，角AOB=70度, 角ACB=35度",
  "dsl": ["circle(O, r=4)", "on(A, circle_O)", "on(B, circle_O)", "on(C, circle_O)", "angle(AOB)=70", "angle(ACB)=35"],
  "image_path": "images/00001.png"
}
```

## Output

For each annotation, generate a JSON file or append to a single JSON list. Each augmented sample must be in this exact format:

```json
{
  "messages": [
    {
      "role": "system",
      "content": "你是一个专业的几何DSL生成器。你的任务是将中文几何描述转换为结构化的DSL程序。\n\nDSL语法规则：\n1. 关键字全小写：triangle, circle, parallel, perpendicular, midpoint, foot, angle, on\n2. 点名大写：A, B, C, A1, B_prime\n3. 线段用两点连写：AB, CD\n4. 每条语句表达一个原子几何事实\n5. 角度单位为度数\n\n【强制决策规则】必须按照以下顺序生成DSL：\n1. 保证语法合法（最高优先级）\n2. 明确所有几何对象，先定义点与图形，再使用它们\n3. 所有复杂语句，必须拆分成多个原子语句\n4. 先输出结构性约束（parallel, perpendicular, midpoint）\n5. 再输出度量约束（angle=30, length=5）\n6. 不允许生成未定义的点名\n7. 不允许出现自然语言、解释或注释"
    },
    {
      "role": "user",
      "content": "将以下中文几何描述转换为DSL程序：{rewritten_text}"
    },
    {
      "role": "assistant",
      "content": "{original_dsl_joined_by_newline}"
    }
  ],
  "augmentation": {
    "type": "semantic_rewrite",
    "style": "style_A",
    "original_text": "{original_text}"
  }
}
```

## Rewriting Rules (CRITICAL)

For each annotation, produce 3 rewrites:

**Style A (oral/casual)**: Like a teacher explaining in class. Natural, conversational.
- "连接AB" becomes "作线段AB"
- "AB=AC" becomes "三角形ABC是等腰三角形，其中AB与AC长度相等"

**Style B (minimal)**: Only list conditions, remove all filler words.
- "在三角形ABC中，AB=AC，角A=60度" becomes "三角形ABC，AB=AC，角A=60度"

**Style C (formal)**: Add formal guiding phrases like "如图所示", "已知", "设".
- "圆O的半径为4" becomes "如图所示，已知圆O，其半径为4"

## Absolute Constraints (MUST NOT VIOLATE)

1. **Numbers must be EXACTLY the same.** If original has "60", rewrite must have "60". If original has "2根号3", rewrite must have "2根号3". No rounding, no conversion, no omission.

2. **All point names must appear.** If original mentions A, B, C, D, the rewrite must mention A, B, C, D. Allow up to 30% missing for non-core points, but core triangle/circle points must all be present.

3. **No geometric information added or removed.** Do not add constraints that don't exist in the original. Do not omit constraints that exist in the original.

4. **DSL output is NEVER modified.** The assistant content is always the original DSL joined by newlines, copied verbatim.

## Validation (apply to every rewrite)

```python
def validate(original_text, rewritten_text, dsl_text):
    # 1. Numbers must match exactly
    orig_nums = set(re.findall(r'\d+\.?\d*', original_text))
    new_nums = set(re.findall(r'\d+\.?\d*', rewritten_text))
    assert orig_nums == new_nums, f"Number mismatch: {orig_nums} vs {new_nums}"

    # 2. Point name recall >= 70%
    dsl_points = set(re.findall(r'\b([A-Z](?:[0-9]*|_prime)?)\b', dsl_text)) - {'O', 'X', 'Y'}
    rewrite_points = set(re.findall(r'[A-Z]', rewritten_text))
    if dsl_points:
        missing = dsl_points - rewrite_points
        assert len(missing) <= len(dsl_points) * 0.3, f"Too many points missing: {missing}"
```

If a rewrite fails validation, discard it and try again. It is acceptable to produce 2 rewrites instead of 3 for some samples, but try to get 3 for most.

## Implementation Approach

Write a Python script `server_scripts/regenerate_augmentation.py` that:

1. Reads all 1292 annotations from `dataset/annotations/`
2. For each annotation, calls the **Anthropic API** (claude-sonnet-4-20250514) to generate 3 rewrites
3. Validates each rewrite (numbers match, point names present)
4. Discards invalid rewrites, keeps valid ones
5. Saves to `dataset/train_augmented.json`
6. Supports `--resume` to continue from where it left off (in case of interruption)

**Use the Anthropic API, NOT rule-based transformations.** Rule-based regex rewriting has extremely low coverage on Chinese geometry text (less than 20% usable), and produces grammatically broken sentences. The LLM rewriting is higher quality by an order of magnitude.

**Environment setup (MUST run before the script):**

```bash
# Proxy (required for API access in this environment)
set https_proxy=http://127.0.0.1:33210
set http_proxy=http://127.0.0.1:33210
set all_proxy=socks5://127.0.0.1:33211
```

The environment has `ANTHROPIC_AUTH_TOKEN` (not the default `ANTHROPIC_API_KEY`) and `ANTHROPIC_BASE_URL=https://anyrouter.top`. The script must explicitly pass the key since the env var name is non-standard:

```python
import anthropic
import time
import os

client = anthropic.Anthropic(
    api_key=os.environ["ANTHROPIC_AUTH_TOKEN"],
    # ANTHROPIC_BASE_URL is auto-read from env
)

def generate_rewrites(original_text: str) -> Dict[str, str]:
    """Call Claude to generate 3 style rewrites."""
    prompt = REWRITE_PROMPT.format(original_text=original_text)

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=800,
        messages=[{"role": "user", "content": prompt}]
    )

    # Parse JSON from response
    text = response.content[0].text
    json_match = re.search(r'\{[^{}]*\}', text, re.DOTALL)
    if json_match:
        return json.loads(json_match.group())
    return {}
```

Use the same REWRITE_PROMPT template from Section "Rewriting Rules" above (the one with style_A/B/C examples).

**Rate limiting and batching:**
- Process annotations sequentially (not in parallel) to stay within API rate limits
- Add `time.sleep(0.5)` between requests
- Save progress every 50 annotations to a checkpoint file
- Support `--resume` flag to restart from the last checkpoint
- Retry failed requests up to 3 times with exponential backoff

**Cost estimate:**
- 1292 annotations, ~100 input tokens + ~200 output tokens per request
- Total: ~400K tokens, approximately $1-2 USD with claude-sonnet-4-20250514

For each annotation:
1. Call Claude API with REWRITE_PROMPT
2. Parse JSON response (style_A, style_B, style_C)
3. Validate each rewrite (numbers match, point names present)
4. Keep valid ones, discard invalid
5. Save checkpoint every 50 annotations

## Expected Output Stats

- Input: 1292 annotations
- Output: ~3000-3800 augmented samples (1292 x ~2.5 average valid rewrites)
- Save as: `dataset/train_augmented.json`
- Checkpoint file: `dataset/augmentation_checkpoint.json` (progress tracking)
- Estimated API cost: ~$1-2 USD
- Estimated time: 30-60 minutes

## After Generation

The augmented data will be merged with original data using:
```bash
python server_scripts/merge_datasets.py --original train_original.json --augmented train_augmented.json --output train_phase3.json
```

Then the Phase IV mixed data (train_phase4_mixed.json) will be merged on top for final training.
