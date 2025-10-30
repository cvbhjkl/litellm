# Gemini 2.5 Pro CUA 兼容性实现方案

## 当前状态

**结论：LiteLLM 目前尚未兼容 `gemini-2.5-pro-cua` 模型**

### 现状分析

1. **已有模型支持**
   - ✅ `gemini-2.5-pro` - 已在 `model_prices_and_context_window.json` 中配置（Vertex AI版本）
   - ✅ `gemini/gemini-2.5-pro` - 已在 `model_prices_and_context_window.json` 中配置（Google AI Studio版本）

2. **缺失的模型**
   - ❌ `gemini-2.5-pro-cua` - 未在模型配置文件中
   - ❌ `gemini/gemini-2.5-pro-cua` - 未在模型配置文件中

3. **关于 "CUA"**
   - CUA 可能是 Google 的区域变体标识（例如：China User Access）
   - 需要确认该模型与标准 `gemini-2.5-pro` 的差异（定价、功能限制等）

## 实现方案

### 方案概述

在 `model_prices_and_context_window.json` 中添加 `gemini-2.5-pro-cua` 相关配置，使其能够被 LiteLLM 识别和使用。

### 详细步骤

#### 步骤 1: 确定模型规格和定价

**需要确认的信息：**
- 模型的实际定价（输入/输出 token 成本）
- 上下文窗口大小（max_input_tokens, max_output_tokens）
- 功能支持情况（reasoning, vision, function calling 等）
- 速率限制（RPM, TPM）
- API 端点要求

**信息来源：**
- Google AI Studio 文档
- Vertex AI 文档（如果也支持 CUA 版本）
- 实际 API 测试

#### 步骤 2: 在 model_prices_and_context_window.json 添加配置

**需要添加的配置项：**

```json
{
  "gemini-2.5-pro-cua": {
    // Vertex AI 版本的配置（如果支持）
  },
  "gemini/gemini-2.5-pro-cua": {
    // Google AI Studio 版本的配置
    // 参考 gemini/gemini-2.5-pro 的配置结构
  }
}
```

**配置位置：**
- 文件：`model_prices_and_context_window.json`
- 建议位置：紧随 `gemini-2.5-pro` 和 `gemini/gemini-2.5-pro` 之后（约 7864 行之后）

#### 步骤 3: 验证模型识别逻辑

LiteLLM 的模型识别逻辑（在 `litellm/utils.py` 的 `_get_model_info_helper` 函数中）会按以下顺序尝试匹配：

1. `custom_llm_provider/model`（例如：`gemini/gemini-2.5-pro-cua`）
2. `model`（例如：`gemini-2.5-pro-cua`）
3. `combined_stripped_model_name`（去除版本号后的组合名称）
4. `stripped_model_name`（去除版本号后的名称）
5. `split_model`（去除前缀后的名称）

**验证点：**
- ✅ 确保 `_get_potential_model_names` 函数能正确处理 `-cua` 后缀
- ✅ 确保模型名称匹配逻辑能够找到新添加的配置

#### 步骤 4: 测试兼容性

**测试场景：**

1. **基本调用测试**
   ```python
   import litellm
   
   response = litellm.completion(
       model="gemini/gemini-2.5-pro-cua",
       messages=[{"role": "user", "content": "Hello"}]
   )
   ```

2. **模型信息获取测试**
   ```python
   model_info = litellm.get_model_info("gemini/gemini-2.5-pro-cua")
   assert model_info is not None
   ```

3. **功能特性测试**
   - Reasoning 支持（如果支持）
   - Function calling
   - Vision（多模态）
   - Streaming
   - Token 计数

4. **成本计算测试**
   - 验证 input/output cost 是否正确计算

#### 步骤 5: 更新相关文档（可选）

如果模型有特殊说明，可能需要更新：
- `docs/my-website/docs/providers/gemini.md`
- Release notes（如果作为新功能发布）

### 实施检查清单

- [ ] 确认 `gemini-2.5-pro-cua` 的实际定价和规格
- [ ] 在 `model_prices_and_context_window.json` 添加配置
- [ ] 验证模型名称识别逻辑
- [ ] 编写并运行单元测试
- [ ] 进行集成测试（真实 API 调用）
- [ ] 验证成本计算准确性
- [ ] 检查是否有特殊的功能限制需要处理
- [ ] 更新相关文档

### 注意事项

1. **定价信息**
   - 如果 CUA 版本的定价与标准版本不同，确保使用正确的定价
   - 如果定价信息未知，可以使用临时值（需标注为待确认）

2. **功能差异**
   - CUA 版本可能在某些功能上有差异（如区域限制）
   - 需要确认是否支持所有 `gemini-2.5-pro` 的功能

3. **向后兼容**
   - 确保新配置不会影响现有 `gemini-2.5-pro` 的使用
   - 确保模型名称匹配逻辑的优先级正确

4. **错误处理**
   - 如果模型不可用，确保错误信息清晰

### 代码示例

#### 示例 1: 添加模型配置

```json
"gemini/gemini-2.5-pro-cua": {
    "max_tokens": 65535,
    "max_input_tokens": 1048576,
    "max_output_tokens": 65535,
    "max_images_per_prompt": 3000,
    "max_videos_per_prompt": 10,
    "max_video_length": 1,
    "max_audio_length_hours": 8.4,
    "max_audio_per_prompt": 1,
    "max_pdf_size_mb": 30,
    "input_cost_per_token": 1.25e-06,  // 待确认
    "input_cost_per_token_above_200k_tokens": 2.5e-06,  // 待确认
    "output_cost_per_token": 1e-05,  // 待确认
    "output_cost_per_token_above_200k_tokens": 1.5e-05,  // 待确认
    "litellm_provider": "gemini",
    "mode": "chat",
    "rpm": 2000,
    "tpm": 800000,
    "supports_system_messages": true,
    "supports_function_calling": true,
    "supports_vision": true,
    "supports_audio_input": true,
    "supports_video_input": true,
    "supports_pdf_input": true,
    "supports_response_schema": true,
    "supports_tool_choice": true,
    "supports_reasoning": true,
    "supported_endpoints": [
        "/v1/chat/completions",
        "/v1/completions"
    ],
    "supported_modalities": [
        "text",
        "image",
        "audio",
        "video"
    ],
    "supported_output_modalities": [
        "text"
    ],
    "source": "https://ai.google.dev/gemini-api/docs/models",
    "supports_web_search": true,
    "cache_read_input_token_cost": 3.125e-07,
    "supports_prompt_caching": true
}
```

#### 示例 2: 测试代码

```python
# tests/test_litellm/llms/gemini/test_gemini_2_5_pro_cua.py

def test_gemini_2_5_pro_cua_model_info():
    """Test that gemini-2.5-pro-cua model info can be retrieved"""
    model_info = litellm.get_model_info("gemini/gemini-2.5-pro-cua")
    assert model_info is not None
    assert model_info.key == "gemini/gemini-2.5-pro-cua"
    assert model_info.litellm_provider == "gemini"

def test_gemini_2_5_pro_cua_completion():
    """Test basic completion with gemini-2.5-pro-cua"""
    response = litellm.completion(
        model="gemini/gemini-2.5-pro-cua",
        messages=[{"role": "user", "content": "Say hello"}]
    )
    assert response is not None
    assert response.choices is not None
    assert len(response.choices) > 0
```

### 参考资源

1. **现有配置参考**
   - `gemini/gemini-2.5-pro` (行 ~7820)
   - `gemini-2.5-pro` (行 ~7733)

2. **相关代码文件**
   - `litellm/utils.py` - 模型信息获取逻辑
   - `litellm/llms/gemini/` - Gemini 提供者实现
   - `litellm/llms/gemini/common_utils.py` - Gemini 通用工具

3. **文档**
   - Google AI Studio: https://ai.google.dev/
   - Vertex AI: https://cloud.google.com/vertex-ai

## 下一步行动

1. 确认 `gemini-2.5-pro-cua` 的实际 API 规格和定价
2. 根据确认的信息更新配置
3. 实现并测试
