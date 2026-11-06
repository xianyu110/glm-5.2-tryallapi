# GLM-5.2 API 调用教程：用 TryAllAPI 做 OpenAI / Anthropic 兼容接入

> **GLM-5.2**（TryAllAPI 模型 ID：`glm-5.2`）是智谱面向长任务时代的旗舰对话模型。价格接口描述提到可用 1M 上下文、更稳定的长程任务执行，以及从需求到多平台部署的完整开发链路。标签为「对话、工具」。本文记录如何通过聚合站 **[TryAllAPI](https://tryallapi.com/)**，从模型广场 / 价格页选型、读基价与分组倍率，到 `POST /v1/chat/completions` 的完整接入流程。价格与分组倍率取数 **2026-11-06（北京）**，**以页面实时为准**；示例仅为请求形态说明，未跑通真实长任务计费。

**Keywords:** GLM-5.2 · glm-5.2 · 智谱 · ChatGLM · TryAllAPI · OpenAI 兼容 · `/v1/chat/completions` · Anthropic `/v1/messages`

---

## 目录

- [一、为什么用聚合站调 GLM](#一为什么用聚合站调-glm)
- [二、在价格页 / 模型广场找到 GLM-5.2](#二在价格页--模型广场找到-GLM-5.2)
- [三、模型怎么选：与相邻档对照](#三模型怎么选与相邻档对照)
- [四、价格怎么读](#四价格怎么读)
- [五、调用流程：Key → Base URL → chat/completions](#五调用流程key--base-url--chatcompletions)
- [六、curl / SDK 最小示例](#六curl--sdk-最小示例)
- [七、常见注意](#七常见注意)
- [FAQ](#faq)
- [八、小结与试用入口](#八小结与试用入口)

---

## 一、为什么用聚合站调 GLM

只用 GLM、需要智谱官方发票或套餐时，直连智谱开放平台即可。如果同一个项目里还要调 Claude、GPT、Qwen，希望一个 Key、一份账单，聚合站更顺手：

1. **统一鉴权**：一个 Bearer Key，切换 `model` 字段即可；
2. **OpenAI 兼容形态**：Base URL 固定为 `https://tryallapi.com/v1`；价格接口同时标注 `anthropic` 端点，需要 Anthropic SDK / Claude Code 时可另看站点说明；
3. **模型广场可对照**：标签、分组倍率、端点类型一目了然。

代价是多一跳：延迟与可用性取决于分组上游，建议先用小流量探针。注册试用入口：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 二、在价格页 / 模型广场找到 GLM-5.2

打开 [价格页](https://tryallapi.com/pricing) 或模型广场，按供应商 **ChatGLM**、类型 **对话**、关键词 `glm-5.2` 筛选。详情卡截图如下：

![GLM-5.2 模型广场详情卡：模型 ID、API 与分组价格](docs/model-square.png)

详情卡要点（取自价格接口 2026-11-06（北京） 快照）：

| 字段 | 内容 |
| --- | --- |
| **模型 ID** | `glm-5.2` |
| **供应商** | ChatGLM（智谱） |
| **标签** | 对话、工具 |
| **端点类型** | `anthropic`、`openai`（对话主路径：`POST /v1/chat/completions`；另有 Anthropic 形态以站点说明为准） |
| **可用分组** | Self-Deployed-2、Alibaba-2、Self-Deployed-3、Alibaba-3、glm-1 |
| **计费** | 按量（token） |

文中 UI 品牌可能显示为模型广场 / 第三方壳，实际对接产品以 [TryAllAPI](https://tryallapi.com/) 控制台与价格页为准。

---

## 三、模型怎么选：与相邻档对照

只根据价格接口可见信息做对照，**不编造未出现的能力差异**。基价按价格接口换算（$/1M，未乘分组倍率）：

| 模型 ID（可见） | 基价 输入 / 输出 / 缓存读 | 怎么理解 |
| --- | --- | --- |
| **`glm-5.2`** | $1.4 / $4.4 / $0.26 | 本文主线：长任务旗舰，广场描述含可用 1M 上下文 |
| `glm-5.3` | $1.4 / $4.4 / $0.26 | 同厂商最新旗舰；广场描述称与 5.2 同基础模型、提升来自后训练 |
| `glm-5.3-flash` | $0.15 / $0.5 / $0.03 | 同代 Flash 档；标签含识图，基价更低 |
| `glm-5.1` | $1.4 / $4.4 / $0.26 | 同系列上一档；标签为对话、思考 |
| `glm-5` | $1 / $3.2 / — | 同厂商前代旗舰；基价略低 |

实务上：经 TryAllAPI 调用统一写精确字符串 `glm-5.2`；遇限流或分组无货，再在同厂商可见 ID 之间切换，而不是改 Base URL。

---

## 四、价格怎么读

对话模型常见公式：**≈ 基价 × 分组倍率**（以控制台为准）。取数快照：**2026-11-06（北京）**；**以页面实时为准**。

换算约定：基价输入 $= model_ratio×2；输出 $= 输入×completion_ratio；缓存读 $= 输入×cache_ratio（若有）；缓存创建（若有）= 输入×cache_creation_ratio。分组估价 = 基价 × group_ratio。

### 基价（每百万 tokens，由价格接口换算）

| 计费项 | 基价（$/1M，未乘分组） |
| --- | --- |
| 输入 | **$1.4** |
| 输出 | **$4.4** |
| 缓存读 | **$0.26** |

公开接口按 **基价 × group_ratio** 估算即可；**实际扣费以控制台为准**。

本模型快照含缓存读倍率（cache_ratio），未见 cache_creation_ratio。

### 分组价格估算（基价 × group_ratio，$/1M）

| 分组 | group_ratio | 输入估 | 缓存读估 | 输出估 |
| --- | --- | --- | --- | --- |
| **Self-Deployed-2** | 1.1 | ≈ $1.54 | ≈ $0.286 | ≈ $4.84 |
| **Alibaba-2** | 1.65 | ≈ $2.31 | ≈ $0.429 | ≈ $7.26 |
| **Self-Deployed-3** | 1.65 | ≈ $2.31 | ≈ $0.429 | ≈ $7.26 |
| **Alibaba-3** | 2.42 | ≈ $3.388 | ≈ $0.6292 | ≈ $10.648 |
| **glm-1** | 2.42 | ≈ $3.388 | ≈ $0.6292 | ≈ $10.648 |

**最终扣费以控制台流水为准**。卡片展示价取自最低 `group_ratio` 分组 **Self-Deployed-2**（与定价页公开数字一致）。

---

## 五、调用流程：Key → Base URL → chat/completions

```text
控制台创建 API Key（选本模型 enable_groups 中的分组）
  → Base URL = https://tryallapi.com/v1
  → POST /v1/chat/completions   （model=glm-5.2）
  → 读 choices[0].message.content（或流式 delta）
```

建议：

- **超时**：长输出 / 思考模式建议开流式并调高 timeout；
- **密钥**：只放环境变量（如 `TRYALLAPI_KEY`），勿提交仓库；
- **端点类型**：价格接口标注端点类型：`anthropic`、`openai`。本文以 OpenAI 兼容 `chat/completions` 为主，其它端点以站点说明为准。

---

## 六、curl / SDK 最小示例

占位 Key `sk-xxx`（**不要**填入真实 Key）。

### 1）OpenAI 兼容 chat/completions

```bash
curl -sS https://tryallapi.com/v1/chat/completions \
  -H 'Authorization: Bearer sk-xxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "glm-5.2",
    "messages": [
      {"role": "user", "content": "把这份多步调研任务拆成可执行的验收清单（不超过 8 条）"}
    ]
  }'
```

### 2）Python（OpenAI SDK）

```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-xxx",  # 建议改为读取环境变量
    base_url="https://tryallapi.com/v1",
    timeout=180,
)

r = client.chat.completions.create(
    model="glm-5.2",
    messages=[{"role": "user", "content": "用三句话说明如何给这段多步 agent 工作流写回归用例。"}],
)
print(r.choices[0].message.content)
```

Cursor：Settings → Models → OpenAI API Key + Override Base URL `https://tryallapi.com/v1` → 自定义模型 `glm-5.2`。Dify：OpenAI-API-compatible，endpoint 填 `https://tryallapi.com/v1`。

---

## 七、常见注意

1. **模型名**：请求体 `"model"` 必须写广场上的精确 ID `glm-5.2`（含连字符 / 点号，大小写敏感）；写混成简称或漏掉后缀容易 404。
2. **基价 × 倍率**：估算用价格接口快照；峰谷 / 促销字段若页面另有说明，以控制台为准。
3. **分组**：403 多为分组无货，换 `enable_groups` 里其他分组；低倍率分组先抽检质量与延迟。
4. **报错速查**：401 查 Bearer / 令牌；404 确认模型名且 base_url 含 `/v1`；429 退避或换分组。
5. **合规**：第三方聚合不是 ChatGLM 官方服务，敏感代码先看条款。

---

## FAQ

### GLM-5.2 的模型 ID 是什么？

TryAllAPI 上是 **`glm-5.2`**。请求体 `"model"` 填该字符串。

### TryAllAPI 的对话 endpoint 是什么？

- Base URL：`https://tryallapi.com/v1`
- 对话：`POST /v1/chat/completions`
- 价格接口标注端点类型：`anthropic`、`openai`。本文以 OpenAI 兼容 `chat/completions` 为主，其它端点以站点说明为准。

### 基价大概是多少？

按 2026-11-06（北京） 价格接口快照：输入约 $1.4 / 输出约 $4.4 / 缓存读约 $0.26（$/1M，未乘分组）。再 × `group_ratio` 得分组估价；**以页面实时为准**。

### 如何试用 TryAllAPI？

站点首页：[https://tryallapi.com/](https://tryallapi.com/)；注册试用入口：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 八、小结与试用入口

把 GLM-5.2 接到业务里，核心是：**在价格页确认 model id 与 OpenAI 兼容 Base URL → 读懂基价 × 分组倍率 → 用 chat/completions 先跑通非流式 ping，再开流式 / 接工具调用**。

| 资源 | 链接 |
| --- | --- |
| TryAllAPI 首页 | [https://tryallapi.com/](https://tryallapi.com/) |
| 注册试用（aff） | [https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A) |
| 价格页 | [tryallapi.com/pricing](https://tryallapi.com/pricing) |

价格取数 2026-11-06（北京），以页面实时为准。

---

## License

本文档以个人接入笔记形式发布，示例代码可自由复用。截图与模型广场文案归原平台；价格以实时页面为准。TryAllAPI 为第三方聚合，非 ChatGLM 官方产品。
