# FastGPT 外部集成节点深度分析

## 🌐 外部集成节点概述

外部集成节点是 FastGPT 工作流编排系统的**对外连接桥梁**，负责与第三方服务、API接口、插件系统和外部工具的集成交互。这些节点将 FastGPT 的内部能力扩展到整个互联网生态，实现无边界的工作流集成能力。

### 核心设计理念

- **🔌 开放集成** - 支持任意第三方服务的灵活集成
- **🛡️ 安全可控** - 完善的认证授权和安全隔离机制
- **⚡ 高性能** - 异步执行和连接池优化
- **🔧 标准化** - 遵循行业标准协议和最佳实践
- **📦 可扩展** - 模块化设计支持自定义扩展
- **🎯 智能化** - AI驱动的工具选择和参数推理

## 📊 外部集成节点分类

### 按集成类型分类

| 分类 | 节点类型 | 主要功能 | 集成范围 |
|------|----------|----------|----------|
| **API集成** | `httpRequest468` | HTTP API调用 | REST API、GraphQL、Webhook |
| **插件系统** | `runPlugin`, `pluginModule` | 自定义插件执行 | 团队插件、系统插件 |
| **工具集成** | `runTool`, `agent` | 外部工具调用 | MCP工具、系统工具 |
| **代码执行** | `code`, `lafModule` | 代码沙箱执行 | JavaScript、Python、云函数 |
| **文件集成** | `readFiles` | 外部文件处理 | 文档解析、内容提取 |

### 按技术协议分类

| 协议 | 适用节点 | 技术特点 | 应用场景 |
|------|----------|----------|----------|
| **HTTP/HTTPS** | `httpRequest468`, `lafModule` | RESTful API调用 | 通用API集成 |
| **MCP协议** | `runTool` | 模型上下文协议 | AI工具标准化 |
| **Plugin API** | `runPlugin`, `pluginModule` | 插件系统协议 | 自定义功能扩展 |
| **File Protocol** | `readFiles` | 文件访问协议 | 文档内容处理 |
| **Serverless** | `lafModule`, `code` | 函数计算协议 | 云端代码执行 |

## 🚀 核心外部集成节点详解

### 1. HTTP请求节点 (httpRequest468)

**节点标识**: `FlowNodeTypeEnum.httpRequest468`  
**模板分类**: `FlowNodeTemplateTypeEnum.tools`  
**核心功能**: 通用HTTP API集成和第三方服务调用

#### 输入接口定义

```typescript
interface HttpRequest468Inputs {
  // 请求URL
  url: {
    key: 'url'
    renderTypeList: [FlowNodeInputTypeEnum.input, FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.string
    label: '请求地址'
    placeholder: 'https://api.example.com/{{endpoint}}'
    required: true
    description: '支持变量替换的API端点地址'
  }
  
  // HTTP方法
  httpMethod: {
    key: 'httpMethod'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: 'HTTP方法'
    value: 'POST'
    list: [
      { label: 'GET', value: 'GET' },
      { label: 'POST', value: 'POST' },
      { label: 'PUT', value: 'PUT' },
      { label: 'DELETE', value: 'DELETE' },
      { label: 'PATCH', value: 'PATCH' },
      { label: 'HEAD', value: 'HEAD' },
      { label: 'OPTIONS', value: 'OPTIONS' }
    ]
  }
  
  // 请求头配置
  headers: {
    key: 'headers'
    renderTypeList: [FlowNodeInputTypeEnum.custom]
    valueType: WorkflowIOValueTypeEnum.any
    label: '请求头'
    description: 'HTTP请求头配置，支持密钥管理'
    value: [] // HttpHeaderConfig[]
  }
  
  // 请求参数
  params: {
    key: 'params'
    renderTypeList: [FlowNodeInputTypeEnum.custom]
    valueType: WorkflowIOValueTypeEnum.any
    label: '请求参数'
    description: '请求体参数或查询参数'
    value: [] // HttpParamConfig[]
  }
  
  // 内容类型
  httpContentType: {
    key: 'httpContentType'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: '请求格式'
    value: 'json'
    list: [
      { label: 'JSON', value: 'json' },
      { label: 'Form Data', value: 'form-data' },
      { label: 'URL Encoded', value: 'x-www-form-urlencoded' },
      { label: 'Raw Text', value: 'raw' },
      { label: 'XML', value: 'xml' }
    ]
  }
  
  // 响应解析配置
  extractResponse: {
    key: 'extractResponse'
    renderTypeList: [FlowNodeInputTypeEnum.custom]
    valueType: WorkflowIOValueTypeEnum.any
    label: '响应提取'
    description: 'JSONPath提取响应中的特定字段'
    value: [] // ResponseExtractionConfig[]
  }
}

interface HttpHeaderConfig {
  key: string              // 请求头名称
  value: string            // 请求头值(支持变量替换)
  type: 'string' | 'secret' // 值类型(普通字符串或密钥)
  description?: string     // 描述信息
}

interface HttpParamConfig {
  key: string              // 参数名
  value: any              // 参数值(支持变量替换)
  type: WorkflowIOValueTypeEnum // 参数数据类型
  required: boolean       // 是否必填
  description?: string    // 参数描述
}

interface ResponseExtractionConfig {
  key: string             // 提取字段名
  jsonPath: string        // JSONPath表达式
  valueType: WorkflowIOValueTypeEnum // 提取值类型
  required: boolean       // 是否必需
  defaultValue?: any      // 默认值
}
```

#### 输出接口定义

```typescript
interface HttpRequest468Outputs {
  // HTTP状态码
  httpStatus: {
    id: 'httpStatus'
    key: 'httpStatus'
    label: 'HTTP状态码'
    description: 'HTTP响应状态码'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.number
  }
  
  // 原始响应
  httpRawResponse: {
    id: 'httpRawResponse'
    key: 'httpRawResponse'
    label: '原始响应'
    description: 'HTTP响应的完整内容'
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
  
  // 提取的字段(动态生成基于extractResponse配置)
  [extractedField: string]: {
    id: string
    key: string
    label: string
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum
  }
  
  // 错误信息
  httpError: {
    id: 'httpError'
    key: 'httpError'
    label: '请求错误'
    description: 'HTTP请求失败时的错误信息'
    type: FlowNodeOutputTypeEnum.error
    valueType: WorkflowIOValueTypeEnum.string
  }
}
```

#### HTTP集成示例配置

```typescript
// OpenAI API集成示例
const openaiIntegration: HttpRequest468Config = {
  url: 'https://api.openai.com/v1/chat/completions',
  httpMethod: 'POST',
  headers: [
    {
      key: 'Authorization',
      value: 'Bearer {{openai_api_key}}',
      type: 'secret',
      description: 'OpenAI API密钥'
    },
    {
      key: 'Content-Type',
      value: 'application/json',
      type: 'string'
    }
  ],
  params: [
    {
      key: 'model',
      value: 'gpt-4',
      type: WorkflowIOValueTypeEnum.string,
      required: true
    },
    {
      key: 'messages',
      value: [
        {
          role: 'user',
          content: '{{user_input}}'
        }
      ],
      type: WorkflowIOValueTypeEnum.arrayObject,
      required: true
    },
    {
      key: 'temperature',
      value: 0.7,
      type: WorkflowIOValueTypeEnum.number,
      required: false
    }
  ],
  httpContentType: 'json',
  extractResponse: [
    {
      key: 'aiResponse',
      jsonPath: '$.choices[0].message.content',
      valueType: WorkflowIOValueTypeEnum.string,
      required: true
    },
    {
      key: 'tokenUsage',
      jsonPath: '$.usage.total_tokens',
      valueType: WorkflowIOValueTypeEnum.number,
      required: false
    }
  ]
}

// 微信企业微信机器人集成示例
const wechatBotIntegration: HttpRequest468Config = {
  url: 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key={{webhook_key}}',
  httpMethod: 'POST',
  headers: [
    {
      key: 'Content-Type',
      value: 'application/json',
      type: 'string'
    }
  ],
  params: [
    {
      key: 'msgtype',
      value: 'text',
      type: WorkflowIOValueTypeEnum.string,
      required: true
    },
    {
      key: 'text',
      value: {
        content: '{{message_content}}',
        mentioned_list: ['{{mentioned_users}}']
      },
      type: WorkflowIOValueTypeEnum.object,
      required: true
    }
  ],
  httpContentType: 'json',
  extractResponse: [
    {
      key: 'sendSuccess',
      jsonPath: '$.errcode',
      valueType: WorkflowIOValueTypeEnum.number,
      required: true,
      defaultValue: -1
    }
  ]
}
```

#### 高级HTTP特性

```typescript
interface AdvancedHttpFeatures {
  // 连接管理
  connectionManagement: {
    keepAlive: boolean           // 保持连接
    connectionTimeout: number    // 连接超时(毫秒)
    readTimeout: number         // 读取超时(毫秒)
    maxConnections: number      // 最大连接数
    retryAttempts: number       // 重试次数
    retryDelay: number          // 重试延迟
  }
  
  // 安全配置
  securityConfig: {
    validateSSL: boolean        // SSL证书验证
    allowSelfSigned: boolean    // 允许自签名证书
    followRedirects: boolean    // 跟随重定向
    maxRedirects: number        // 最大重定向次数
    userAgent: string           // 用户代理字符串
  }
  
  // 缓存策略
  cachingStrategy: {
    enableCache: boolean        // 启用缓存
    cacheTimeout: number        // 缓存超时(秒)
    cacheKey: string           // 缓存键策略
    varyHeaders: string[]       // 变化头列表
  }
  
  // 响应处理
  responseProcessing: {
    maxResponseSize: number     // 最大响应大小(字节)
    encoding: string           // 响应编码
    parseJson: boolean         // 自动JSON解析
    validateSchema: boolean    // JSON Schema验证
  }
}
```

### 2. 插件执行节点 (runPlugin)

**节点标识**: `FlowNodeTypeEnum.runPlugin`  
**模板分类**: `FlowNodeTemplateTypeEnum.function`  
**核心功能**: 自定义插件和工作流模块的执行

#### 输入接口定义

```typescript
interface RunPluginInputs {
  // 插件选择
  pluginId: {
    key: 'pluginId'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: '选择插件'
    required: true
    description: '要执行的插件ID'
  }
  
  // 动态输入参数(基于插件配置生成)
  [paramName: string]: {
    key: string
    renderTypeList: FlowNodeInputTypeEnum[]
    valueType: WorkflowIOValueTypeEnum
    label: string
    required: boolean
    description?: string
    placeholder?: string
  }
}
```

#### 输出接口定义

```typescript
interface RunPluginOutputs {
  // 动态输出(基于插件配置生成)
  [outputName: string]: {
    id: string
    key: string
    label: string
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum
    description?: string
  }
  
  // 插件执行状态
  pluginStatus: {
    id: 'pluginStatus'
    key: 'pluginStatus'
    label: '执行状态'
    description: '插件执行的状态信息'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.object
  }
  
  // 执行错误
  pluginError: {
    id: 'pluginError'
    key: 'pluginError'
    label: '执行错误'
    description: '插件执行失败时的错误信息'
    type: FlowNodeOutputTypeEnum.error
    valueType: WorkflowIOValueTypeEnum.string
  }
}
```

#### 插件系统架构

```typescript
interface PluginSystemArchitecture {
  // 插件类型
  pluginTypes: {
    teamApp: {
      description: '团队自定义插件',
      characteristics: [
        '团队私有',
        '可自定义输入输出',
        '支持复杂工作流',
        '版本管理'
      ],
      creationMethod: 'workflow_composition'
    },
    
    systemPlugin: {
      description: '系统内置插件',
      characteristics: [
        '官方维护',
        '稳定可靠',
        '性能优化',
        '标准化接口'
      ],
      examples: ['http请求', '数据处理', '文本操作']
    },
    
    communityPlugin: {
      description: '社区贡献插件',
      characteristics: [
        '开源共享',
        '多样化功能',
        '社区维护',
        '审核机制'
      ],
      distributionMethod: 'plugin_marketplace'
    }
  }
  
  // 插件生命周期
  lifecycle: {
    development: '开发阶段',
    testing: '测试验证',
    deployment: '部署发布',
    maintenance: '维护更新',
    deprecation: '废弃下线'
  }
  
  // 插件安全
  security: {
    sandboxExecution: '沙箱隔离执行',
    permissionControl: '权限访问控制',
    resourceLimits: '资源使用限制',
    auditLogging: '审计日志记录'
  }
}
```

#### 插件开发框架

```typescript
interface PluginDevelopmentFramework {
  // 插件配置模板
  pluginTemplate: {
    id: string
    name: string
    description: string
    version: string
    author: string
    
    // 输入配置
    inputs: Array<{
      key: string
      label: string
      description?: string
      valueType: WorkflowIOValueTypeEnum
      required: boolean
      default?: any
      validation?: {
        min?: number
        max?: number
        pattern?: string
        enum?: any[]
      }
    }>
    
    // 输出配置
    outputs: Array<{
      key: string
      label: string
      description?: string
      valueType: WorkflowIOValueTypeEnum
    }>
    
    // 工作流定义
    workflow: FlowNodeType[]
    
    // 元数据
    metadata: {
      tags: string[]
      category: string
      icon: string
      documentation: string
      examples: any[]
    }
  }
  
  // 插件API接口
  pluginApi: {
    // 生命周期钩子
    hooks: {
      beforeExecute?: (context: PluginContext) => Promise<void>
      afterExecute?: (context: PluginContext, result: any) => Promise<any>
      onError?: (context: PluginContext, error: Error) => Promise<void>
    }
    
    // 上下文访问
    context: {
      workflowVariables: Record<string, any>
      userInfo: UserInfo
      teamInfo: TeamInfo
      systemConfig: SystemConfig
    }
    
    // 工具函数
    utilities: {
      http: HttpClient
      database: DatabaseClient
      fileSystem: FileSystemClient
      logger: Logger
    }
  }
}
```

### 3. 工具执行节点 (runTool)

**节点标识**: `FlowNodeTypeEnum.runTool`  
**模板分类**: `FlowNodeTemplateTypeEnum.tools`  
**核心功能**: MCP协议和系统工具的标准化执行

#### 输入接口定义

```typescript
interface RunToolInputs {
  // 工具选择
  toolChoice: {
    key: 'toolChoice'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: '工具选择'
    required: true
    list: [] // 动态加载可用工具列表
  }
  
  // 工具配置模式
  configMode: {
    key: 'configMode'
    renderTypeList: [FlowNodeInputTypeEnum.select]
    valueType: WorkflowIOValueTypeEnum.string
    label: '配置模式'
    value: 'manual'
    list: [
      { label: '手动配置', value: 'manual' },
      { label: '团队配置', value: 'team' },
      { label: '系统配置', value: 'system' }
    ]
  }
  
  // 动态工具参数(基于选择的工具生成)
  [toolParam: string]: {
    key: string
    renderTypeList: FlowNodeInputTypeEnum[]
    valueType: WorkflowIOValueTypeEnum
    label: string
    required: boolean
    description?: string
    validation?: ToolParameterValidation
  }
}

interface ToolParameterValidation {
  type: 'string' | 'number' | 'boolean' | 'array' | 'object'
  format?: string            // 格式约束(email, url, date等)
  minimum?: number           // 最小值
  maximum?: number           // 最大值
  minLength?: number         // 最小长度
  maxLength?: number         // 最大长度
  pattern?: string           // 正则模式
  enum?: any[]              // 枚举值
  properties?: Record<string, ToolParameterValidation> // 对象属性
  items?: ToolParameterValidation // 数组项约束
}
```

#### 输出接口定义

```typescript
interface RunToolOutputs {
  // 工具执行结果
  toolResponse: {
    id: 'toolResponse'
    key: 'toolResponse'
    label: '工具响应'
    description: '工具执行返回的结果'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.any
  }
  
  // 工具元数据
  toolMetadata: {
    id: 'toolMetadata'
    key: 'toolMetadata'
    label: '工具元数据'
    description: '工具执行的元信息'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.object
  }
  
  // 执行统计
  executionStats: {
    id: 'executionStats'
    key: 'executionStats'
    label: '执行统计'
    description: '工具执行的性能统计'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.object
  }
  
  // 工具错误
  toolError: {
    id: 'toolError'
    key: 'toolError'
    label: '工具错误'
    description: '工具执行失败的错误信息'
    type: FlowNodeOutputTypeEnum.error
    valueType: WorkflowIOValueTypeEnum.string
  }
}
```

#### MCP协议集成

```typescript
interface MCPProtocolIntegration {
  // MCP客户端配置
  mcpClient: {
    // 传输协议
    transport: {
      type: 'sse' | 'http',
      endpoint: string,
      headers?: Record<string, string>,
      timeout: number
    }
    
    // 认证配置
    authentication: {
      type: 'none' | 'bearer' | 'apikey' | 'oauth2',
      credentials?: {
        token?: string,
        apiKey?: string,
        clientId?: string,
        clientSecret?: string
      }
    }
    
    // 重试策略
    retryPolicy: {
      maxRetries: number,
      backoffMultiplier: number,
      initialDelay: number,
      maxDelay: number
    }
  }
  
  // 工具发现
  toolDiscovery: {
    // 列出可用工具
    listTools: () => Promise<MCPTool[]>
    
    // 获取工具详情
    getTool: (name: string) => Promise<MCPToolDetail>
    
    // 工具缓存
    cacheConfig: {
      enabled: boolean,
      ttl: number,
      maxSize: number
    }
  }
  
  // 工具执行
  toolExecution: {
    // 执行工具
    execute: (
      toolName: string,
      parameters: Record<string, any>
    ) => Promise<MCPToolResult>
    
    // 流式执行
    executeStream: (
      toolName: string,
      parameters: Record<string, any>
    ) => AsyncIterator<MCPToolPartialResult>
    
    // 执行上下文
    context: {
      workflowId: string,
      nodeId: string,
      userId: string,
      variables: Record<string, any>
    }
  }
}

interface MCPTool {
  name: string
  description: string
  inputSchema: JSONSchema
  outputSchema?: JSONSchema
  category?: string
  tags?: string[]
}

interface MCPToolResult {
  success: boolean
  data?: any
  error?: string
  metadata?: {
    executionTime: number,
    tokensUsed?: number,
    cost?: number
  }
}
```

### 4. 代码执行节点 (code)

**节点标识**: `FlowNodeTypeEnum.code`  
**模板分类**: `FlowNodeTemplateTypeEnum.tools`  
**核心功能**: 沙箱环境中的自定义代码执行

#### 支持的执行环境

```typescript
interface CodeExecutionEnvironments {
  // JavaScript环境
  javascript: {
    runtime: 'Node.js 18+',
    features: [
      'ES2022语法支持',
      '异步/await',
      '模块导入/导出',
      '错误处理'
    ],
    preInstalledModules: [
      'lodash',         // 工具函数库
      'moment',         // 日期处理
      'crypto',         // 加密功能
      'buffer',         // 缓冲区操作
      'querystring',    // 查询字符串处理
      'url'            // URL处理
    ],
    restrictions: {
      networkAccess: false,      // 无网络访问
      fileSystemAccess: false,   // 无文件系统访问
      processAccess: false,      // 无进程访问
      timeout: 30000,           // 30秒超时
      memoryLimit: '128MB'      // 内存限制
    }
  },
  
  // Python环境
  python: {
    runtime: 'Python 3.9+',
    features: [
      '标准库支持',
      '异常处理',
      '类型提示',
      '上下文管理器'
    ],
    preInstalledPackages: [
      'json',           // JSON处理
      'datetime',       // 日期时间
      'math',           // 数学函数
      'random',         // 随机数
      'base64',         // Base64编码
      're',             // 正则表达式
      'hashlib',        // 哈希函数
      'urllib'          // URL处理
    ],
    restrictions: {
      networkAccess: false,
      fileSystemAccess: false,
      subprocessAccess: false,
      timeout: 30000,
      memoryLimit: '128MB'
    }
  }
}
```

#### 代码模板系统

```javascript
// JavaScript代码模板
function main({param1, param2, param3}) {
  // ====== 在这里编写你的代码逻辑 ======
  
  try {
    // 示例：数据处理
    const processedData = {
      input_param1: param1,
      input_param2: param2,
      processed_at: new Date().toISOString()
    };
    
    // 示例：字符串处理
    if (typeof param1 === 'string') {
      processedData.param1_length = param1.length;
      processedData.param1_upper = param1.toUpperCase();
      processedData.param1_words = param1.split(' ').length;
    }
    
    // 示例：数组处理
    if (Array.isArray(param2)) {
      processedData.param2_count = param2.length;
      processedData.param2_sum = param2
        .filter(item => typeof item === 'number')
        .reduce((sum, num) => sum + num, 0);
    }
    
    // 示例：对象处理
    if (typeof param3 === 'object' && param3 !== null) {
      processedData.param3_keys = Object.keys(param3);
      processedData.param3_values = Object.values(param3);
    }
    
    // 返回处理结果
    return {
      success: true,
      result: processedData,
      message: '数据处理完成',
      timestamp: Date.now()
    };
    
  } catch (error) {
    // 错误处理
    return {
      success: false,
      error: error.message,
      result: null,
      timestamp: Date.now()
    };
  }
}
```

```python
# Python代码模板
def main(param1, param2, param3):
    """
    主处理函数
    
    Args:
        param1: 输入参数1
        param2: 输入参数2  
        param3: 输入参数3
        
    Returns:
        dict: 处理结果字典
    """
    import json
    from datetime import datetime
    
    try:
        # 初始化结果字典
        processed_data = {
            'input_param1': param1,
            'input_param2': param2,
            'processed_at': datetime.now().isoformat()
        }
        
        # 字符串处理示例
        if isinstance(param1, str):
            processed_data['param1_length'] = len(param1)
            processed_data['param1_words'] = len(param1.split())
            processed_data['param1_upper'] = param1.upper()
        
        # 列表处理示例
        if isinstance(param2, list):
            processed_data['param2_count'] = len(param2)
            # 计算数值元素的和
            numeric_items = [x for x in param2 if isinstance(x, (int, float))]
            processed_data['param2_numeric_sum'] = sum(numeric_items)
        
        # 字典处理示例
        if isinstance(param3, dict):
            processed_data['param3_keys'] = list(param3.keys())
            processed_data['param3_values'] = list(param3.values())
        
        return {
            'success': True,
            'result': processed_data,
            'message': '数据处理完成',
            'timestamp': datetime.now().timestamp()
        }
        
    except Exception as e:
        # 异常处理
        return {
            'success': False,
            'error': str(e),
            'result': None,
            'timestamp': datetime.now().timestamp()
        }
```

### 5. LAF云函数节点 (lafModule)

**节点标识**: `FlowNodeTypeEnum.lafModule`  
**模板分类**: `FlowNodeTemplateTypeEnum.tools`  
**核心功能**: LAF云函数平台集成

#### 输入接口定义

```typescript
interface LAFModuleInputs {
  // LAF函数URL
  url: {
    key: 'url'
    renderTypeList: [FlowNodeInputTypeEnum.input, FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.string
    label: 'LAF函数地址'
    placeholder: 'https://your-app.laf.run/function-name'
    required: true
  }
  
  // 动态参数(基于函数签名生成)
  [paramName: string]: {
    key: string
    renderTypeList: [FlowNodeInputTypeEnum.reference, FlowNodeInputTypeEnum.input]
    valueType: WorkflowIOValueTypeEnum
    label: string
    required: boolean
    description?: string
  }
}
```

#### LAF集成特性

```typescript
interface LAFIntegrationFeatures {
  // 函数发现
  functionDiscovery: {
    // 自动识别函数参数
    parseParameters: (functionUrl: string) => Promise<LAFFunctionSignature>
    
    // 函数元数据获取
    getMetadata: (functionUrl: string) => Promise<LAFFunctionMetadata>
  }
  
  // 执行环境
  executionEnvironment: {
    serverless: true,
    autoScaling: true,
    coldStart: 'optimized',
    runtime: 'Node.js',
    timeout: 'configurable'
  }
  
  // 数据传输
  dataTransport: {
    method: 'POST',
    format: 'JSON',
    encoding: 'UTF-8',
    compression: 'gzip'
  }
}

interface LAFFunctionSignature {
  name: string
  parameters: Array<{
    name: string
    type: string
    required: boolean
    description?: string
    default?: any
  }>
  returnType: string
  documentation?: string
}
```

### 6. 文件读取节点 (readFiles)

**节点标识**: `FlowNodeTypeEnum.readFiles`  
**模板分类**: `FlowNodeTemplateTypeEnum.tools`  
**核心功能**: 外部文件内容读取和处理

#### 支持的文件格式

```typescript
interface SupportedFileFormats {
  // 文档格式
  documents: {
    pdf: {
      extensions: ['.pdf'],
      features: ['文本提取', 'OCR识别', '表格解析', '图像提取'],
      maxSize: '100MB',
      encoding: 'UTF-8'
    },
    
    office: {
      extensions: ['.docx', '.xlsx', '.pptx'],
      features: ['内容提取', '格式保持', '表格数据', '元数据读取'],
      maxSize: '50MB',
      encoding: 'UTF-8'
    },
    
    text: {
      extensions: ['.txt', '.md', '.csv', '.json', '.xml', '.yaml'],
      features: ['编码检测', '格式解析', '结构化提取'],
      maxSize: '10MB',
      encoding: 'auto-detect'
    }
  },
  
  // 代码文件
  code: {
    extensions: ['.js', '.ts', '.py', '.java', '.cpp', '.go', '.rs'],
    features: ['语法高亮', '注释提取', '结构分析'],
    maxSize: '5MB',
    encoding: 'UTF-8'
  },
  
  // 配置文件
  config: {
    extensions: ['.ini', '.conf', '.toml', '.properties'],
    features: ['键值解析', '节段识别', '格式转换'],
    maxSize: '1MB',
    encoding: 'UTF-8'
  },
  
  // 图像文件
  images: {
    extensions: ['.jpg', '.jpeg', '.png', '.gif', '.bmp', '.webp'],
    features: ['OCR文字识别', '元数据提取', '尺寸信息'],
    maxSize: '20MB',
    encoding: 'binary'
  }
}
```

## 🎯 集成架构模式

### 1. API聚合模式

```
多个API源 → HTTP请求节点 → 数据标准化 → 结果合并 → 统一输出
```

**适用场景**:
- 多数据源整合
- 价格比较系统
- 信息聚合服务
- 跨平台数据同步

### 2. 插件生态模式

```
核心工作流 → 插件节点A → 插件节点B → 插件节点C → 最终结果
```

**适用场景**:
- 模块化业务逻辑
- 可扩展的处理管道
- 团队协作开发
- 功能复用和组合

### 3. 混合集成模式

```
工作流开始 → API调用 → 代码处理 → 工具执行 → 结果输出
```

**适用场景**:
- 复杂业务场景
- 多技术栈集成
- 灵活的处理流程
- 自定义业务逻辑

### 4. 实时流处理模式

```
外部事件 → Webhook接收 → 即时处理 → 结果推送 → 状态更新
```

**适用场景**:
- 实时通知系统
- 事件驱动架构
- 监控和告警
- 自动化响应

## 🔒 安全与治理

### 安全控制机制

```typescript
interface SecurityControlMechanisms {
  // 访问控制
  accessControl: {
    authentication: '身份认证',
    authorization: '权限授权',
    rateLimiting: '频率限制',
    ipWhitelist: 'IP白名单'
  }
  
  // 数据保护
  dataProtection: {
    encryption: '传输加密',
    tokenization: '敏感数据标记化',
    dataMinimization: '数据最小化原则',
    retentionPolicy: '数据保留政策'
  }
  
  // 执行隔离
  executionIsolation: {
    sandboxing: '沙箱隔离',
    resourceLimits: '资源限制',
    networkRestrictions: '网络访问限制',
    timeoutControls: '超时控制'
  }
  
  // 审计监控
  auditMonitoring: {
    accessLogging: '访问日志',
    performanceMetrics: '性能指标',
    errorTracking: '错误跟踪',
    complianceReporting: '合规报告'
  }
}
```

### 治理框架

```typescript
interface GovernanceFramework {
  // API管理
  apiManagement: {
    versionControl: 'API版本控制',
    lifecycleManagement: 'API生命周期管理',
    documentationStandards: 'API文档标准',
    testingRequirements: 'API测试要求'
  }
  
  // 质量保证
  qualityAssurance: {
    codeReview: '代码审查',
    securityScanning: '安全扫描',
    performanceTesting: '性能测试',
    reliabilityMetrics: '可靠性指标'
  }
  
  // 运维管理
  operationsManagement: {
    deploymentPipeline: '部署流水线',
    monitoringDashboard: '监控仪表板',
    incidentResponse: '事件响应',
    capacityPlanning: '容量规划'
  }
}
```

---

FastGPT 的外部集成节点体系构建了一个开放、安全、高效的集成平台。通过HTTP API、插件系统、MCP协议、代码执行等多种集成方式，用户可以将FastGPT与任何外部服务或系统进行无缝连接，实现真正的"无边界"工作流自动化。这种强大的集成能力，使得FastGPT不仅仅是一个AI工作流平台，更是一个完整的企业级集成和自动化解决方案。