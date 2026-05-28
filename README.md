# Paper Daily

<p align="center">
  <strong>每天自动追踪 arXiv 新论文，按你的研究方向打分，并生成中文技术摘要。</strong>
</p>

<p align="center">
  <img alt="GitHub Actions" src="https://img.shields.io/badge/automation-GitHub%20Actions-24292f?style=for-the-badge&logo=githubactions&logoColor=white" />
  <img alt="GitHub Pages" src="https://img.shields.io/badge/site-GitHub%20Pages-0f766e?style=for-the-badge&logo=githubpages&logoColor=white" />
  <img alt="arXiv" src="https://img.shields.io/badge/source-arXiv-b31b1b?style=for-the-badge&logo=arxiv&logoColor=white" />
  <img alt="LLM" src="https://img.shields.io/badge/summary-LLM%20optional-a15c18?style=for-the-badge" />
</p>

<p align="center">
  <a href="#-它能做什么">功能</a> ·
  <a href="#-先看这个哪些必须配置">配置清单</a> ·
  <a href="#-5-分钟部署">快速部署</a> ·
  <a href="#-配置-secrets重点">配置 Secrets</a> ·
  <a href="#-配置研究方向">研究方向</a> ·
  <a href="#-本地运行">本地运行</a>
</p>

## 它能做什么

Paper Daily 是一个完全托管在 GitHub 上的每日论文雷达：

- 每天北京时间 06:00 自动检索 arXiv，也支持手动运行。
- 按你配置的研究方向、关键词和 arXiv 分类计算匹配度。
- 用 DeepSeek、OpenAI 或其他 OpenAI-compatible 服务生成中文论文分析。
- 自动生成 `web/data/papers.json`，并把 `web/` 部署到 GitHub Pages。
- 前端页面只展示数据和筛选结果，不保存密钥，也不会调用模型 API。

## 工作流程

```text
Research Interests Issue / config/interests.json
                |
                v
GitHub Actions 定时运行 scripts/collect_papers.py
                |
                v
检索 arXiv -> 计算匹配度 -> 调用可选 LLM -> 写入 web/data/papers.json
                |
                v
GitHub Pages 发布 web/ 静态页面
```

数据保留规则：

- 当前运行发现的论文都会进入候选列表。
- 历史论文只保留匹配度为 `high` 和 `medium` 的条目，`low` 不跨天保留。
- 默认最多保存 300 篇论文或 5 MiB 数据。
- 超过限制时会先删除 `low`，再按论文发布时间从旧到新删除。

## 先看这个：哪些必须配置

新手可以先按下面这张表做。结论很简单：**GitHub Pages 必须开启；研究方向强烈建议改；模型 Secret 可选；运行参数都可以先不配置。**

| 配置项 | 必须自己配置吗 | 不配置会怎样 | 推荐做法 |
| --- | --- | --- | --- |
| GitHub Pages | 必须 | 页面不会发布出来 | 按「5 分钟部署」里的步骤开启 `GitHub Actions` 发布源 |
| `Research Interests` Issue | 强烈建议 | 会使用仓库自带的 `config/interests.json` 示例方向 | 新建标题为 `Research Interests` 的 issue，填自己的方向 |
| `DEEPSEEK_API_KEY` / `OPENAI_API_KEY` / `LLM_API_KEY` | 可选 | 仍会抓论文，但中文总结是基础摘要，不是高质量模型分析 | 推荐配置一个，DeepSeek 上手最简单 |
| `LLM_BASE_URL` | 可选 | 使用默认地址：DeepSeek key 对应 `https://api.deepseek.com/v1`，OpenAI key 对应 `https://api.openai.com/v1` | 只有使用第三方兼容服务时才必须填 |
| `LLM_MODEL` | 可选 | 使用默认模型：DeepSeek 为 `deepseek-chat`，OpenAI 为 `gpt-4o-mini` | 想换模型时再填 |
| `LOOKBACK_DAYS` | 可选 | 默认检索最近 `7` 天 | 论文太少可改成 `14` 或 `30` |
| `MAX_PER_TOPIC` | 可选 | 默认每个方向拉取 `25` 篇 | 方向少可调大，方向多可保持默认 |
| `MAX_SUMMARIES` | 可选 | 默认每次最多总结 `40` 篇 | 想省 API 费用可调小 |
| `LLM_CONCURRENCY` | 可选 | 默认并发 `2` | API 容易限流就改成 `1` |
| `MAX_STORED_PAPERS` | 可选 | 默认最多保存 `300` 篇 | 一般不用改 |
| `MAX_DATA_BYTES` | 可选 | 默认最大 `5242880` 字节，也就是 5 MiB | 一般不用改 |

最小可用配置：

```text
必须做：
1. 开启 GitHub Pages
2. 手动运行一次 Paper Daily workflow

建议做：
3. 新建 Research Interests issue，改成自己的研究方向
4. 添加一个模型 API key Secret，让总结质量更好
```

不用自己配置的东西：

- `GITHUB_TOKEN`：GitHub Actions 会自动提供，不需要你新建。
- `web/data/papers.json`：脚本会自动生成和更新。
- `requirements.txt` 里的依赖：当前项目只使用 Python 标准库，本地不装也能跑。

## 5 分钟部署

### 1. 准备仓库

把项目 Fork 到自己的 GitHub，或直接把代码推到自己的仓库。后续所有配置都在你的仓库里完成。

### 2. 开启 GitHub Pages（必须）

进入仓库页面：

```text
Settings -> Pages -> Build and deployment -> Source -> GitHub Actions
```

保存即可。这个项目不需要你手动选择分支，Actions 会把 `web/` 目录作为站点发布。

### 3. 配置模型 Secret（可选，但推荐）

如果只想先跑通流程，可以暂时跳过这一步。不配置模型 API key 时，项目仍会抓论文，只是中文总结会退化为基础摘要。

推荐至少配置一个模型 Secret，详细步骤见下面的「配置 Secrets」。

### 4. 配置研究方向（强烈建议）

最推荐的方式是使用 Issue 配置：

1. 打开仓库的 `Issues` 页面。
2. 点击 `New issue`。
3. 选择 `Research Interests` 模板。
4. 修改 JSON 里的研究方向、关键词和 arXiv 分类。
5. 保持标题为 `Research Interests`，提交 issue。

提交后，workflow 会自动运行一次。以后你只要编辑这个 issue，项目就会按新配置更新。

### 5. 手动运行一次

进入：

```text
Actions -> Paper Daily -> Run workflow -> Run workflow
```

运行成功后，回到：

```text
Settings -> Pages
```

页面里会显示你的 GitHub Pages 地址。

## 配置 Secrets（重点）

Secrets 是 GitHub 提供的密钥保险箱。API key 必须放在 Secrets 里，不要写进 README、代码、Issue、网页或 `config/interests.json`。

### 添加 Secret 的位置

进入你的 GitHub 仓库：

```text
Settings -> Secrets and variables -> Actions -> Secrets -> Repository secrets -> New repository secret
```

然后填写：

- `Name`：Secret 名称，例如 `DEEPSEEK_API_KEY`
- `Secret`：你的真实 API key，例如 `sk-...`

保存后，GitHub 页面不会再显示完整密钥，这是正常现象。

### DeepSeek 推荐配置

如果你使用 DeepSeek，添加这个 Repository secret：

| Name | Secret |
| --- | --- |
| `DEEPSEEK_API_KEY` | 你的 DeepSeek API key |

然后在 Variables 里添加：

| Name | Value |
| --- | --- |
| `LLM_BASE_URL` | `https://api.deepseek.com/v1` |
| `LLM_MODEL` | `deepseek-chat` |

Variables 的入口是：

```text
Settings -> Secrets and variables -> Actions -> Variables -> Repository variables -> New repository variable
```

说明：

- `DEEPSEEK_API_KEY` 是密钥，必须放 Secrets。
- `LLM_BASE_URL` 和 `LLM_MODEL` 不是密钥，可以放 Variables。
- 如果只添加了 `DEEPSEEK_API_KEY`，代码会自动使用 DeepSeek 默认地址和 `deepseek-chat`。

### OpenAI 配置

如果你使用 OpenAI，添加这个 Repository secret：

| Name | Secret |
| --- | --- |
| `OPENAI_API_KEY` | 你的 OpenAI API key |

可选 Variables：

| Name | Value |
| --- | --- |
| `LLM_BASE_URL` | `https://api.openai.com/v1` |
| `LLM_MODEL` | `gpt-4o-mini` 或其他支持 JSON 输出的模型 |

如果只添加了 `OPENAI_API_KEY`，代码会默认使用 `https://api.openai.com/v1` 和 `gpt-4o-mini`。

### 其他 OpenAI-compatible 服务

如果你使用兼容 `/chat/completions` 的服务，添加：

| 类型 | Name | Value |
| --- | --- | --- |
| Secret | `LLM_API_KEY` | 你的服务商 API key |
| Variable | `LLM_BASE_URL` | 服务商 API 地址，例如 `https://example.com/v1` |
| Variable | `LLM_MODEL` | 服务商模型名 |

注意：`LLM_BASE_URL` 只写到 `/v1`，不要写到 `/chat/completions`。脚本会自动拼接完整接口地址。

### Secret 优先级

脚本会按下面顺序寻找 API key：

1. `LLM_API_KEY`
2. `OPENAI_API_KEY`
3. `DEEPSEEK_API_KEY`

如果你同时配置了多个 key，排在前面的会优先生效。新手建议只配置一种，避免自己搞混。

### 常见 Secret 错误

| 现象 | 常见原因 | 处理方式 |
| --- | --- | --- |
| 页面显示“基础”总结 | 没有配置 API key，或 key 名称写错 | 检查是否叫 `DEEPSEEK_API_KEY` / `OPENAI_API_KEY` / `LLM_API_KEY` |
| Actions 里出现 401 / unauthorized | API key 无效或复制时多了空格 | 重新复制 key，更新 Secret |
| Actions 里出现 404 model not found | `LLM_MODEL` 写错 | 改成服务商实际支持的模型名 |
| Actions 里出现 connection / endpoint 错误 | `LLM_BASE_URL` 写错 | 确认地址类似 `https://api.deepseek.com/v1` |
| Issue 改了但结果没变 | Issue 标题不是 `Research Interests`，或 JSON 格式错误 | 保持标题完全一致，并检查 JSON 逗号和引号 |

## 可选运行参数

这些参数放在 Repository Variables 里即可，入口同上：

```text
Settings -> Secrets and variables -> Actions -> Variables
```

| Name | 默认值 | 作用 |
| --- | ---: | --- |
| `LOOKBACK_DAYS` | `7` | 检索最近多少天的论文 |
| `MAX_PER_TOPIC` | `25` | 每个研究方向最多从 arXiv 拉取多少篇 |
| `MAX_SUMMARIES` | `40` | 每次最多调用模型总结多少篇 |
| `LLM_CONCURRENCY` | `2` | 模型 API 并发数，太高可能触发限流 |
| `MAX_STORED_PAPERS` | `300` | 最多保存多少篇论文，设为 `0` 表示不按数量裁剪 |
| `MAX_DATA_BYTES` | `5242880` | `web/data/papers.json` 最大字节数，设为 `0` 表示不按大小裁剪 |

一般建议新手只改 `LOOKBACK_DAYS`、`MAX_PER_TOPIC` 和 `MAX_SUMMARIES`。

## 配置研究方向

项目会优先读取标题为 `Research Interests` 的 open issue。如果找不到这个 issue，才会读取仓库里的 `config/interests.json`。

一个方向的格式如下：

```json
{
  "id": "llm_low_precision_quantization",
  "name": "大模型低精度量化",
  "description": "关注大语言模型低比特量化、混合精度、量化感知训练、后训练量化、权重量化、激活量化、KV cache 量化以及量化对推理性能和精度的影响。",
  "keywords": [
    "large language model quantization",
    "LLM quantization",
    "low-bit quantization",
    "post-training quantization",
    "INT4",
    "FP8"
  ],
  "arxiv_categories": ["cs.CL", "cs.LG", "cs.AI", "cs.DC"]
}
```

字段说明：

| 字段 | 是否必填 | 怎么写 |
| --- | --- | --- |
| `id` | 建议填写 | 英文、数字、下划线组成的唯一 ID，不要和其他方向重复 |
| `name` | 必填 | 页面上显示的方向名称 |
| `description` | 建议填写 | 用中文说明你到底关心什么，越具体越好 |
| `keywords` | 建议填写 | 英文关键词效果更好，因为 arXiv 标题和摘要主要是英文 |
| `arxiv_categories` | 建议填写 | arXiv 分类，例如 `cs.CL`、`cs.LG`、`cs.AI`、`cs.DC` |

完整配置必须是下面这种结构：

```json
{
  "topics": [
    {
      "id": "your_topic_id",
      "name": "你的研究方向",
      "description": "你关心的问题、方法、系统或应用场景。",
      "keywords": ["keyword one", "keyword two"],
      "arxiv_categories": ["cs.CL", "cs.LG"]
    }
  ]
}
```

建议：

- 一个方向放 6 到 12 个关键词，不要塞太多泛词。
- 关键词优先写论文里常见的英文表达。
- `description` 可以写中文，模型总结会用它判断相关性。
- 如果你不确定分类，可以先用 `cs.CL`、`cs.LG`、`cs.AI`，再根据结果慢慢调。

## 本地运行

本地运行主要用于调试配置和页面，不需要安装额外 Python 包。

### 1. 准备 Python

建议使用 Python 3.12：

```bash
python -V
```

如果你想使用虚拟环境：

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

当前项目只使用 Python 标准库，`requirements.txt` 只是预留给未来依赖。

### 2. 不调用模型，直接抓论文

```bash
python scripts/collect_papers.py --days 7 --max-per-topic 10 --max-summaries 5
```

生成结果会写入：

```text
web/data/papers.json
```

### 3. 本地调用 DeepSeek

```bash
export DEEPSEEK_API_KEY="sk-你的真实密钥"
export LLM_BASE_URL="https://api.deepseek.com/v1"
export LLM_MODEL="deepseek-chat"
python scripts/collect_papers.py --days 7 --max-per-topic 10 --max-summaries 5
```

### 4. 本地调用 OpenAI

```bash
export OPENAI_API_KEY="sk-你的真实密钥"
export LLM_BASE_URL="https://api.openai.com/v1"
export LLM_MODEL="gpt-4o-mini"
python scripts/collect_papers.py --days 7 --max-per-topic 10 --max-summaries 5
```

不要把真实密钥提交到 Git。如果你临时写了 `.env`，也不要提交；项目的 `.gitignore` 已经忽略 `.env`。

### 5. 预览网页

```bash
python -m http.server 8000 --directory web
```

浏览器打开：

```text
http://localhost:8000
```

## 手动触发和自动触发

workflow 会在这些情况下运行：

- 每天北京时间 06:00 自动运行。
- 推送到 `main` 分支时运行。
- 手动点击 `Actions -> Paper Daily -> Run workflow` 时运行。
- 标题为 `Research Interests` 的 issue 被创建、编辑或重新打开时运行。

GitHub Actions 的 cron 使用 UTC 时间，仓库里配置的是：

```yaml
schedule:
  - cron: "0 22 * * *"
```

也就是 UTC 22:00，对应北京时间第二天 06:00。

## 排查问题

### 看 Actions 日志

进入：

```text
Actions -> Paper Daily -> 最近一次运行 -> collect-and-deploy
```

重点看 `Collect papers` 这一步。

常见日志：

- `Wrote N papers to web/data/papers.json`：运行成功。
- `Warning: LLM summary failed`：模型调用失败，会自动退回基础摘要。
- `Warning: cannot read GitHub issues`：读取 Issue 配置失败，会退回 `config/interests.json`。
- `All arXiv requests failed`：arXiv 暂时不可用，会尽量保留已有数据。

### 页面没有更新

可以按顺序检查：

1. `Actions` 最近一次是否成功。
2. `Settings -> Pages` 是否选择了 `GitHub Actions`。
3. `web/data/papers.json` 是否在最近一次运行中被更新。
4. 浏览器是否缓存了旧数据，可以强制刷新页面。

### 没有匹配到论文

可以尝试：

- 增大 `LOOKBACK_DAYS`，例如改成 `14`。
- 增大 `MAX_PER_TOPIC`，例如改成 `50`。
- 给每个方向补充更常见的英文关键词。
- 放宽 arXiv 分类范围。

## 目录结构

```text
.
├── .github/
│   ├── ISSUE_TEMPLATE/research-interests.md
│   └── workflows/paper-daily.yml
├── config/
│   └── interests.json
├── scripts/
│   └── collect_papers.py
├── tests/
│   └── test_collect_papers.py
└── web/
    ├── app.js
    ├── data/papers.json
    ├── index.html
    └── styles.css
```

## 数据源

当前版本使用 arXiv API。后续可以继续扩展 Semantic Scholar、OpenReview、Papers with Code、GitHub Trending 和 Hugging Face Papers。
