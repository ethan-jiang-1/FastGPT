# FastGPT 数据处理节点深度分析

## 📊 数据处理节点概述

数据处理节点是 FastGPT 工作流编排系统的**数据基础设施**，负责所有与数据转换、文件处理、文本操作、变量管理和结果聚合相关的任务。这些节点专注于纯数据操作，不依赖AI模型，为工作流提供高效、可靠的数据处理能力。

### 核心设计理念

- **🔧 纯数据处理** - 专注数据操作，不依赖AI模型
- **⚡ 高性能执行** - 轻量级操作，快速响应
- **🔒 类型安全** - 强类型约束保证数据完整性
- **🎯 功能专精** - 每个节点专注特定的数据处理任务
- **🔄 可组合性** - 节点间可灵活组合形成复杂数据流
- **🛡️ 安全可靠** - 内置安全检查和错误处理机制

## 📊 数据处理节点分类

### 按处理类型分类

| 分类 | 节点类型 | 主要功能 | 典型场景 |
|------|----------|----------|----------|
| **文本处理** | `textEditor`, `readFiles` | 文本编辑、文件读取 | 模板渲染、文档解析 |
| **数据操作** | `variableUpdate`, `ifElse` | 变量更新、条件判断 | 数据流控制、状态管理 |
| **循环聚合** | `loop`, `loopStart`, `datasetConcat` | 循环处理、结果聚合 | 批量处理、数据合并 |
| **交互收集** | `userSelect`, `formInput` | 用户选择、表单输入 | 数据采集、用户交互 |
| **代码执行** | `code` | 自定义代码执行 | 复杂计算、数据转换 |
| **外部集成** | `httpRequest468` | HTTP请求处理 | API调用、数据同步 |

### 按数据流方向分类

| 方向 | 适用节点 | 数据特点 | 性能特征 |
|------|----------|----------|----------|
| **数据输入** | `readFiles`, `formInput`, `httpRequest468` | 外部数据导入 | I/O密集型 |
| **数据转换** | `textEditor`, `variableUpdate`, `code` | 数据格式转换 | CPU密集型 |
| **数据控制** | `ifElse`, `loop` | 流程控制逻辑 | 逻辑控制型 |
| **数据输出** | `userSelect`, `datasetConcat` | 结果展示聚合 | 展示友好型 |

## 🚀 核心数据处理节点详解

### 1. 文本编辑器节点 (textEditor)

**节点标识**: `FlowNodeTypeEnum.textEditor`  
**模板分类**: `FlowNodeTemplateTypeEnum.tools`  
**核心功能**: 文本模板渲染和变量替换

#### 输入接口定义

```typescript
interface TextEditorInputs {
  // 文本模板
  text: {
    key: 'text'
    renderTypeList: [FlowNodeInputTypeEnum.textarea, FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.string
    label: '文本内容'
    placeholder: '可以使用 {{变量名}} 来引用其他节点的输出'
    description: '支持变量引用的文本模板'
  }
  
  // 动态变量输入（自动生成）
  // 系统会自动识别模板中的变量并生成对应的输入项
  [variableName: string]: {
    key: string
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.any
    label: string
    canEdit: true
    editField: { key: string }
  }
}
```

#### 输出接口定义

```typescript
interface TextEditorOutputs {
  // 处理后的文本
  text: {
    id: 'text'
    key: 'text'
    label: '输出文本'
    description: '变量替换后的最终文本内容'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.string
  }
}
```

#### 变量替换机制

```typescript
interface VariableReplacement {
  // 变量语法
  syntax: '{{variableName}}'
  
  // 支持的数据类型转换
  typeConversion: {
    string: (value: any) => string
    number: (value: any) => string  
    boolean: (value: any) => 'true' | 'false'
    object: (value: any) => string  // JSON.stringify
    array: (value: any) => string   // 逗号分隔
  }
  
  // 嵌套对象访问
  nestedAccess: '{{user.profile.name}}'
  
  // 数组访问
  arrayAccess: '{{items[0].title}}'
  
  // 默认值支持
  defaultValue: '{{variable || "默认值"}}'
}
```

#### 使用示例

```typescript
// 输入模板
const template = `
尊敬的 {{customerName}}，

您的订单 {{order.id}} 已于 {{order.date}} 提交成功。
订单详情：
- 商品名称：{{order.items[0].name}}
- 数量：{{order.items[0].quantity}}
- 金额：￥{{order.totalAmount}}

预计发货时间：{{estimatedDelivery || "3-5个工作日"}}

感谢您的购买！
`

// 变量输入
const variables = {
  customerName: "张先生",
  order: {
    id: "ORD20241124001",
    date: "2024-11-24",
    items: [{ name: "AI编程书籍", quantity: 2 }],
    totalAmount: 128.00
  },
  estimatedDelivery: "2024-11-26"
}

// 输出结果
const result = `
尊敬的 张先生，

您的订单 ORD20241124001 已于 2024-11-24 提交成功。
订单详情：
- 商品名称：AI编程书籍  
- 数量：2
- 金额：￥128

预计发货时间：2024-11-26

感谢您的购买！
`
```

### 2. 文件读取节点 (readFiles)

**节点标识**: `FlowNodeTypeEnum.readFiles`  
**模板分类**: `FlowNodeTemplateTypeEnum.tools`  
**核心功能**: 多格式文件内容提取和处理

#### 输入接口定义

```typescript
interface ReadFilesInputs {
  // 文件列表
  fileUrls: {
    key: 'fileUrls'
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.arrayString
    label: '文件链接'
    required: true
    description: '需要读取的文件URL列表'
  }
  
  // 读取模式
  readMode: {
    key: 'readMode'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: '读取模式'
    value: 'content'
    list: [
      { label: '仅内容', value: 'content' },
      { label: '内容+元数据', value: 'detailed' },
      { label: '结构化解析', value: 'structured' }
    ]
  }
  
  // 字符编码
  encoding: {
    key: 'encoding'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: '字符编码'
    value: 'auto'
    list: [
      { label: '自动检测', value: 'auto' },
      { label: 'UTF-8', value: 'utf8' },
      { label: 'GBK', value: 'gbk' },
      { label: 'ASCII', value: 'ascii' }
    ]
  }
  
  // 最大文件大小(MB)
  maxFileSize: {
    key: 'maxFileSize'
    renderTypeList: [FlowNodeInputTypeEnum.numberInput]
    valueType: WorkflowIOValueTypeEnum.number
    label: '最大文件大小(MB)'
    value: 10
    min: 1
    max: 100
  }
}
```

#### 输出接口定义

```typescript
interface ReadFilesOutputs {
  // 文件内容
  content: {
    id: 'content'
    key: 'content'
    label: '文件内容'
    description: '所有文件的合并内容'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.string
  }
  
  // 文件详情
  filesDetail: {
    id: 'filesDetail'
    key: 'filesDetail'
    label: '文件详情'
    description: '每个文件的详细信息'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.arrayObject
  }
  
  // 处理状态
  processStats: {
    id: 'processStats'
    key: 'processStats'
    label: '处理统计'
    description: '文件处理的统计信息'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.object
  }
  
  // 错误信息
  errors: {
    id: 'errors'
    key: 'errors'
    label: '错误信息'
    description: '处理过程中的错误记录'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.arrayString
  }
}
```

#### 支持的文件格式

```typescript
interface SupportedFileTypes {
  // 文本文件
  text: {
    extensions: ['.txt', '.md', '.csv', '.json', '.xml', '.yaml', '.yml']
    maxSize: '50MB'
    encoding: 'auto-detect'
  }
  
  // Office文档
  office: {
    extensions: ['.docx', '.xlsx', '.pptx']
    maxSize: '100MB'
    features: ['text_extraction', 'table_parsing', 'image_extraction']
  }
  
  // PDF文档
  pdf: {
    extensions: ['.pdf']
    maxSize: '100MB'
    features: ['text_extraction', 'ocr_support', 'table_detection']
  }
  
  // 代码文件
  code: {
    extensions: ['.js', '.ts', '.py', '.java', '.cpp', '.go', '.rust']
    maxSize: '10MB'
    features: ['syntax_highlighting', 'comment_extraction']
  }
  
  // 配置文件
  config: {
    extensions: ['.ini', '.conf', '.toml', '.properties']
    maxSize: '5MB'
    features: ['key_value_parsing', 'section_detection']
  }
}
```

#### 文件处理管道

```typescript
interface FileProcessingPipeline {
  // 1. 文件获取
  fetch: {
    urlValidation: boolean      // URL合法性检查
    sizeLimit: number          // 文件大小限制
    typeCheck: boolean         // 文件类型验证
    securityScan: boolean      // 安全扫描
  }
  
  // 2. 内容提取
  extraction: {
    encodingDetection: boolean  // 编码检测
    formatParsing: boolean     // 格式解析
    structureAnalysis: boolean // 结构分析
    errorHandling: boolean     // 错误处理
  }
  
  // 3. 内容处理
  processing: {
    textCleaning: boolean      // 文本清理
    formatNormalization: boolean // 格式标准化
    metadataExtraction: boolean // 元数据提取
    contentCaching: boolean     // 内容缓存
  }
  
  // 4. 结果输出
  output: {
    contentMerging: boolean     // 内容合并
    statisticsGeneration: boolean // 统计生成
    errorReporting: boolean     // 错误报告
    formatValidation: boolean   // 输出验证
  }
}
```

### 3. 变量更新节点 (variableUpdate)

**节点标识**: `FlowNodeTypeEnum.variableUpdate`  
**模板分类**: `FlowNodeTemplateTypeEnum.tools`  
**核心功能**: 工作流变量的动态更新和管理

#### 输入接口定义

```typescript
interface VariableUpdateInputs {
  // 变量配置
  updateList: {
    key: 'updateList'
    renderTypeList: [FlowNodeInputTypeEnum.custom]
    valueType: WorkflowIOValueTypeEnum.any
    label: '变量更新列表'
    description: '需要更新的变量配置'
    value: [] // VariableUpdateConfig[]
  }
}

interface VariableUpdateConfig {
  // 变量键名
  key: string
  
  // 变量值
  value: {
    type: 'reference' | 'input'     // 引用值或直接输入
    source?: string                 // 引用源节点ID
    field?: string                  // 引用字段名
    inputValue?: any               // 直接输入值
  }
  
  // 数据类型
  valueType: WorkflowIOValueTypeEnum
  
  // 更新模式
  updateMode: 'replace' | 'merge' | 'append'
  
  // 条件更新
  condition?: {
    enable: boolean
    field: string
    operator: 'equal' | 'notEqual' | 'exists' | 'notExists'
    value: any
  }
}
```

#### 输出接口定义

```typescript
interface VariableUpdateOutputs {
  // 更新的变量（动态生成）
  [variableKey: string]: {
    id: string
    key: string
    label: string
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum
  }
  
  // 更新统计
  updateStats: {
    id: 'updateStats'
    key: 'updateStats'
    label: '更新统计'
    description: '变量更新的统计信息'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.object
  }
}
```

#### 变量更新策略

```typescript
interface VariableUpdateStrategy {
  // 替换模式
  replace: {
    description: '完全替换原变量值'
    applicableTypes: ['string', 'number', 'boolean', 'object', 'array']
    behavior: (oldValue: any, newValue: any) => any
  }
  
  // 合并模式
  merge: {
    description: '对象属性合并，数组元素追加'
    applicableTypes: ['object', 'array']
    behavior: {
      object: (oldObj: object, newObj: object) => object  // 深度合并
      array: (oldArr: any[], newArr: any[]) => any[]      // 数组连接
    }
  }
  
  // 追加模式
  append: {
    description: '字符串拼接，数组元素追加'
    applicableTypes: ['string', 'array']
    behavior: {
      string: (oldStr: string, newStr: string) => string  // 字符串连接
      array: (oldArr: any[], newItem: any) => any[]       // 元素追加
    }
  }
}
```

#### 使用示例

```typescript
// 配置示例：用户信息管理
const variableUpdateConfig: VariableUpdateConfig[] = [
  {
    key: 'userProfile',
    value: {
      type: 'reference',
      source: 'formInput_node_id',
      field: 'userInfo'
    },
    valueType: WorkflowIOValueTypeEnum.object,
    updateMode: 'merge'
  },
  {
    key: 'processStep',
    value: {
      type: 'input',
      inputValue: 'profile_completed'
    },
    valueType: WorkflowIOValueTypeEnum.string,
    updateMode: 'replace'
  },
  {
    key: 'tags',
    value: {
      type: 'reference',
      source: 'classify_node_id', 
      field: 'categories'
    },
    valueType: WorkflowIOValueTypeEnum.arrayString,
    updateMode: 'append',
    condition: {
      enable: true,
      field: 'confidence',
      operator: 'equal',
      value: true
    }
  }
]
```

### 4. 条件判断节点 (ifElse)

**节点标识**: `FlowNodeTypeEnum.ifElseNode`  
**模板分类**: `FlowNodeTemplateTypeEnum.tools`  
**核心功能**: 基于条件的流程控制和数据路由

#### 输入接口定义

```typescript
interface IFElseInputs {
  // 条件配置
  ifElseList: {
    key: 'ifElseList'
    renderTypeList: [FlowNodeInputTypeEnum.custom]
    valueType: WorkflowIOValueTypeEnum.any
    label: '条件配置'
    description: '条件判断规则列表'
    value: [] // ConditionConfig[]
  }
}

interface ConditionConfig {
  // 条件标识
  id: string
  
  // 条件描述
  description: string
  
  // 判断字段
  variable: {
    type: 'reference' | 'input'
    source?: string  // 引用源
    field?: string   // 字段名
    value?: any      // 直接值
  }
  
  // 判断条件
  condition: {
    type: ConditionTypeEnum
    value?: any           // 比较值
    regex?: string        // 正则表达式
    caseSensitive?: boolean // 大小写敏感
  }
  
  // 逻辑关系
  logic?: 'AND' | 'OR'
}

enum ConditionTypeEnum {
  isEmpty = 'isEmpty',
  isNotEmpty = 'isNotEmpty',
  equalTo = 'equalTo',
  notEqualTo = 'notEqualTo',
  greaterThan = 'greaterThan',
  lessThan = 'lessThan',
  greaterThanOrEqualTo = 'greaterThanOrEqualTo',
  lessThanOrEqualTo = 'lessThanOrEqualTo',
  include = 'include',
  notInclude = 'notInclude',
  startWith = 'startWith',
  endWith = 'endWith',
  regexMatch = 'regexMatch',
  lengthEqualTo = 'lengthEqualTo',
  lengthGreaterThan = 'lengthGreaterThan',
  lengthLessThan = 'lengthLessThan'
}
```

#### 输出接口定义

```typescript
interface IFElseOutputs {
  // 条件结果（动态生成）
  [conditionId: string]: {
    id: string
    key: string
    label: string
    type: FlowNodeOutputTypeEnum.source
    valueType: WorkflowIOValueTypeEnum.boolean
  }
  
  // 条件详情
  conditionResults: {
    id: 'conditionResults'
    key: 'conditionResults'
    label: '条件判断结果'
    description: '所有条件的判断结果详情'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.arrayObject
  }
}
```

#### 条件判断逻辑

```typescript
interface ConditionEvaluator {
  // 空值判断
  isEmpty: (value: any) => boolean
  isNotEmpty: (value: any) => boolean
  
  // 相等判断
  equalTo: (value: any, target: any) => boolean
  notEqualTo: (value: any, target: any) => boolean
  
  // 数值比较
  greaterThan: (value: number, target: number) => boolean
  lessThan: (value: number, target: number) => boolean
  greaterThanOrEqualTo: (value: number, target: number) => boolean
  lessThanOrEqualTo: (value: number, target: number) => boolean
  
  // 字符串操作
  include: (value: string, target: string) => boolean
  notInclude: (value: string, target: string) => boolean
  startWith: (value: string, target: string) => boolean
  endWith: (value: string, target: string) => boolean
  regexMatch: (value: string, pattern: string) => boolean
  
  // 长度判断
  lengthEqualTo: (value: string | any[], target: number) => boolean
  lengthGreaterThan: (value: string | any[], target: number) => boolean
  lengthLessThan: (value: string | any[], target: number) => boolean
}
```

#### 复杂条件组合示例

```typescript
// 用户权限验证示例
const permissionCheck: ConditionConfig[] = [
  {
    id: 'isAdmin',
    description: '管理员权限检查',
    variable: {
      type: 'reference',
      source: 'userAuth_node',
      field: 'role'
    },
    condition: {
      type: ConditionTypeEnum.equalTo,
      value: 'admin'
    }
  },
  {
    id: 'isVipUser',
    description: 'VIP用户检查',
    variable: {
      type: 'reference',
      source: 'userProfile_node',
      field: 'membershipLevel'
    },
    condition: {
      type: ConditionTypeEnum.include,
      value: 'VIP'
    },
    logic: 'OR'
  },
  {
    id: 'hasValidSubscription',
    description: '订阅状态检查',
    variable: {
      type: 'reference',
      source: 'subscription_node',
      field: 'status'
    },
    condition: {
      type: ConditionTypeEnum.equalTo,
      value: 'active'
    },
    logic: 'AND'
  }
]
```

### 5. 循环处理节点组 (loop, loopStart)

**节点标识**: `FlowNodeTypeEnum.loop`, `FlowNodeTypeEnum.loopStart`  
**模板分类**: `FlowNodeTemplateTypeEnum.tools`  
**核心功能**: 数组数据的批量处理和结果聚合

#### 循环开始节点 (loopStart)

```typescript
interface LoopStartInputs {
  // 循环数据
  loopArray: {
    key: 'loopArray'
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.any
    label: '循环数组'
    required: true
    description: '需要循环处理的数组数据'
  }
  
  // 批处理大小
  batchSize: {
    key: 'batchSize'
    renderTypeList: [FlowNodeInputTypeEnum.numberInput]
    valueType: WorkflowIOValueTypeEnum.number
    label: '批处理大小'
    value: 10
    min: 1
    max: 100
    description: '每批处理的数据量'
  }
  
  // 并发控制
  concurrency: {
    key: 'concurrency'
    renderTypeList: [FlowNodeInputTypeEnum.numberInput]
    valueType: WorkflowIOValueTypeEnum.number
    label: '并发数量'
    value: 3
    min: 1
    max: 10
    description: '同时处理的并发数'
  }
}

interface LoopStartOutputs {
  // 当前项
  currentItem: {
    id: 'currentItem'
    key: 'currentItem'
    label: '当前项'
    description: '当前循环处理的数据项'
    type: FlowNodeOutputTypeEnum.source
    valueType: WorkflowIOValueTypeEnum.any
  }
  
  // 当前索引
  currentIndex: {
    id: 'currentIndex'
    key: 'currentIndex'
    label: '当前索引'
    description: '当前项在数组中的索引位置'
    type: FlowNodeOutputTypeEnum.source
    valueType: WorkflowIOValueTypeEnum.number
  }
  
  // 总数量
  totalCount: {
    id: 'totalCount'
    key: 'totalCount'
    label: '总数量'
    description: '循环数组的总长度'
    type: FlowNodeOutputTypeEnum.source
    valueType: WorkflowIOValueTypeEnum.number
  }
}
```

#### 循环结束节点 (loop)

```typescript
interface LoopInputs {
  // 循环结果收集
  loopResults: {
    key: 'loopResults'
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.any
    label: '循环结果'
    description: '每次循环的处理结果'
  }
  
  // 聚合模式
  aggregationMode: {
    key: 'aggregationMode'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: '聚合模式'
    value: 'array'
    list: [
      { label: '数组聚合', value: 'array' },
      { label: '字符串拼接', value: 'concat' },
      { label: '数值求和', value: 'sum' },
      { label: '统计计数', value: 'count' }
    ]
  }
}

interface LoopOutputs {
  // 聚合结果
  aggregatedResult: {
    id: 'aggregatedResult'
    key: 'aggregatedResult'
    label: '聚合结果'
    description: '所有循环结果的聚合数据'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.any
  }
  
  // 处理统计
  processStats: {
    id: 'processStats'
    key: 'processStats'
    label: '处理统计'
    description: '循环处理的统计信息'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.object
  }
}
```

#### 循环处理模式

```typescript
interface LoopProcessingMode {
  // 顺序处理
  sequential: {
    description: '按顺序逐个处理数组元素'
    advantages: ['内存占用低', '结果顺序可控']
    disadvantages: ['处理速度较慢']
    applicableScenarios: ['依赖顺序的处理', '资源敏感场景']
  }
  
  // 并发处理
  concurrent: {
    description: '多个元素同时并发处理'
    advantages: ['处理速度快', '资源利用率高']
    disadvantages: ['内存占用高', '结果顺序不确定']
    applicableScenarios: ['独立元素处理', '高性能需求']
  }
  
  // 批处理
  batch: {
    description: '将数组分批处理'
    advantages: ['内存可控', '错误隔离']
    disadvantages: ['实现复杂度高']
    applicableScenarios: ['大数据量处理', '容错要求高']
  }
}
```

### 6. 代码执行节点 (code)

**节点标识**: `FlowNodeTypeEnum.code`  
**模板分类**: `FlowNodeTemplateTypeEnum.tools`  
**核心功能**: 自定义代码执行和复杂数据处理

#### 输入接口定义

```typescript
interface CodeNodeInputs {
  // 动态输入参数
  [paramName: string]: {
    key: string
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum
    canEdit: true
    customInputConfig: {
      selectValueTypeList: WorkflowIOValueTypeEnum[]
      showDescription: false
      showDefaultValue: true
    }
    required: true
  }
  
  // 代码类型
  codeType: {
    key: 'codeType'
    renderTypeList: [FlowNodeInputTypeEnum.hidden]
    label: '代码类型'
    valueType: WorkflowIOValueTypeEnum.string
    value: 'js' // 'js' | 'python'
  }
  
  // 代码内容
  code: {
    key: 'code'
    renderTypeList: [FlowNodeInputTypeEnum.custom]
    label: '代码内容'
    valueType: WorkflowIOValueTypeEnum.string
    value: string // 默认代码模板
  }
}
```

#### 输出接口定义

```typescript
interface CodeNodeOutputs {
  // 动态输出（基于代码返回值）
  [outputKey: string]: {
    id: string
    type: FlowNodeOutputTypeEnum.dynamic
    key: string
    valueType: WorkflowIOValueTypeEnum
    label: string
  }
  
  // 完整响应
  rawResponse: {
    id: 'rawResponse'
    key: 'rawResponse'
    label: '完整响应数据'
    valueType: WorkflowIOValueTypeEnum.object
    type: FlowNodeOutputTypeEnum.static
  }
  
  // 错误信息
  error: {
    id: 'error'
    key: 'error'
    label: '错误信息'
    valueType: WorkflowIOValueTypeEnum.string
    type: FlowNodeOutputTypeEnum.error
  }
}
```

#### JavaScript代码模板

```javascript
// 默认JavaScript模板
function main({data1, data2}) {
  // 在这里编写你的处理逻辑
  
  // 数据处理示例
  const processedData = {
    originalData1: data1,
    originalData2: data2,
    processedAt: new Date().toISOString()
  }
  
  // 执行自定义逻辑
  if (typeof data1 === 'string') {
    processedData.data1Length = data1.length
    processedData.data1Upper = data1.toUpperCase()
  }
  
  if (Array.isArray(data2)) {
    processedData.data2Count = data2.length
    processedData.data2First = data2[0]
  }
  
  // 返回结果对象
  return {
    result: processedData,
    summary: `处理完成，数据1类型: ${typeof data1}, 数据2类型: ${typeof data2}`,
    timestamp: Date.now()
  }
}
```

#### Python代码模板

```python
# 默认Python模板
def main(data1, data2):
    """
    主处理函数
    
    Args:
        data1: 输入数据1
        data2: 输入数据2
        
    Returns:
        dict: 处理结果
    """
    import json
    from datetime import datetime
    
    # 数据处理逻辑
    processed_data = {
        'original_data1': data1,
        'original_data2': data2,
        'processed_at': datetime.now().isoformat()
    }
    
    # 字符串处理
    if isinstance(data1, str):
        processed_data['data1_length'] = len(data1)
        processed_data['data1_words'] = len(data1.split())
    
    # 列表处理
    if isinstance(data2, list):
        processed_data['data2_count'] = len(data2)
        processed_data['data2_sum'] = sum(x for x in data2 if isinstance(x, (int, float)))
    
    return {
        'result': processed_data,
        'summary': f'处理完成，数据1类型: {type(data1).__name__}, 数据2类型: {type(data2).__name__}',
        'success': True
    }
```

#### 代码执行环境

```typescript
interface CodeExecutionEnvironment {
  // JavaScript环境
  javascript: {
    runtime: 'Node.js'
    version: '18+'
    availableModules: ['lodash', 'moment', 'crypto', 'buffer']
    restrictions: {
      fileSystemAccess: false
      networkAccess: false
      processAccess: false
    }
    timeout: 30000 // 30秒
    memoryLimit: '128MB'
  }
  
  // Python环境
  python: {
    runtime: 'Python'
    version: '3.9+'
    availableModules: ['json', 'datetime', 'math', 'random', 'base64']
    restrictions: {
      fileSystemAccess: false
      networkAccess: false
      subprocessAccess: false
    }
    timeout: 30000 // 30秒
    memoryLimit: '128MB'
  }
}
```

### 7. HTTP请求节点 (httpRequest468)

**节点标识**: `FlowNodeTypeEnum.httpRequest468`  
**模板分类**: `FlowNodeTemplateTypeEnum.tools`  
**核心功能**: 外部API调用和数据集成

#### 输入接口定义

```typescript
interface HttpRequestInputs {
  // 请求URL
  url: {
    key: 'url'
    renderTypeList: [FlowNodeInputTypeEnum.input, FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.string
    label: '请求地址'
    placeholder: 'https://api.example.com/data'
    required: true
  }
  
  // 请求方法
  httpMethod: {
    key: 'httpMethod'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: 'HTTP方法'
    value: 'GET'
    list: [
      { label: 'GET', value: 'GET' },
      { label: 'POST', value: 'POST' },
      { label: 'PUT', value: 'PUT' },
      { label: 'DELETE', value: 'DELETE' },
      { label: 'PATCH', value: 'PATCH' }
    ]
  }
  
  // 请求头
  headers: {
    key: 'headers'
    renderTypeList: [FlowNodeInputTypeEnum.custom]
    valueType: WorkflowIOValueTypeEnum.any
    label: '请求头'
    description: 'HTTP请求头配置'
    value: [] // HeaderConfig[]
  }
  
  // 请求参数
  params: {
    key: 'params'
    renderTypeList: [FlowNodeInputTypeEnum.custom]
    valueType: WorkflowIOValueTypeEnum.any
    label: '请求参数'
    description: 'URL查询参数或请求体参数'
    value: [] // ParamConfig[]
  }
  
  // 内容类型
  contentType: {
    key: 'contentType'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: '内容类型'
    value: 'json'
    list: [
      { label: 'JSON', value: 'json' },
      { label: 'Form Data', value: 'form-data' },
      { label: 'Form URL Encoded', value: 'x-www-form-urlencoded' },
      { label: 'XML', value: 'xml' },
      { label: 'Raw Text', value: 'raw' }
    ]
  }
  
  // 超时设置
  timeout: {
    key: 'timeout'
    renderTypeList: [FlowNodeInputTypeEnum.numberInput]
    valueType: WorkflowIOValueTypeEnum.number
    label: '超时时间(秒)'
    value: 30
    min: 1
    max: 300
  }
  
  // 重试配置
  retryConfig: {
    key: 'retryConfig'
    renderTypeList: [FlowNodeInputTypeEnum.custom]
    valueType: WorkflowIOValueTypeEnum.object
    label: '重试配置'
    value: {
      maxRetries: 3,
      retryDelay: 1000,
      retryOnStatus: [500, 502, 503, 504]
    }
  }
}

interface HeaderConfig {
  key: string
  value: string
  description?: string
}

interface ParamConfig {
  key: string
  value: any
  type: 'string' | 'number' | 'boolean' | 'object'
  required: boolean
}
```

#### 输出接口定义

```typescript
interface HttpRequestOutputs {
  // 响应数据
  httpRawResponse: {
    id: 'httpRawResponse'
    key: 'httpRawResponse'
    label: '原始响应'
    description: 'HTTP请求的完整响应数据'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.object
  }
  
  // 响应状态
  httpStatus: {
    id: 'httpStatus'
    key: 'httpStatus'
    label: '响应状态'
    description: 'HTTP响应状态码'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.number
  }
  
  // 响应体
  httpBody: {
    id: 'httpBody'
    key: 'httpBody'
    label: '响应内容'
    description: 'HTTP响应体内容'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.string
  }
  
  // 响应头
  httpHeaders: {
    id: 'httpHeaders'
    key: 'httpHeaders'
    label: '响应头'
    description: 'HTTP响应头信息'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.object
  }
  
  // 错误信息
  error: {
    id: 'error'
    key: 'error'
    label: '错误信息'
    description: '请求失败时的错误信息'
    type: FlowNodeOutputTypeEnum.error
    valueType: WorkflowIOValueTypeEnum.string
  }
}
```

#### 请求配置示例

```typescript
// REST API调用示例
const apiRequestConfig = {
  url: 'https://api.github.com/repos/{{owner}}/{{repo}}/issues',
  httpMethod: 'POST',
  headers: [
    { key: 'Authorization', value: 'Bearer {{github_token}}' },
    { key: 'Accept', value: 'application/vnd.github.v3+json' },
    { key: 'User-Agent', value: 'FastGPT-Workflow' }
  ],
  params: [
    { key: 'title', value: '{{issue_title}}', type: 'string', required: true },
    { key: 'body', value: '{{issue_body}}', type: 'string', required: false },
    { key: 'labels', value: ['bug', 'urgent'], type: 'object', required: false }
  ],
  contentType: 'json',
  timeout: 30,
  retryConfig: {
    maxRetries: 3,
    retryDelay: 2000,
    retryOnStatus: [429, 500, 502, 503, 504]
  }
}

// 微信API调用示例
const wechatApiConfig = {
  url: 'https://api.weixin.qq.com/cgi-bin/message/template/send',
  httpMethod: 'POST',
  headers: [
    { key: 'Content-Type', value: 'application/json' }
  ],
  params: [
    { key: 'access_token', value: '{{wechat_access_token}}', type: 'string', required: true },
    { key: 'touser', value: '{{user_openid}}', type: 'string', required: true },
    { key: 'template_id', value: '{{template_id}}', type: 'string', required: true },
    { key: 'data', value: {
        'first': {'value': '{{title}}'},
        'keyword1': {'value': '{{content}}'},
        'remark': {'value': '{{remark}}'}
      }, type: 'object', required: true }
  ],
  contentType: 'json'
}
```

## 🔧 执行引擎特性

### 并发处理能力

```typescript
interface ConcurrencyConfig {
  // 节点级并发
  nodeLevel: {
    maxConcurrentNodes: 10      // 最大并发节点数
    executionQueue: 'priority'  // 执行队列策略
    resourceLimit: {
      memory: '512MB',
      cpu: '2cores'
    }
  }
  
  // HTTP请求并发
  httpLevel: {
    maxConcurrentRequests: 20   // 最大并发请求数
    connectionPoolSize: 50      // 连接池大小
    keepAlive: true            // 保持连接
    timeout: 30000             // 超时时间
  }
  
  // 文件处理并发
  fileLevel: {
    maxConcurrentFiles: 5       // 最大并发文件数
    maxFileSize: '100MB',       // 单文件大小限制
    totalMemoryLimit: '1GB'     // 总内存限制
  }
}
```

### 缓存机制

```typescript
interface CachingStrategy {
  // HTTP请求缓存
  httpCache: {
    enabled: boolean
    ttl: number              // 缓存时间(秒)
    maxSize: '100MB'         // 缓存大小
    keyStrategy: 'url+params' // 缓存键策略
  }
  
  // 文件内容缓存
  fileCache: {
    enabled: boolean
    ttl: 1200               // 20分钟
    maxFiles: 1000          // 最大文件数
    compressionEnabled: true // 压缩存储
  }
  
  // 计算结果缓存
  computeCache: {
    enabled: boolean
    ttl: 3600              // 1小时
    maxEntries: 10000      // 最大条目数
  }
}
```

### 错误处理策略

```typescript
interface ErrorHandlingStrategy {
  // 重试策略
  retry: {
    maxRetries: 3
    backoffStrategy: 'exponential' | 'linear' | 'fixed'
    initialDelay: 1000
    maxDelay: 30000
    retryableErrors: ['TIMEOUT', 'CONNECTION_ERROR', 'SERVER_ERROR']
  }
  
  // 降级策略
  fallback: {
    enabled: boolean
    fallbackValue: any      // 降级默认值
    fallbackSource?: string // 降级数据源
  }
  
  // 错误传播
  propagation: {
    skipOnError: boolean    // 错误时跳过后续节点
    errorOutput: boolean    // 输出错误信息
    logLevel: 'error' | 'warn' | 'info'
  }
}
```

## 🎯 最佳实践模式

### 1. ETL数据管道

```
外部数据源 → HTTP请求 → 数据清洗(代码) → 格式转换(文本编辑) → 条件筛选 → 结果存储
```

### 2. 批量文件处理

```
文件列表 → 循环开始 → 文件读取 → 内容处理(代码) → 结果聚合 → 循环结束
```

### 3. 智能数据路由

```
数据输入 → 条件判断 → 分支处理A/B → 变量更新 → 结果合并
```

### 4. API集成工作流

```
参数准备 → HTTP请求 → 响应解析(代码) → 错误处理 → 结果格式化
```

## 🔒 安全与性能优化

### 安全控制

- **输入验证** - 严格的参数和数据类型验证
- **执行隔离** - 代码和网络请求的沙箱执行
- **访问控制** - 内网地址和敏感接口保护
- **资源限制** - 内存、CPU、网络的使用限制

### 性能优化

- **智能缓存** - 多层缓存减少重复计算
- **并发执行** - 合理的并发控制提升效率
- **资源复用** - 连接池和对象池优化
- **懒加载** - 按需加载和执行优化

---

FastGPT 的数据处理节点体系为工作流提供了强大的数据操作基础能力。通过这些专业化的节点，用户可以构建出高效、可靠的数据处理管道，实现从简单的文本处理到复杂的数据转换和集成任务。这些节点与AI处理节点的完美结合，形成了一个功能完整、性能卓越的工作流编排平台。