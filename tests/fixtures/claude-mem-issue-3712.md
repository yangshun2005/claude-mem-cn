Issue: https://github.com/thedotmack/claude-mem/issues/3712
标题：OpenRouter 提供商与需要 max_completion_tokens 的 GPT-5 模型不兼容

claude-mem 生成的请求包含：

"max_tokens":4096

上游模型只接受：

"max_completion_tokens":4096

错误：

Unsupported parameter: 'max_tokens' is not supported with this model.
Use 'max_completion_tokens' instead.