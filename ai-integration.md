# RP-Hub Toolkit · AI 集成方案

> 本文件描述 AI 在整合工具中的完整集成方式，供开发者参考。
> **核心原则**：所有 AI 调用走统一接口，结果先预览再确认，确认后实时更新 UI。

---

## 目录

1. [整体架构](#一整体架构)
2. [API 配置管理](#二api-配置管理)
3. [统一调用接口](#三统一调用接口)
4. [更新机制](#四更新机制)
5. [预设 Prompt 体系](#五预设-prompt-体系)
6. [各模块 AI 功能清单](#六各模块-ai-功能清单)
7. [HTML 嵌套安全处理](#七html-嵌套安全处理)

---

## 一、整体架构

```mermaid
graph TB
    subgraph 用户操作层
        A1[字段级···按钮<br/>快速操作]
        A2[底部AI对话区<br/>自由输入]
        A3[一键AI按钮<br/>预设功能]
    end

    subgraph 调用层
        B[app.llm()<br/>统一调用接口]
    end

    subgraph 处理层
        C1[解析返回数据<br/>JSON / 文本]
        C2[构建 diff 预览<br/>修改前 vs 修改后]
    end

    subgraph 确认层
        D{用户确认}
    end

    subgraph 更新层
        E1[apply 写入数据]
        E2[save 持久化]
        E3[render 刷新 UI]
    end

    A1 --> B
    A2 --> B
    A3 --> B
    B --> C1 --> C2 --> D
    D -->|接受| E1 --> E2 --> E3
    D -->|取消| F[不改变]

    style A1 fill:#1a3a1a,stroke:#3fb950,color:#e6edf3
    style A2 fill:#1a3a1a,stroke:#3fb950,color:#e6edf3
    style A3 fill:#1a3a1a,stroke:#3fb950,color:#e6edf3
    style B fill:#2d1b2d,stroke:#bc8cff,color:#e6edf3
    style C2 fill:#1b2d3a,stroke:#58a6ff,color:#e6edf3
    style D fill:#3a2d1b,stroke:#d29922,color:#e6edf3
    style E3 fill:#1a3a1a,stroke:#3fb950,color:#e6edf3
```

### 1.1 三条 AI 触发路径

| 路径 | 触发方式 | 适用场景 | 示例 |
|------|----------|----------|------|
| **字段级按钮** | 字段旁的 `[···]` 弹出菜单 | 快速操作某一段文本 | 润色描述、生成标签 |
| **一键 AI 按钮** | 模块内的预设功能按钮 | 标准化的批量操作 | 诊断世界书、检查完整性 |
| **底部 AI 对话区** | 底部输入框自由输入 | 复杂需求、多轮对话 | "帮我生成一个校园背景的世界书" |

### 1.2 所有路径都走同一套流程

```
1. 构造 prompt（预设或用户输入）
     ↓
2. 调用 app.llm() → fetch → 流式或非流式
     ↓
3. 解析返回数据（JSON 或纯文本）
     ↓
4. 构建 diff 预览（修改前 vs 修改后）
     ↓
5. 用户确认 or 取消
     ↓
6. 确认 → apply() → save() → render()
    取消 → 不做任何修改
```

---

## 二、API 配置管理

### 2.1 配置存储

存储在全局设置的 ⚙️ 弹窗中，数据存到 `localStorage`：

```javascript
// 默认配置
const defaultLLMConfig = {
  apiUrl: '',           // OpenAI 兼容 API 地址
  apiKey: '',           // API Key
  model: 'gpt-4o-mini', // 模型名
  stream: true,         // 是否流式输出
  temperature: 0.7,     // 默认温度
};

// 存储键
app.storage.set('llm-config', config);
```

### 2.2 配置弹窗 UI

```
┌─ 设置 ──────────────────────────────────┐
│                                          │
│  🤖 AI 模型配置                          │
│                                          │
│  API 地址: [https://api.openai.com/v1]   │
│  API Key:  [sk-****************]         │
│  模型:     [gpt-4o-mini ▼]              │
│  温度:     [0.7 ─────────────●──]       │
│  流式输出: [☑]                           │
│                                          │
│  [测试连接] [保存] [取消]                │
└──────────────────────────────────────────┘
```

**测试连接**：发送一条简单消息验证 API 可用性，结果显示为 toast。

---

## 三、统一调用接口

参考原工坊源码 `callLLM()` 实现：

```javascript
const app = {
  // ...其他方法

  /**
   * 统一 AI 调用接口
   * @param {string} systemPrompt - 系统提示词
   * @param {string} userMessage  - 用户消息
   * @param {object} [opts]       - 可选参数
   * @param {string} [opts.model] - 覆盖模型名
   * @param {number} [opts.temperature] - 覆盖温度
   * @param {boolean} [opts.stream] - 覆盖流式
   * @param {string} [opts.apiUrl]  - 覆盖 API 地址
   * @param {string} [opts.apiKey]  - 覆盖 API Key
   * @returns {Promise<string>} AI 返回的文本
   */
  async llm(systemPrompt, userMessage, opts = {}) {
    const config = app.storage.get('llm-config') || {};
    const url = (opts.apiUrl || config.apiUrl).replace(/\/+$/, '') + '/chat/completions';
    const model = opts.model || config.model || 'gpt-4o-mini';
    const stream = opts.stream !== undefined ? opts.stream : (config.stream !== false);
    const temperature = opts.temperature !== undefined ? opts.temperature : (config.temperature || 0.7);

    const body = {
      model,
      messages: [
        { role: 'system', content: systemPrompt },
        { role: 'user', content: userMessage }
      ],
      stream,
      temperature
    };

    try {
      const resp = await fetch(url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': 'Bearer ' + (opts.apiKey || config.apiKey)
        },
        body: JSON.stringify(body)
      });

      if (!resp.ok) {
        let errMsg = '';
        try { const d = await resp.json(); errMsg = d.error?.message || ''; }
        catch (e) { errMsg = await resp.text(); }
        throw new Error('API ' + resp.status + (errMsg ? ': ' + errMsg.substring(0, 80) : ''));
      }

      if (stream && resp.body) {
        let full = '';
        const reader = resp.body.getReader();
        const decoder = new TextDecoder();
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;
          const chunk = decoder.decode(value, { stream: true });
          const lines = chunk.split('\n').filter(l => l.startsWith('data: '));
          for (const line of lines) {
            const data = line.slice(6).trim();
            if (data === '[DONE]') break;
            try {
              const parsed = JSON.parse(data);
              const content = parsed.choices?.[0]?.delta?.content || '';
              full += content;
            } catch (e) { /* 忽略解析错误 */ }
          }
        }
        return full;
      } else {
        const data = await resp.json();
        return data.choices?.[0]?.message?.content || '';
      }
    } catch (e) {
      app.toast('AI 调用失败: ' + e.message, 'error');
      throw e;
    }
  }
};
```

### 3.1 流式回调解耦

当 `stream: true` 时，可以在 `opts` 中传入 `onChunk` 回调实现逐字更新：

```javascript
// 在对话区使用流式输出
app.llm(sysPrompt, userMsg, {
  stream: true,
  onChunk: (text) => {
    // 每次收到新内容时更新 UI
    document.getElementById('ai-response').textContent += text;
  }
});
```

---

## 四、更新机制

参考原工坊 `feRun()` → `feShowResult()` → `feApply()` 流程。

### 4.1 完整流程

```
用户点击"润色"按钮
      │
      ▼
1. feRun(field, 'polish')
   ├─ 构建 prompt（当前文本 + 操作指令）
   ├─ 调用 app.llm()
   ├─ 显示 loading toast
   └─ 获取 AI 结果
      │
      ▼
2. feShowResult(field, label, oldVal, newVal)
   ├─ 弹出 diff 预览弹窗
   ├─ 左栏：修改前（红色标注）
   ├─ 右栏：修改后（绿色标注）
   └─ 显示字数变化 + [接受] [取消] 按钮
      │
      ▼
3. 用户选择
      │
      ├─ 接受 → feApply(field, newVal)
      │         ├─ 数据写入当前对象
      │         ├─ saveChars() 持久化
      │         └─ render() 刷新 UI
      │
      └─ 取消 → 关闭弹窗，不做任何修改
```

### 4.2 diff 预览 UI 设计

```
┌─ AI 修改预览 ──────────────────────────────────┐
│                                                  │
│  润色描述                                        │
│                                                  │
│  ┌────── 修改前 ──────┬────── 修改后 ──────┐    │
│  │                     │                     │    │
│  │ (红色标注)           │ (绿色标注)           │    │
│  │ 原名：小满是一个     │ 小满是一位性格        │    │
│  │ 开朗的女孩...       │ 开朗的少女...        │    │
│  │                     │                     │    │
│  └─────────────────────┴─────────────────────┘    │
│                                                  │
│  120 字 → 108 字 (-12)                            │
│                                                  │
│              [接受]  [取消]                        │
└──────────────────────────────────────────────────┘
```

### 4.3 无 diff 的直接操作

对于不需要预览的操作（如"生成世界书条目"），可以直接写入：

```javascript
async function generateWIEntry() {
  const result = await app.llm(prompt, userMsg);
  const entry = JSON.parse(result); // AI 返回结构化数据
  worldbook.push(normWI(entry));
  save();
  render();
  app.toast('世界书条目已生成', 'success');
}
```

---

## 五、预设 Prompt 体系

### 5.1 Prompt 组织方式

每个模块的 AI 功能都有一组预设 prompt，用 `buildPrompt(type, context)` 函数统一构造：

```javascript
function buildPrompt(type, context) {
  const basePrompt = `你是 RP-Hub 角色卡制作助手。
  当前角色: ${context.name}
  当前角色描述: ${context.description}`;

  const prompts = {
    // ===== 角色卡 =====
    'polish_description': basePrompt + '\n润色以下描述，保持原意，使表达更清晰自然。\n\n' + context.value,
    'generate_tags': basePrompt + '\n根据角色信息生成标签，格式：#+关键词，空格分隔。\n\n' + context.info,

    // ===== 世界书 =====
    'diagnose_triggers': basePrompt + '\n检查以下世界书条目的触发词是否合理，是否存在宽泛词。\n\n' + context.value,
    'optimize_content': basePrompt + '\n优化以下世界书条目内容，使其表达更精准、约束力更强。\n\n' + context.value,
    'diagnose_completeness': basePrompt + '\n诊断当前世界书的完整性，检查是否有遗漏的关键模块。\n\n' + context.wiSummary,
    'generate_entry': basePrompt + '\n根据以下描述生成一条世界书条目，以 JSON 格式返回。\n\n' + context.userInput,

    // ===== UI 模板 =====
    'generate_ui_template': `你是 RP-Hub 的专业 UI 模板生成器。
    输出格式必须是 JSON：
    {
      "name": "模板名称",
      "htmlTemplate": "<style>...</style><section>...</section>",
      "initialVariableState": {},
      "variableSchema": {}
    }
    硬性规则：
    - htmlTemplate 必须是 HTML 片段，不包含 <!DOCTYPE>/<html>/<head>/<body>
    - 必须包含 <style> 和可见顶层容器
    - 变量用 {{english_snake_case}}
    - UI 内部交互用局部 JS，剧情操作用 triggerSlash
    \n\n用户需求：` + context.userInput,

    // ===== 正则 =====
    'generate_regex': '根据以下示例文本和需求生成正则表达式，返回 JSON：\n{ "regex": "...", "flags": "...", "replacement": "..." }\n\n' + context.userInput,

    // ===== 验收 =====
    'analyze_card': '对以下角色卡进行全面评估，从脑洞创意、世界书质量、UI设计三个维度评分。\n\n' + context.cardJSON,

    // ===== 生图提示词 =====
    'optimize_prompt': '优化以下提示词，使其更符合 NAI 格式。\n\n' + context.value,
  };

  return prompts[type] || basePrompt;
}
```

### 5.2 各模块 Prompt 对照表

| 模块 | prompt type | 输入 | 输出格式 |
|------|-------------|------|----------|
| 角色卡 | `polish_description` | 当前描述文本 | 文本 |
| 角色卡 | `expand_description` | 当前描述文本 | 文本 |
| 角色卡 | `generate_tags` | 角色信息 | 文本（#标签格式） |
| 角色卡 | `generate_character` | 用户关键词 | JSON |
| 世界书 | `diagnose_triggers` | 世界书条目 | 文本（诊断报告） |
| 世界书 | `optimize_content` | 某条目内容 | 文本 |
| 世界书 | `diagnose_completeness` | 世界书摘要 | 文本（诊断报告） |
| 世界书 | `generate_entry` | 用户描述 | JSON（世界书条目） |
| UI模板 | `generate_ui_template` | 用户需求 | JSON |
| UI模板 | `transform_ui_template` | 已有HTML | JSON |
| 正则 | `generate_regex` | 示例文本+需求 | JSON |
| 验收 | `analyze_card` | 完整角色卡JSON | JSON（评分报告） |
| 生图 | `optimize_prompt` | 提示词文本 | 文本 |

---

## 六、各模块 AI 功能清单

### 6.1 角色卡编辑器

**字段级按钮**（每个字段旁的 `[···]` 菜单）：

| 字段 | 可用操作 | prompt type |
|------|----------|-------------|
| 描述 | 润色、扩写 | `polish_description`, `expand_description` |
| 性格 | 润色、扩写 | `polish_description`, `expand_description` |
| 场景 | 润色、生成 | `polish_description`, `expand_description` |
| 标签 | 生成标签 | `generate_tags` |
| 所有字段 | 翻译（中→英/英→中） | 通用翻译 prompt |

**一键 AI 按钮**（页面底部工具栏）：

| 按钮 | 说明 | prompt type |
|------|------|-------------|
| 生成角色设定 | 从关键词生成完整角色卡 | `generate_character` |
| 优化描述 | 批量润色所有文本字段 | 循环调用 `polish_description` |
| 检查完整性 | 检查缺失字段 | 通用检查 prompt |

### 6.2 世界书管理器

**字段级按钮**（条目编辑面板内）：

| 字段 | 可用操作 | prompt type |
|------|----------|-------------|
| 内容 | 润色、扩写、压缩 | `optimize_content` |
| 触发词 | 建议补充 | `diagnose_triggers` |

**一键 AI 按钮**（条目编辑标签页顶部）：

| 按钮 | 说明 | prompt type |
|------|------|-------------|
| 诊断触发词 | 检查所有条目的触发词是否合理 | `diagnose_triggers` |
| 优化内容 | 批量优化所有条目的内容表述 | `optimize_content` |
| 诊断完整性 | 检查世界书是否覆盖了必要模块 | `diagnose_completeness` |
| 生成条目 | 根据用户描述生成新条目 | `generate_entry` |

**对话区 AI 用法**（底部输入框 + [🤖AI] 标签）：

```
用户输入示例：
  "帮我生成一个校园背景的世界书，包含教室、图书馆、运动场三个场景"
  → AI 返回 JSON 格式的条目列表
  → 用户确认后一次性追加到世界书
```

### 6.3 体检验收台

| 按钮 | 说明 | prompt type |
|------|------|-------------|
| AI 评分 | 调用 AI 对导入的角色卡综合评分 | `analyze_card` |
| 生成报告 | AI 生成详细的验收报告 | `analyze_card` |

### 6.4 UI 模板实验室

**一键 AI 按钮**：

| 按钮 | 说明 | prompt type |
|------|------|-------------|
| 从描述生成 | 输入需求，AI 生成完整 UI 模板 | `generate_ui_template` |
| 改造现有模板 | 改造当前编辑中的模板 | `transform_ui_template` |
| 分析变量 | AI 分析模板中的变量并更新 schema | 通用分析 prompt |

### 6.5 正则工坊

**一键 AI 按钮**：

| 按钮 | 说明 | prompt type |
|------|------|-------------|
| 从文本生成 | 输入示例文本，AI 生成正则 | `generate_regex` |
| 优化正则 | 优化当前正则表达式 | 通用优化 prompt |

### 6.6 生图提示词管理

| 功能 | 说明 |
|------|------|
| 对话生成 | 底部对话区自由对话生成提示词 |
| 优化提示词 | AI 优化当前选中的提示词 |
| 建议补充 | AI 建议添加缺失的提示词元素 |

---

## 七、HTML 嵌套安全处理

### 7.1 问题定义

UI 模板的内容即为 HTML 代码，需要在工具中编辑和预览。如果直接把 HTML 插入到页面 DOM 中，会导致：
1. 样式冲突（模板 CSS 影响工具页面）
2. DOM 结构错乱（内部 div 破坏布局）
3. XSS 风险（模板中的 JS 可能恶意执行）

### 7.2 解决方案

**编辑时用 `<textarea>`（纯文本，不渲染）：**

```html
<!-- 编辑 UI 模板 HTML -->
<textarea id="ui-editor" class="textarea mono"
  placeholder="粘贴或编辑 HTML 模板代码..."
  style="height:300px;font-family:monospace;font-size:12px">
</textarea>
```

**预览时用 `<iframe srcdoc="...">`（完全隔离）：**

```javascript
function renderUIPreview(htmlContent) {
  const iframe = document.getElementById('ui-preview');
  const sandboxHtml = `<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <style>
    /* 这里可以加一些基础的 reset 样式 */
    body { margin: 0; padding: 0; }
  </style>
</head>
<body>
  ${htmlContent}
</body>
</html>`;

  iframe.srcdoc = sandboxHtml;
}
```

```html
<!-- UI 模板预览区 -->
<iframe id="ui-preview"
  sandbox="allow-scripts"
  style="width:100%;height:400px;border:1px solid var(--bd);border-radius:var(--r-sm)">
</iframe>
```

### 7.3 安全措施

| 措施 | 说明 |
|------|------|
| `sandbox="allow-scripts"` | 允许执行 JS，但禁止访问父页面 |
| `srcdoc` | 直接注入 HTML 内容，不走 URL 加载 |
| `textarea` 编辑 | 纯文本编辑，不会被浏览器渲染 |
| 导出时校验 | 导出 UI 模板时做基本的安全过滤 |
| 不允许 `<!DOCTYPE>` / `<html>` / `<head>` / `<body>` | 模板内只允许片段，防止完整的页面嵌入 |

### 7.4 模板存储

```javascript
// 存储到角色卡 extensions 中
{
  htmlTemplate: '<style>.my-panel { color: red; }</style><div class="my-panel">{{content}}</div>',
  initialVariableState: { content: '默认内容' },
  variableState: { content: '当前内容' },
  variableSchema: { content: '文本内容变量' }
}
```

---

## 八、对话区（底部常驻 + [🤖AI] 标签展开）

### 8.1 布局

```
┌──────────────────────────────────────────────────────────┐
│  模块内标签栏                                              │
│  [📝条目编辑] [📊统计概览] [🤖AI对话]                    │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  当 [🤖AI对话] 未激活时:                                  │
│  ┌─ 条目列表 + 编辑面板 ─────────────────────────────┐   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                           │
│  当 [🤖AI对话] 激活时:                                    │
│  ┌─ AI 对话记录 ─────────────────────────────────────┐   │
│  │  [用户] 帮我生成一个校园背景的世界书               │   │
│  │  [AI]   好的，以下是为您生成的世界书条目...        │   │
│  │  [AI]   (JSON 格式条目列表)                       │   │
│  │  [确认追加] [取消]                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  输入你的问题或需求...                     [发送]    │  │ ← 始终显示
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### 8.2 交互

| 状态 | 底部输入条 | 内容区 |
|------|-----------|--------|
| 未激活 AI 对话 | ✅ 可见 | 显示当前模块的主内容 |
| 激活 AI 对话 | ✅ 可见 | 显示对话记录 |
| AI 生成中 | ✅ 可见 + 停止按钮 | 流式显示生成内容 |

### 8.3 对话数据结构

```javascript
const chatHistory = [
  { role: 'system', content: '你是 RP-Hub 角色卡制作助手...' },
  { role: 'user', content: '帮我生成一个校园世界书' },
  { role: 'assistant', content: '{\n  "entries": [\n    {...}\n  ]\n}' },
];

// AI 返回 JSON 时，解析后在消息下方显示"确认追加"按钮
```

---

## 九、错误处理

| 错误类型 | 表现 | 处理方式 |
|----------|------|----------|
| 网络断开 | toast "网络连接失败" | 提示检查网络 |
| API 地址错误 | toast "API 响应 404" | 提示检查 API 配置 |
| API Key 无效 | toast "API 响应 401：认证失败" | 提示检查 API Key |
| 模型返回格式错误 | toast "AI 返回格式异常" | 显示原始返回文本 |
| 超时 | toast "请求超时，请重试" | 自动重试一次 |
| 流中断 | toast "连接中断，已获取部分内容" | 保留已获取的内容 |

---

## 更新日志

| 日期 | 版本 | 变更说明 |
|------|:----:|----------|
| 2026-06-26 | v0.1 | 初始 AI 集成方案 |
