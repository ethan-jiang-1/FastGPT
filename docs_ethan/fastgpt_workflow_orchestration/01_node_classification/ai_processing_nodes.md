# FastGPT AI处理节点深度分析

## 🤖 AI处理节点概述

AI处理节点是 FastGPT 工作流编排系统的**智能核心**，负责所有涉及人工智能模型调用、智能分析、内容理解和生成的处理任务。这些节点封装了复杂的AI能力，通过简洁的接口为用户提供强大的智能处理功能。

### 核心设计理念

- **🧠 智能驱动** - 以AI模型为核心的智能处理能力
- **🔧 模块化设计** - 每个节点专注特定的AI处理任务
- **⚡ 流式响应** - 支持实时流式输出和交互
- **🎯 任务专精** - 针对不同AI任务优化的专用节点
- **🔒 模型无关** - 支持多种LLM提供商和模型类型
- **📊 结构化输出** - 标准化的数据输入输出格式

## 📊 AI处理节点分类

### 按AI任务类型分类

| 分类 | 节点类型 | 主要功能 | 典型场景 |
|------|----------|----------|----------|
| **对话生成** | `chatNode` | AI对话和文本生成 | 聊天机器人、内容创作 |
| **智能代理** | `agent` | 工具调用和多步推理 | 复杂任务自动化、API调用 |
| **内容分类** | `classifyQuestion` | 问题意图分类 | 智能路由、内容标签 |
| **信息抽取** | `contentExtract` | 结构化信息提取 | 数据解析、表单填充 |
| **语义搜索** | `datasetSearchNode` | 向量语义检索 | 知识库问答、相似性匹配 |
| **查询增强** | `queryExtension` | 查询扩展优化 | 搜索优化、语义理解 |
| **结果聚合** | `datasetConcatNode` | 数据集结果合并 | 多源信息整合 |

### 按模型能力分类

| 能力类别 | 适用节点 | 模型要求 | 性能特点 |
|----------|----------|----------|----------|
| **文本生成** | `chatNode`, `agent` | 大语言模型 | 创造性、流畅性 |
| **文本理解** | `classifyQuestion`, `contentExtract` | 理解型模型 | 准确性、结构化 |
| **向量计算** | `datasetSearchNode`, `queryExtension` | 嵌入模型 | 语义相似性 |
| **多模态** | `chatNode` (vision) | 视觉语言模型 | 图文理解 |

## 🚀 核心AI处理节点详解

### 1. AI聊天节点 (chatNode)

**节点标识**: `FlowNodeTypeEnum.chatNode`  
**模板分类**: `FlowNodeTemplateTypeEnum.ai`  
**核心功能**: 基于大语言模型的对话生成和文本处理

#### 输入接口定义

```typescript
interface ChatNodeInputs {
  // 模型选择
  model: {
    key: 'model'
    renderTypeList: [FlowNodeInputTypeEnum.selectLLMModel, FlowNodeInputTypeEnum.reference]
    llmModelType: LLMModelTypeEnum.chat
    label: 'AI模型'
    required: true
    toolDescription: '选择对话AI模型'
  }
  
  // 系统提示词
  systemPrompt: {
    key: 'systemPrompt'
    renderTypeList: [FlowNodeInputTypeEnum.textarea, FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.string
    label: '系统提示词'
    max: 3000
    placeholder: '模型固定的引导词，通过调整该内容，可以引导模型聊天方向。'
    toolDescription: '模型固定的引导词'
  }
  
  // 对话历史
  history: {
    key: 'history'
    renderTypeList: [FlowNodeInputTypeEnum.numberInput, FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.chatHistory
    label: '聊天记录'
    required: true
    min: 0
    max: 30
    value: 6
    toolDescription: '最多携带多少轮对话记录'
  }
  
  // 用户问题
  userChatInput: {
    key: 'userChatInput'
    renderTypeList: [FlowNodeInputTypeEnum.reference, FlowNodeInputTypeEnum.textarea]
    valueType: WorkflowIOValueTypeEnum.string
    label: '用户问题'
    required: true
    toolDescription: '用户问题'
  }
  
  // 知识库引用
  quoteQA: {
    key: 'quoteQA'
    renderTypeList: [FlowNodeInputTypeEnum.settingDatasetQuotePrompt]
    label: '知识库引用'
    debugLabel: '知识库引用'
    description: '来自知识库搜索的引用内容'
    valueType: WorkflowIOValueTypeEnum.datasetQuote
  }
  
  // 温度参数
  temperature: {
    key: 'temperature'
    renderTypeList: [FlowNodeInputTypeEnum.hidden]
    label: '温度'
    value: 0
    valueType: WorkflowIOValueTypeEnum.number
    min: 0
    max: 10
    step: 1
  }
  
  // 最大令牌数
  maxToken: {
    key: 'maxToken'
    renderTypeList: [FlowNodeInputTypeEnum.hidden]
    label: '回复上限'
    value: 4000
    valueType: WorkflowIOValueTypeEnum.number
    min: 100
    max: 4000
    step: 50
  }
  
  // 文件输入
  files: {
    key: 'files'
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.arrayString
    label: '文件'
    toolDescription: '待处理的文件列表'
  }
}
```

#### 输出接口定义

```typescript
interface ChatNodeOutputs {
  // 回答文本
  answerText: {
    id: 'answerText'
    key: 'answerText'
    label: 'AI回复'
    description: 'AI的回复/总结'
    valueType: WorkflowIOValueTypeEnum.string
    type: FlowNodeOutputTypeEnum.static
  }
  
  // 推理过程
  reasoning: {
    id: 'reasoning'
    key: 'reasoning'
    label: '推理过程'
    description: 'AI推理过程，需要模型支持'
    valueType: WorkflowIOValueTypeEnum.string
    type: FlowNodeOutputTypeEnum.static
  }
  
  // 更新的历史记录
  history: {
    id: 'history'
    key: 'history'
    label: '新的上下文'
    description: '将本次回复内容拼接上历史记录，作为新的上下文返回'
    valueType: WorkflowIOValueTypeEnum.chatHistory
    type: FlowNodeOutputTypeEnum.static
  }
  
  // 使用的模型
  model: {
    id: 'model'
    key: 'model'
    label: '模型名'
    description: '本次对话使用的模型名称'
    valueType: WorkflowIOValueTypeEnum.string
    type: FlowNodeOutputTypeEnum.static
  }
}
```

#### 高级特性配置

```typescript
interface ChatNodeAdvancedConfig {
  // 视觉能力
  vision: boolean          // 是否支持图像理解
  reasoning: boolean       // 是否启用推理模式
  functionCall: boolean    // 是否支持函数调用
  
  // 响应控制
  stream: boolean          // 是否流式响应
  stopWords: string[]      // 停止词列表
  
  // 安全控制
  contentFilter: boolean   // 内容过滤
  maxRetries: number      // 最大重试次数
}
```

### 2. AI智能代理节点 (agent)

**节点标识**: `FlowNodeTypeEnum.agent`  
**模板分类**: `FlowNodeTemplateTypeEnum.ai`  
**核心功能**: 具备工具调用能力的智能代理，可执行复杂的多步骤任务

#### 输入接口定义

```typescript
interface AgentNodeInputs {
  // 模型配置
  model: {
    key: 'model'
    renderTypeList: [FlowNodeInputTypeEnum.selectLLMModel]
    llmModelType: LLMModelTypeEnum.chat
    label: 'AI模型'
    required: true
  }
  
  // 系统提示词
  systemPrompt: {
    key: 'systemPrompt'
    renderTypeList: [FlowNodeInputTypeEnum.textarea, FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.string
    label: '系统提示词'
    max: 3000
    placeholder: '你是一个有用的AI助手...'
  }
  
  // 工具调用策略
  toolChoice: {
    key: 'toolChoice'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: '工具调用策略'
    list: [
      { label: '自动选择', value: 'auto' },
      { label: '必须调用', value: 'required' },
      { label: '不调用工具', value: 'none' }
    ]
    value: 'auto'
  }
  
  // 用户输入
  userChatInput: {
    key: 'userChatInput'
    renderTypeList: [FlowNodeInputTypeEnum.reference, FlowNodeInputTypeEnum.textarea]
    valueType: WorkflowIOValueTypeEnum.string
    label: '用户问题'
    required: true
    toolDescription: '需要代理处理的任务或问题'
  }
  
  // 可调用工具
  tools: {
    key: 'tools'
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.arrayString
    label: '可调用工具'
    description: '代理可以调用的工具列表'
  }
  
  // 历史记录
  history: {
    key: 'history'
    renderTypeList: [FlowNodeInputTypeEnum.numberInput, FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.chatHistory
    label: '对话历史'
    min: 0
    max: 30
    value: 6
  }
  
  // 文件输入
  files: {
    key: 'files'
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.arrayString
    label: '文件'
    toolDescription: '需要处理的文件'
  }
}
```

#### 输出接口定义

```typescript
interface AgentNodeOutputs {
  // 代理回复
  agentResponse: {
    id: 'agentResponse'
    key: 'agentResponse'
    label: '代理回复'
    description: '智能代理的最终回复内容'
    valueType: WorkflowIOValueTypeEnum.string
    type: FlowNodeOutputTypeEnum.static
  }
  
  // 工具调用结果
  toolCallResults: {
    id: 'toolCallResults'
    key: 'toolCallResults'
    label: '工具调用结果'
    description: '所有工具调用的结果集合'
    valueType: WorkflowIOValueTypeEnum.arrayObject
    type: FlowNodeOutputTypeEnum.static
  }
  
  // 思考过程
  reasoning: {
    id: 'reasoning'
    key: 'reasoning'
    label: '思考过程'
    description: '代理的思考和决策过程'
    valueType: WorkflowIOValueTypeEnum.string
    type: FlowNodeOutputTypeEnum.static
  }
  
  // 交互响应(如需用户确认)
  interactive: {
    id: 'interactive'
    key: 'interactive'
    label: '交互响应'
    description: '需要用户交互确认的内容'
    valueType: WorkflowIOValueTypeEnum.object
    type: FlowNodeOutputTypeEnum.static
  }
}
```

#### 工具调用机制

```typescript
interface ToolCallConfig {
  // 调用策略
  strategy: 'toolChoice' | 'functionCall' | 'promptCall'
  
  // 工具选择模式
  toolChoice: {
    type: 'auto' | 'required' | 'none' | 'specific'
    toolName?: string  // 当type为specific时指定工具
  }
  
  // 并行调用配置
  parallelToolCalls: boolean
  
  // 最大调用次数
  maxToolCalls: number
  
  // 工具调用超时
  toolCallTimeout: number
}
```

### 3. 问题分类节点 (classifyQuestion)

**节点标识**: `FlowNodeTypeEnum.classifyQuestion`  
**模板分类**: `FlowNodeTemplateTypeEnum.ai`  
**核心功能**: 基于AI模型的智能问题分类和意图识别

#### 输入接口定义

```typescript
interface ClassifyQuestionInputs {
  // 模型选择
  model: {
    key: 'model'
    renderTypeList: [FlowNodeInputTypeEnum.selectLLMModel]
    llmModelType: LLMModelTypeEnum.classify
    label: '分类模型'
    required: true
  }
  
  // 系统提示
  systemPrompt: {
    key: 'systemPrompt'
    renderTypeList: [FlowNodeInputTypeEnum.textarea, FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.string
    label: '背景知识'
    description: '分类任务的背景信息，帮助模型更好地理解分类标准'
    placeholder: '请根据用户的问题，进行意图分类...'
  }
  
  // 分类代理配置
  agents: {
    key: 'agents'
    renderTypeList: [FlowNodeInputTypeEnum.custom]
    valueType: WorkflowIOValueTypeEnum.any
    label: '分类代理'
    description: '配置分类的选项和对应的处理逻辑'
    value: [] // Array<{value: string, key: string, description: string}>
  }
  
  // 用户输入
  userChatInput: {
    key: 'userChatInput'  
    renderTypeList: [FlowNodeInputTypeEnum.reference, FlowNodeInputTypeEnum.textarea]
    valueType: WorkflowIOValueTypeEnum.string
    label: '用户问题'
    required: true
    toolDescription: '需要进行分类的用户问题'
  }
  
  // 历史记录
  history: {
    key: 'history'
    renderTypeList: [FlowNodeInputTypeEnum.numberInput, FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.chatHistory
    label: '聊天记录'
    min: 0
    max: 6
    value: 6
  }
}
```

#### 输出接口定义

```typescript
interface ClassifyQuestionOutputs {
  // 分类结果
  cqResult: {
    id: 'cqResult'
    key: 'cqResult'
    label: '分类结果'
    description: '问题分类的结果'
    valueType: WorkflowIOValueTypeEnum.string
    type: FlowNodeOutputTypeEnum.static
  }
  
  // 分类置信度
  confidence: {
    id: 'confidence'
    key: 'confidence'
    label: '置信度'
    description: '分类预测的置信度分数'
    valueType: WorkflowIOValueTypeEnum.number
    type: FlowNodeOutputTypeEnum.static
  }
  
  // 分类详情
  classifyDetail: {
    id: 'classifyDetail'
    key: 'classifyDetail'
    label: '分类详情'
    description: '详细的分类信息和理由'
    valueType: WorkflowIOValueTypeEnum.object
    type: FlowNodeOutputTypeEnum.static
  }
}
```

#### 分类代理配置示例

```typescript
interface ClassifyAgent {
  value: string      // 分类标签
  key: string        // 分类键值
  description: string // 分类描述
  examples?: string[] // 示例问题
  keywords?: string[] // 关键词匹配
  priority?: number   // 优先级
}

// 配置示例
const classifyAgents: ClassifyAgent[] = [
  {
    value: '技术支持',
    key: 'tech_support', 
    description: '用户遇到技术问题需要帮助解决',
    examples: ['登录不了', '功能报错', '系统崩溃'],
    keywords: ['错误', '故障', '问题', 'bug']
  },
  {
    value: '产品咨询',
    key: 'product_inquiry',
    description: '用户询问产品功能、价格、使用方法等信息',
    examples: ['这个功能怎么用', '多少钱', '有什么特点'],
    keywords: ['价格', '功能', '特性', '使用']
  },
  {
    value: '投诉建议',
    key: 'feedback',
    description: '用户提出投诉或改进建议',
    examples: ['服务态度不好', '希望增加某功能', '体验很差'],
    keywords: ['投诉', '建议', '改进', '不满意']
  }
]
```

### 4. 内容提取节点 (contentExtract)

**节点标识**: `FlowNodeTypeEnum.contentExtract`  
**模板分类**: `FlowNodeTemplateTypeEnum.ai`  
**核心功能**: 从非结构化文本中提取结构化信息

#### 输入接口定义

```typescript
interface ContentExtractInputs {
  // 模型选择
  model: {
    key: 'model'
    renderTypeList: [FlowNodeInputTypeEnum.selectLLMModel]
    llmModelType: LLMModelTypeEnum.extractFields
    label: '提取模型'
    required: true
  }
  
  // 提取描述
  description: {
    key: 'description'
    renderTypeList: [FlowNodeInputTypeEnum.textarea, FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.string
    label: '提取要求描述'
    description: '描述需要提取的内容和格式要求'
    placeholder: '你需要从文本中提取...'
  }
  
  // 历史记录
  history: {
    key: 'history'
    renderTypeList: [FlowNodeInputTypeEnum.numberInput, FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.chatHistory
    label: '聊天记录'
    min: 0
    max: 6
    value: 6
  }
  
  // 待提取内容
  contextExtractInput: {
    key: 'contextExtractInput'
    renderTypeList: [FlowNodeInputTypeEnum.reference, FlowNodeInputTypeEnum.textarea]
    label: '待提取的文本'
    required: true
    valueType: WorkflowIOValueTypeEnum.string
    toolDescription: '需要进行内容提取的文本'
  }
  
  // 提取字段配置
  extractKeys: {
    key: 'extractKeys'
    renderTypeList: [FlowNodeInputTypeEnum.custom]
    label: '目标字段'
    valueType: WorkflowIOValueTypeEnum.any
    description: '定义需要提取的字段结构'
    value: [] // ExtractFieldConfig[]
  }
}
```

#### 输出接口定义

```typescript
interface ContentExtractOutputs {
  // 提取成功标志
  success: {
    id: 'success'
    key: 'success'
    label: '提取成功'
    required: true
    description: '是否成功提取所有字段'
    valueType: WorkflowIOValueTypeEnum.boolean
    type: FlowNodeOutputTypeEnum.static
  }
  
  // 提取结果
  contextExtractFields: {
    id: 'contextExtractFields'
    key: 'contextExtractFields'
    label: '完整提取结果'
    required: true
    description: '提取的结构化数据'
    valueType: WorkflowIOValueTypeEnum.string
    type: FlowNodeOutputTypeEnum.static
  }
  
  // 动态字段输出 (根据extractKeys配置动态生成)
  // 例如: name, age, email 等字段
}
```

#### 字段提取配置

```typescript
interface ExtractFieldConfig {
  valueType: WorkflowIOValueTypeEnum // 字段数据类型
  desc: string                       // 字段描述
  key: string                        // 字段键名
  required: boolean                  // 是否必填
  enum?: string[]                    // 枚举值选项
  defaultValue?: any                 // 默认值
  
  // 高级配置
  validation?: {
    pattern?: string     // 正则验证
    minLength?: number   // 最小长度
    maxLength?: number   // 最大长度
    min?: number         // 最小值
    max?: number         // 最大值
  }
}

// 配置示例：提取联系人信息
const extractConfig: ExtractFieldConfig[] = [
  {
    key: 'name',
    valueType: WorkflowIOValueTypeEnum.string,
    desc: '联系人姓名',
    required: true,
    validation: { minLength: 1, maxLength: 50 }
  },
  {
    key: 'phone',
    valueType: WorkflowIOValueTypeEnum.string,
    desc: '电话号码',
    required: false,
    validation: { pattern: '^1[3-9]\\d{9}$' }
  },
  {
    key: 'email',
    valueType: WorkflowIOValueTypeEnum.string,
    desc: '邮箱地址',
    required: false,
    validation: { pattern: '^[\\w.-]+@[\\w.-]+\\.[a-zA-Z]{2,}$' }
  },
  {
    key: 'category',
    valueType: WorkflowIOValueTypeEnum.string,
    desc: '联系人类别',
    required: true,
    enum: ['朋友', '同事', '客户', '家人']
  }
]
```

### 5. 知识库搜索节点 (datasetSearchNode)

**节点标识**: `FlowNodeTypeEnum.datasetSearchNode`  
**模板分类**: `FlowNodeTemplateTypeEnum.ai`  
**核心功能**: 基于向量嵌入的语义搜索和知识检索

#### 输入接口定义

```typescript
interface DatasetSearchInputs {
  // 数据集选择
  datasets: {
    key: 'datasets'
    renderTypeList: [FlowNodeInputTypeEnum.selectDataset]
    label: '关联的知识库'
    value: []
    valueType: WorkflowIOValueTypeEnum.selectDataset
    required: true
  }
  
  // 搜索模式
  searchMode: {
    key: 'searchMode'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: '搜索模式'
    value: 'embedding'
    list: [
      { label: '语义检索', value: 'embedding' },
      { label: '混合检索', value: 'mixedRecall' }
    ]
  }
  
  // 相似度阈值
  similarity: {
    key: 'similarity'
    renderTypeList: [FlowNodeInputTypeEnum.selectDatasetParamsModal]
    label: '相似度'
    valueType: WorkflowIOValueTypeEnum.number
    showTargetInApp: false
    showTargetInPlugin: false
    value: 0.4
    min: 0
    max: 1
    step: 0.01
  }
  
  // 搜索条数
  limit: {
    key: 'limit'
    renderTypeList: [FlowNodeInputTypeEnum.selectDatasetParamsModal]
    label: '搜索条数'
    description: '最多取多少条记录'
    valueType: WorkflowIOValueTypeEnum.number
    showTargetInApp: false
    showTargetInPlugin: false
    value: 5
    min: 1
    max: 20
  }
  
  // 搜索内容
  userChatInput: {
    key: 'userChatInput'
    renderTypeList: [FlowNodeInputTypeEnum.reference, FlowNodeInputTypeEnum.textarea]
    valueType: WorkflowIOValueTypeEnum.string
    label: '用户问题'
    required: true
    toolDescription: '需要检索的问题'
  }
  
  // 重排序配置
  reRankQuery: {
    key: 'reRankQuery'
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.string
    label: '重排序查询词'
    description: '空则以"用户问题"作为查询词'
    canEdit: true
    editField: { key: 'reRankQuery' }
  }
  
  // 查询扩展
  queryExtension: {
    key: 'queryExtension'
    renderTypeList: [FlowNodeInputTypeEnum.switch]
    label: '查询扩展'
    valueType: WorkflowIOValueTypeEnum.boolean
    description: '是否对查询进行扩展以提高召回率'
    value: false
  }
  
  // 数据集过滤
  datasetFilters: {
    key: 'datasetFilters'
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.any
    label: '数据集过滤'
    description: '根据标签等条件过滤数据集'
  }
}
```

#### 输出接口定义

```typescript
interface DatasetSearchOutputs {
  // 引用内容
  quoteQA: {
    id: 'quoteQA'
    key: 'quoteQA'
    label: '引用内容'
    description: '搜索结果的引用内容'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.datasetQuote
  }
  
  // 搜索结果
  searchResults: {
    id: 'searchResults'
    key: 'searchResults'
    label: '搜索结果'
    description: '详细的搜索结果列表'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.arrayObject
  }
  
  // 相似度分数
  similarity: {
    id: 'similarity'
    key: 'similarity'
    label: '相似度'
    description: '最高匹配结果的相似度分数'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.number
  }
  
  // 匹配数量
  matchedCount: {
    id: 'matchedCount'
    key: 'matchedCount'
    label: '匹配数量'
    description: '符合阈值的搜索结果数量'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.number
  }
}
```

#### 搜索模式详解

```typescript
interface SearchModeConfig {
  // 嵌入搜索
  embedding: {
    model: string           // 嵌入模型
    topK: number           // 初步召回数量
    similarity: number     // 相似度阈值
    rerank?: {
      model: string        // 重排序模型
      topK: number        // 重排序后保留数量
    }
  }
  
  // 混合召回
  mixedRecall: {
    embedding: {
      weight: number       // 向量搜索权重
      topK: number
    }
    fullText: {
      weight: number       // 全文搜索权重  
      topK: number
    }
    rerank?: {
      model: string
      topK: number
    }
  }
}
```

### 6. 查询扩展节点 (queryExtension)

**节点标识**: `FlowNodeTypeEnum.queryExtension`  
**模板分类**: `FlowNodeTemplateTypeEnum.ai`  
**核心功能**: 智能查询扩展和改写，提高搜索召回率

#### 输入接口定义

```typescript
interface QueryExtensionInputs {
  // 原始查询
  query: {
    key: 'query'
    renderTypeList: [FlowNodeInputTypeEnum.reference, FlowNodeInputTypeEnum.textarea]
    valueType: WorkflowIOValueTypeEnum.string
    label: '原始查询'
    required: true
    toolDescription: '需要扩展的查询内容'
  }
  
  // 扩展模式
  extensionMode: {
    key: 'extensionMode'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: '扩展模式'
    value: 'auto'
    list: [
      { label: '自动扩展', value: 'auto' },
      { label: '同义词扩展', value: 'synonym' },
      { label: '上下文扩展', value: 'context' },
      { label: '意图扩展', value: 'intent' }
    ]
  }
  
  // 扩展数量
  maxExtensions: {
    key: 'maxExtensions'
    renderTypeList: [FlowNodeInputTypeEnum.numberInput]
    valueType: WorkflowIOValueTypeEnum.number
    label: '最大扩展数'
    value: 3
    min: 1
    max: 10
  }
  
  // 领域知识
  domainKnowledge: {
    key: 'domainKnowledge'
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.string
    label: '领域知识'
    description: '相关领域的背景知识，用于指导扩展'
  }
}
```

#### 输出接口定义

```typescript
interface QueryExtensionOutputs {
  // 扩展查询列表
  extendedQueries: {
    id: 'extendedQueries'
    key: 'extendedQueries'
    label: '扩展查询'
    description: '生成的扩展查询列表'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.arrayString
  }
  
  // 最佳扩展查询
  bestQuery: {
    id: 'bestQuery'
    key: 'bestQuery'
    label: '最佳查询'
    description: 'AI推荐的最佳扩展查询'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.string
  }
  
  // 扩展详情
  extensionDetails: {
    id: 'extensionDetails'
    key: 'extensionDetails'
    label: '扩展详情'
    description: '查询扩展的详细信息和理由'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.object
  }
}
```

### 7. 数据集拼接节点 (datasetConcatNode)

**节点标识**: `FlowNodeTypeEnum.datasetConcatNode`  
**模板分类**: `FlowNodeTemplateTypeEnum.ai`  
**核心功能**: 合并多个数据集搜索结果，进行智能去重和排序

#### 输入接口定义

```typescript
interface DatasetConcatInputs {
  // 数据集1
  quoteQA1: {
    key: 'quoteQA1'
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.datasetQuote
    label: '数据集1'
    required: true
  }
  
  // 数据集2  
  quoteQA2: {
    key: 'quoteQA2'
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.datasetQuote
    label: '数据集2'
    required: true
  }
  
  // 合并策略
  mergeStrategy: {
    key: 'mergeStrategy'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: '合并策略'
    value: 'score'
    list: [
      { label: '按相似度排序', value: 'score' },
      { label: '按时间排序', value: 'time' },
      { label: '轮询合并', value: 'round_robin' }
    ]
  }
  
  // 去重阈值
  duplicateThreshold: {
    key: 'duplicateThreshold'
    renderTypeList: [FlowNodeInputTypeEnum.numberInput]
    valueType: WorkflowIOValueTypeEnum.number
    label: '去重阈值'
    description: '内容相似度超过此阈值将被视为重复'
    value: 0.8
    min: 0.5
    max: 1.0
    step: 0.05
  }
  
  // 最大输出数量
  maxOutputs: {
    key: 'maxOutputs'
    renderTypeList: [FlowNodeInputTypeEnum.numberInput]
    valueType: WorkflowIOValueTypeEnum.number
    label: '最大输出数'
    value: 10
    min: 1
    max: 50
  }
}
```

#### 输出接口定义

```typescript
interface DatasetConcatOutputs {
  // 合并结果
  mergedQuoteQA: {
    id: 'mergedQuoteQA'
    key: 'mergedQuoteQA'
    label: '合并结果'
    description: '合并后的数据集引用内容'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.datasetQuote
  }
  
  // 合并统计
  mergeStats: {
    id: 'mergeStats'
    key: 'mergeStats'
    label: '合并统计'
    description: '合并过程的统计信息'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.object
  }
}
```

## 🔧 执行引擎机制

### AI节点调度器

```typescript
const aiNodeCallbackMap: Record<FlowNodeTypeEnum, Function> = {
  [FlowNodeTypeEnum.chatNode]: dispatchChatCompletion,
  [FlowNodeTypeEnum.agent]: dispatchAgent,
  [FlowNodeTypeEnum.classifyQuestion]: dispatchClassifyQuestion,
  [FlowNodeTypeEnum.contentExtract]: dispatchContentExtract,
  [FlowNodeTypeEnum.datasetSearchNode]: dispatchDatasetSearch,
  [FlowNodeTypeEnum.queryExtension]: dispatchQueryExtension,
  [FlowNodeTypeEnum.datasetConcatNode]: dispatchDatasetConcat
}
```

### 流式响应支持

```typescript
interface StreamingConfig {
  // 流式输出
  stream: boolean
  
  // 响应回调
  onMessage?: (data: {
    text: string
    type: 'text' | 'tool_call' | 'reasoning'
    nodeId: string
  }) => void
  
  // 完成回调
  onComplete?: (result: any) => void
  
  // 错误回调  
  onError?: (error: Error) => void
}
```

### 模型配置管理

```typescript
interface ModelConfig {
  // 模型标识
  model: string
  
  // 生成参数
  temperature: number
  maxTokens: number
  topP: number
  frequencyPenalty: number
  presencePenalty: number
  
  // 功能开关
  vision: boolean
  reasoning: boolean
  functionCall: boolean
  
  // 安全配置
  contentFilter: boolean
  safetyLevel: 'strict' | 'moderate' | 'permissive'
}
```

## 🎯 最佳实践模式

### 1. RAG增强模式

```
用户问题 → 查询扩展 → 知识库搜索 → 结果拼接 → AI聊天 → 答案输出
```

### 2. 智能分流模式  

```
用户问题 → 问题分类 → 不同AI代理 → 结果汇总 → 统一回复
```

### 3. 多步推理模式

```
用户任务 → AI代理 → 工具调用 → 结果分析 → 决策执行 → 最终输出
```

### 4. 内容处理管道

```
文档输入 → 内容提取 → 结构化存储 → 语义索引 → 智能问答
```

## 🔒 安全与质量控制

### 内容安全

- **敏感词过滤** - 自动检测和过滤敏感内容
- **输出审核** - AI生成内容的自动审核机制  
- **权限控制** - 基于角色的模型访问权限
- **日志记录** - 完整的AI调用和输出日志

### 质量保障

- **输出验证** - 结构化输出的格式和内容验证
- **重试机制** - 失败自动重试和降级策略
- **性能监控** - AI节点执行耗时和成功率监控
- **A/B测试** - 不同AI配置的效果对比

---

FastGPT 的AI处理节点体系为用户提供了完整的人工智能处理能力，从基础的对话生成到复杂的多步推理，从简单的分类识别到精确的信息提取，构建了一个强大而灵活的AI工作流平台。通过这些专业化的AI节点，用户可以构建出满足各种业务需求的智能应用。