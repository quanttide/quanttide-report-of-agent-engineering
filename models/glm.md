# GLM模型能力测评报告

## 实验设置

- 供应商：智谱官网
- 接入方式：API密钥
- 模型：GLM 5.2和GLM 5v
- 智能体：OpenCode和Zed Agent

## 实验1：使用OpenCode修复Zed Agent配置

本报告记录参照智谱（Zhipu/Z.AI）官方文档，修复 Zed Agent 中 GLM 系列模型（GLM-5.2、GLM-5V-Turbo）OpenAI-compatible provider 配置的工作。

### 意图

参照智谱官方文档，修复 `~/.config/zed/settings.json` 中 Z.AI（智谱）OpenAI-compatible provider 的 Agent 配置，使 GLM-5.2 与 GLM-5V-Turbo 两个模型都能在 Zed Agent 中正确可用。

### 过程

1. **取证**：并行抓取智谱官方 GLM-5V-Turbo 文档、读取本地 Zed 配置；5.2 的初探 URL 404 后，通过 `llms.txt` 索引定位到正确路径 `glm-5.2.md` 并抓取。
2. **对照 Zed 约定**：抓取 Zed 的 `llm-providers` → `use-api-access` 文档，确认 `api_url` 为 base URL（无尾斜杠、Zed 追加 `/chat/completions`）、默认 capabilities、以及 provider ID `Z.AI` 对应环境变量 `Z_AI_API_KEY`。
3. **定位三类缺陷**：`api_url` 前导空格、5.2 token 限额全错、缺 5v-turbo。
4. **改写**：单次 Edit 替换整个 `language_models` 块，补两个模型规格 + `display_name`。
5. **校验**：因文件是 JSONC（含注释/尾逗号），手写字符串感知的注释剥离器校验通过。

### 修复项对照

| 修复项 | 修复前 | 修复后（官方文档） |
|---|---|---|
| `api_url` 前导空格 | `" https://..."`（URL 畸形，请求会失败） | `"https://open.bigmodel.cn/api/paas/v4"` |
| GLM-5.2 上下文 | 200000 | 1000000（1M） |
| GLM-5.2 最大输出 | 32000 | 128000（128K） |
| GLM-5.2 max_completion_tokens | 200000 | 128000 |
| GLM-5V-Turbo | 缺失 | 已补全：200K 上下文 / 128K 输出 / `images: true`（多模态） |

### 最终配置

```jsonc
"language_models": {
  "openai_compatible": {
    "Z.AI": {
      "api_url": "https://open.bigmodel.cn/api/paas/v4",
      "available_models": [
        {
          "name": "glm-5.2",
          "display_name": "GLM-5.2",
          "max_tokens": 1000000,
          "max_output_tokens": 128000,
          "max_completion_tokens": 128000,
          "capabilities": {
            "tools": true,
            "images": false,
            "parallel_tool_calls": false,
            "prompt_cache_key": false,
            "chat_completions": true,
            "interleaved_reasoning": false
          }
        },
        {
          "name": "glm-5v-turbo",
          "display_name": "GLM-5V-Turbo",
          "max_tokens": 200000,
          "max_output_tokens": 128000,
          "max_completion_tokens": 128000,
          "capabilities": {
            "tools": true,
            "images": true,
            "parallel_tool_calls": false,
            "prompt_cache_key": false,
            "chat_completions": true,
            "interleaved_reasoning": false
          }
        }
      ]
    }
  }
}
```

### 经验

- **先查权威再动手**：模型限额（上下文/输出 token）必须以官方文档为准，凭记忆填会出错（5.2 实为 1M/128K 而非配置里的 200K/32K）。
- **隐蔽 bug 在细节**：`api_url` 前导空格不会触发语法错误，但会让请求打到错误地址——JSON 校验通过不代表配置正确。
- **校验要尊重格式**：Zed 用 JSONC，标准 `json.load` 会误判；且粗暴正则删 `//` 会误伤 URL 里的 `https://`，需字符串感知剥离。
- **环境变量派生规则**：openai_compatible 的 key 环境变量由 provider ID 转大写蛇形得来（`Z.AI`→`Z_AI_API_KEY`），这点容易遗漏，是配置"看起来对却调不通"的常见根因。
- **能力位按模型特性设**：5v-turbo 多模态需 `images: true`；思考模式走独立 `reasoning_content` 字段而非交错，故 `interleaved_reasoning` 留 `false`。

### 参考文档

- 智谱 GLM-5.2：<https://docs.bigmodel.cn/cn/guide/models/text/glm-5.2>
- 智谱 GLM-5V-Turbo：<https://docs.bigmodel.cn/cn/guide/models/vlm/glm-5v-turbo>
- Zed LLM Providers：<https://zed.dev/docs/ai/llm-providers>
- Zed Use API Access（OpenAI-compatible）：<https://zed.dev/docs/ai/use-api-access>
