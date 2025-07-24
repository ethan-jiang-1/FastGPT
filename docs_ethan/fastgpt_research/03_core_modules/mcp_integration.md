# FastGPT MCP 集成深度分析

## 🔌 MCP 集成概述

FastGPT 是业内**率先深度集成 Model Context Protocol (MCP)** 的 AI 平台之一。MCP 是由 Anthropic 制定的开放标准，旨在让 AI 模型能够安全、标准化地访问外部工具和数据源。FastGPT 的 MCP 集成为用户提供了**无缝的工具扩展能力**和**标准化的服务接入方式**。

### MCP 协议核心价值

- **标准化工具接入** - 统一的工具定义和调用规范
- **安全的资源访问** - 受控的外部资源访问机制
- **双向通信支持** - 客户端和服务端的双向数据交换
- **生态系统建设** - 促进AI工具生态的标准化发展
- **开发者友好** - 简化工具开发和集成流程

## 🏗️ MCP 集成架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP 集成架构                              │
├─────────────────────────────────────────────────────────────┤
│  FastGPT 工作流层 (Workflow Layer)                           │
│  ├── MCP 工具节点 (MCP Tool Nodes)                          │
│  ├── 工具调用器 (Tool Invoker)                              │
│  ├── 参数映射器 (Parameter Mapper)                          │
│  └── 结果处理器 (Result Processor)                          │
├─────────────────────────────────────────────────────────────┤
│  MCP 客户端层 (MCP Client Layer)                            │
│  ├── 连接管理器 (Connection Manager)                        │
│  ├── 协议处理器 (Protocol Handler)                          │
│  ├── 消息路由器 (Message Router)                            │
│  └── 错误处理器 (Error Handler)                             │
├─────────────────────────────────────────────────────────────┤
│  MCP 服务层 (MCP Service Layer)                             │
│  ├── 工具注册中心 (Tool Registry)                           │
│  ├── 资源管理器 (Resource Manager)                          │
│  ├── 模板引擎 (Template Engine)                             │
│  └── 会话管理器 (Session Manager)                           │
├─────────────────────────────────────────────────────────────┤
│  外部服务层 (External Services Layer)                        │
│  ├── MCP 服务器实例 (MCP Server Instances)                  │
│  ├── 第三方工具服务 (Third-party Tool Services)             │
│  ├── 数据源连接器 (Data Source Connectors)                  │
│  └── API 适配器 (API Adapters)                              │
└─────────────────────────────────────────────────────────────┘
```

## 🔧 MCP 客户端实现

### MCP 客户端核心

**MCP 客户端管理器** (`projects/app/src/service/core/app/mcp.ts`)
```typescript
import { Client } from '@modelcontextprotocol/sdk/client/index.js'
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'
import { CallToolRequestSchema, ListToolsRequestSchema } from '@modelcontextprotocol/sdk/types.js'

export class MCPClientManager {
  private clients = new Map<string, MCPClient>()
  private connections = new Map<string, MCPConnection>()
  
  // 创建 MCP 客户端连接
  async createClient(params: {
    serverId: string
    serverConfig: MCPServerConfig
    userId: string
  }): Promise<MCPClient> {
    
    const { serverId, serverConfig, userId } = params
    
    // 检查是否已存在连接
    const existingClient = this.clients.get(serverId)
    if (existingClient && existingClient.isConnected()) {
      return existingClient
    }
    
    try {
      // 创建传输层
      const transport = new StdioClientTransport({
        command: serverConfig.command,
        args: serverConfig.args || [],
        env: {
          ...process.env,
          ...serverConfig.env
        }
      })
      
      // 创建 MCP 客户端
      const client = new Client({
        name: 'FastGPT',
        version: '1.0.0'
      }, {
        capabilities: {
          tools: {},
          resources: {},
          prompts: {}
        }
      })
      
      // 连接到服务器
      await client.connect(transport)
      
      // 创建包装客户端
      const mcpClient = new MCPClient({
        serverId,
        client,
        transport,
        config: serverConfig,
        userId
      })
      
      // 初始化客户端
      await mcpClient.initialize()
      
      // 缓存客户端
      this.clients.set(serverId, mcpClient)
      
      // 设置错误处理
      this.setupErrorHandling(mcpClient)
      
      return mcpClient
      
    } catch (error) {
      console.error(`MCP客户端创建失败 [${serverId}]:`, error)
      throw new Error(`无法连接到MCP服务器: ${error.message}`)
    }
  }
  
  // 获取可用工具列表
  async listTools(serverId: string): Promise<ToolDefinition[]> {
    const client = await this.getClient(serverId)
    
    try {
      const response = await client.client.request(
        { method: 'tools/list' },
        ListToolsRequestSchema
      )
      
      return response.tools.map(tool => ({
        name: tool.name,
        description: tool.description,
        inputSchema: tool.inputSchema,
        metadata: {
          serverId,
          version: tool.version || '1.0.0',
          category: tool.category || 'general'
        }
      }))
      
    } catch (error) {
      console.error(`获取工具列表失败 [${serverId}]:`, error)
      throw error
    }
  }
  
  // 调用工具
  async callTool(params: {
    serverId: string
    toolName: string
    arguments: Record<string, any>
    context?: MCPCallContext
  }): Promise<MCPToolResult> {
    
    const { serverId, toolName, arguments: toolArgs, context } = params
    const client = await this.getClient(serverId)
    
    try {
      // 记录工具调用
      await this.logToolCall({
        serverId,
        toolName,
        arguments: toolArgs,
        userId: client.userId,
        timestamp: new Date()
      })
      
      // 执行工具调用
      const startTime = Date.now()
      const response = await client.client.request(
        {
          method: 'tools/call',
          params: {
            name: toolName,
            arguments: toolArgs
          }
        },
        CallToolRequestSchema
      )
      
      const duration = Date.now() - startTime
      
      // 处理响应
      const result = this.processToolResponse(response)
      
      // 记录结果
      await this.logToolResult({
        serverId,
        toolName,
        arguments: toolArgs,
        result,
        duration,
        success: true,
        userId: client.userId
      })
      
      return result
      
    } catch (error) {
      console.error(`工具调用失败 [${serverId}:${toolName}]:`, error)
      
      // 记录错误
      await this.logToolResult({
        serverId,
        toolName,
        arguments: toolArgs,
        result: null,
        duration: 0,
        success: false,
        error: error.message,
        userId: client.userId
      })
      
      throw error
    }
  }
  
  // 获取资源
  async getResource(params: {
    serverId: string
    uri: string
    mimeType?: string
  }): Promise<MCPResource> {
    
    const { serverId, uri, mimeType } = params
    const client = await this.getClient(serverId)
    
    try {
      const response = await client.client.request({
        method: 'resources/read',
        params: { uri }
      })
      
      return {
        uri,
        content: response.contents[0]?.text || response.contents[0]?.blob,
        mimeType: mimeType || response.contents[0]?.mimeType,
        metadata: response.metadata
      }
      
    } catch (error) {
      console.error(`获取资源失败 [${serverId}:${uri}]:`, error)
      throw error
    }
  }
  
  // 渲染提示模板
  async renderPrompt(params: {
    serverId: string
    promptName: string
    arguments: Record<string, any>
  }): Promise<string> {
    
    const { serverId, promptName, arguments: promptArgs } = params
    const client = await this.getClient(serverId)
    
    try {
      const response = await client.client.request({
        method: 'prompts/get',
        params: {
          name: promptName,
          arguments: promptArgs
        }
      })
      
      // 合并消息内容
      return response.messages
        .map(msg => msg.content.text)
        .join('\n')
        
    } catch (error) {
      console.error(`渲染提示失败 [${serverId}:${promptName}]:`, error)
      throw error
    }
  }
  
  // 处理工具响应
  private processToolResponse(response: any): MCPToolResult {
    const content = response.content || response.result
    
    if (!content || content.length === 0) {
      return {
        type: 'text',
        content: '工具执行完成，无返回内容'
      }
    }
    
    const firstContent = content[0]
    
    switch (firstContent.type) {
      case 'text':
        return {
          type: 'text',
          content: firstContent.text
        }
        
      case 'image':
        return {
          type: 'image',
          content: firstContent.data,
          mimeType: firstContent.mimeType
        }
        
      case 'resource':
        return {
          type: 'resource',
          content: firstContent.resource,
          metadata: firstContent.metadata
        }
        
      default:
        return {
          type: 'text',
          content: JSON.stringify(firstContent, null, 2)
        }
    }
  }
  
  // 获取客户端
  private async getClient(serverId: string): Promise<MCPClient> {
    const client = this.clients.get(serverId)
    
    if (!client) {
      throw new Error(`MCP客户端不存在: ${serverId}`)
    }
    
    if (!client.isConnected()) {
      throw new Error(`MCP客户端未连接: ${serverId}`)
    }
    
    return client
  }
  
  // 设置错误处理
  private setupErrorHandling(client: MCPClient): void {
    client.transport.onerror = (error) => {
      console.error(`MCP传输错误 [${client.serverId}]:`, error)
      this.handleClientError(client.serverId, error)
    }
    
    client.transport.onclose = () => {
      console.log(`MCP连接关闭 [${client.serverId}]`)
      this.clients.delete(client.serverId)
    }
  }
  
  // 处理客户端错误
  private async handleClientError(serverId: string, error: any): Promise<void> {
    // 移除失效的客户端
    this.clients.delete(serverId)
    
    // 尝试重连
    try {
      const serverConfig = await this.getServerConfig(serverId)
      if (serverConfig) {
        setTimeout(async () => {
          try {
            await this.createClient({
              serverId,
              serverConfig,
              userId: 'system'
            })
            console.log(`MCP客户端重连成功 [${serverId}]`)
          } catch (reconnectError) {
            console.error(`MCP客户端重连失败 [${serverId}]:`, reconnectError)
          }
        }, 5000) // 5秒后重试
      }
    } catch (error) {
      console.error(`获取服务器配置失败 [${serverId}]:`, error)
    }
  }
}
```

### MCP 工具节点集成

**工作流中的 MCP 工具节点** (`packages/service/core/workflow/dispatch/tools/runMCPTool.ts`)
```typescript
export const dispatchMCPTool = async (params: {
  node: FlowNodeItemType
  runtimeNodes: RuntimeNodeItemType[]
  variables: Record<string, any>
  context: WorkflowContext
}): Promise<DispatchNodeResponse> => {
  
  const { node, variables, context } = params
  
  try {
    // 获取节点配置
    const {
      serverId,
      toolName,
      toolArguments,
      outputFormat = 'text',
      errorHandling = 'throw'
    } = await getHandleConfig(params)
    
    // 验证必需参数
    if (!serverId || !toolName) {
      throw new Error('MCP服务器ID和工具名称不能为空')
    }
    
    // 解析工具参数
    const resolvedArgs = await resolveToolArguments(toolArguments, variables)
    
    // 获取 MCP 客户端管理器
    const mcpManager = MCPClientManager.getInstance()
    
    // 调用 MCP 工具
    const result = await mcpManager.callTool({
      serverId,
      toolName,
      arguments: resolvedArgs,
      context: {
        userId: context.userId,
        sessionId: context.sessionId,
        workflowId: context.workflowId,
        nodeId: node.nodeId
      }
    })
    
    // 格式化输出
    const formattedResult = await formatToolResult(result, outputFormat)
    
    // 更新统计信息
    context.statistics.toolCalls = (context.statistics.toolCalls || 0) + 1
    
    return {
      [NodeOutputKeyEnum.toolResult]: formattedResult,
      [NodeOutputKeyEnum.toolRawResult]: result,
      [NodeOutputKeyEnum.toolMetadata]: {
        serverId,
        toolName,
        arguments: resolvedArgs,
        executionTime: Date.now() - context.startTime
      },
      finish: true
    }
    
  } catch (error) {
    console.error('MCP工具调用失败:', error)
    
    // 根据错误处理策略处理
    const errorHandling = node.inputs.find(i => i.key === 'errorHandling')?.value || 'throw'
    
    switch (errorHandling) {
      case 'ignore':
        return {
          [NodeOutputKeyEnum.toolResult]: '',
          [NodeOutputKeyEnum.error]: error.message,
          finish: true
        }
        
      case 'fallback':
        const fallbackValue = node.inputs.find(i => i.key === 'fallbackValue')?.value || ''
        return {
          [NodeOutputKeyEnum.toolResult]: fallbackValue,
          [NodeOutputKeyEnum.error]: error.message,
          finish: true
        }
        
      case 'throw':
      default:
        throw new WorkflowExecutionError(
          `MCP工具调用失败: ${error.message}`,
          node.nodeId,
          'mcpTool',
          error
        )
    }
  }
}

// 解析工具参数
async function resolveToolArguments(
  argumentsConfig: any,
  variables: Record<string, any>
): Promise<Record<string, any>> {
  
  if (typeof argumentsConfig === 'string') {
    // JSON 字符串形式
    try {
      const parsed = JSON.parse(argumentsConfig)
      return resolveVariableReferences(parsed, variables)
    } catch (error) {
      throw new Error(`工具参数JSON解析失败: ${error.message}`)
    }
  } else if (typeof argumentsConfig === 'object') {
    // 对象形式
    return resolveVariableReferences(argumentsConfig, variables)
  } else {
    return {}
  }
}

// 解析变量引用
function resolveVariableReferences(
  obj: any,
  variables: Record<string, any>
): any {
  
  if (typeof obj === 'string') {
    // 替换变量引用 {{variableName}}
    return obj.replace(/\{\{([^}]+)\}\}/g, (match, varName) => {
      const value = variables[varName.trim()]
      return value !== undefined ? value : match
    })
  } else if (Array.isArray(obj)) {
    return obj.map(item => resolveVariableReferences(item, variables))
  } else if (typeof obj === 'object' && obj !== null) {
    const resolved: any = {}
    for (const [key, value] of Object.entries(obj)) {
      resolved[key] = resolveVariableReferences(value, variables)
    }
    return resolved
  } else {
    return obj
  }
}

// 格式化工具结果
async function formatToolResult(
  result: MCPToolResult,
  outputFormat: string
): Promise<string> {
  
  switch (outputFormat) {
    case 'json':
      return JSON.stringify(result, null, 2)
      
    case 'text':
      if (result.type === 'text') {
        return result.content
      } else if (result.type === 'image') {
        return `[图片: ${result.mimeType}]`
      } else if (result.type === 'resource') {
        return `[资源: ${result.content}]`
      } else {
        return JSON.stringify(result.content)
      }
      
    case 'markdown':
      if (result.type === 'text') {
        return `\`\`\`\n${result.content}\n\`\`\``
      } else {
        return `\`\`\`json\n${JSON.stringify(result, null, 2)}\n\`\`\``
      }
      
    default:
      return result.content?.toString() || ''
  }
}
```

## 🛠️ MCP 服务器实现

### 内置 MCP 服务器

**FastGPT MCP 服务器** (`projects/mcp_server/src/index.ts`)
```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import {
  CallToolRequestSchema,
  CallToolRequest,
  ListToolsRequestSchema,
  Tool
} from '@modelcontextprotocol/sdk/types.js'

class FastGPTMCPServer {
  private server: Server
  private fastgptApi: FastGPTAPI
  
  constructor() {
    // 创建服务器实例
    this.server = new Server(
      {
        name: 'fastgpt-mcp-server',
        version: '1.0.0'
      },
      {
        capabilities: {
          tools: {},
          resources: {},
          prompts: {}
        }
      }
    )
    
    // 初始化 FastGPT API 客户端
    this.fastgptApi = new FastGPTAPI({
      baseURL: process.env.FASTGPT_BASE_URL,
      apiKey: process.env.FASTGPT_API_KEY
    })
    
    this.setupHandlers()
  }
  
  private setupHandlers(): void {
    // 工具列表处理器
    this.server.setRequestHandler(ListToolsRequestSchema, async () => {
      return {
        tools: [
          {
            name: 'search_dataset',
            description: '在FastGPT知识库中搜索相关内容',
            inputSchema: {
              type: 'object',
              properties: {
                query: {
                  type: 'string',
                  description: '搜索查询'
                },
                datasetId: {
                  type: 'string',
                  description: '知识库ID'
                },
                limit: {
                  type: 'number',
                  description: '返回结果数量限制',
                  default: 5
                }
              },
              required: ['query', 'datasetId']
            }
          },
          {
            name: 'create_app',
            description: '创建新的FastGPT应用',
            inputSchema: {
              type: 'object',
              properties: {
                name: {
                  type: 'string',
                  description: '应用名称'
                },
                intro: {
                  type: 'string',
                  description: '应用描述'
                },
                type: {
                  type: 'string',
                  enum: ['simple', 'workflow'],
                  description: '应用类型'
                }
              },
              required: ['name', 'type']
            }
          },
          {
            name: 'get_chat_history',
            description: '获取对话历史记录',
            inputSchema: {
              type: 'object',
              properties: {
                chatId: {
                  type: 'string',
                  description: '对话ID'
                },
                limit: {
                  type: 'number',
                  description: '返回消息数量',
                  default: 20
                }
              },
              required: ['chatId']
            }
          },
          {
            name: 'list_apps',
            description: '列出用户的所有应用',
            inputSchema: {
              type: 'object',
              properties: {
                page: {
                  type: 'number',
                  description: '页码',
                  default: 1
                },
                pageSize: {
                  type: 'number',
                  description: '每页大小',
                  default: 20
                }
              }
            }
          }
        ]
      }
    })
    
    // 工具调用处理器
    this.server.setRequestHandler(CallToolRequestSchema, async (request: CallToolRequest) => {
      const { name, arguments: args } = request.params
      
      try {
        switch (name) {
          case 'search_dataset':
            return await this.handleSearchDataset(args)
            
          case 'create_app':
            return await this.handleCreateApp(args)
            
          case 'get_chat_history':
            return await this.handleGetChatHistory(args)
            
          case 'list_apps':
            return await this.handleListApps(args)
            
          default:
            throw new Error(`未知工具: ${name}`)
        }
      } catch (error) {
        return {
          content: [
            {
              type: 'text',
              text: `工具执行失败: ${error.message}`
            }
          ],
          isError: true
        }
      }
    })
  }
  
  // 处理知识库搜索
  private async handleSearchDataset(args: any) {
    const { query, datasetId, limit = 5 } = args
    
    try {
      const results = await this.fastgptApi.searchDataset({
        datasetId,
        query,
        limit
      })
      
      const formattedResults = results.map((item, index) => 
        `${index + 1}. ${item.q}\n${item.a}\n(相似度: ${item.score})`
      ).join('\n\n')
      
      return {
        content: [
          {
            type: 'text',
            text: `在知识库中找到 ${results.length} 条相关内容:\n\n${formattedResults}`
          }
        ]
      }
    } catch (error) {
      throw new Error(`知识库搜索失败: ${error.message}`)
    }
  }
  
  // 处理应用创建
  private async handleCreateApp(args: any) {
    const { name, intro, type } = args
    
    try {
      const app = await this.fastgptApi.createApp({
        name,
        intro,
        type
      })
      
      return {
        content: [
          {
            type: 'text',
            text: `应用创建成功!\n\n应用ID: ${app.id}\n应用名称: ${app.name}\n应用类型: ${app.type}\n创建时间: ${app.createTime}`
          }
        ]
      }
    } catch (error) {
      throw new Error(`应用创建失败: ${error.message}`)
    }
  }
  
  // 处理获取对话历史
  private async handleGetChatHistory(args: any) {
    const { chatId, limit = 20 } = args
    
    try {
      const history = await this.fastgptApi.getChatHistory({
        chatId,
        limit
      })
      
      const formattedHistory = history.map(item => 
        `${item.obj}: ${item.value}\n时间: ${item.time}`
      ).join('\n\n')
      
      return {
        content: [
          {
            type: 'text',
            text: `对话历史 (${history.length} 条消息):\n\n${formattedHistory}`
          }
        ]
      }
    } catch (error) {
      throw new Error(`获取对话历史失败: ${error.message}`)
    }
  }
  
  // 处理应用列表
  private async handleListApps(args: any) {
    const { page = 1, pageSize = 20 } = args
    
    try {
      const { apps, total } = await this.fastgptApi.listApps({
        page,
        pageSize
      })
      
      const formattedApps = apps.map(app => 
        `- ${app.name} (ID: ${app.id})\n  类型: ${app.type}\n  描述: ${app.intro || '无'}`
      ).join('\n\n')
      
      return {
        content: [
          {
            type: 'text',
            text: `用户应用列表 (共 ${total} 个):\n\n${formattedApps}`
          }
        ]
      }
    } catch (error) {
      throw new Error(`获取应用列表失败: ${error.message}`)
    }
  }
  
  // 启动服务器
  async start(): Promise<void> {
    const transport = new StdioServerTransport()
    await this.server.connect(transport)
    console.log('FastGPT MCP服务器已启动')
  }
}

// 启动服务器
async function main() {
  const server = new FastGPTMCPServer()
  await server.start()
}

if (import.meta.url === `file://${process.argv[1]}`) {
  main().catch(console.error)
}
```

## 🔗 MCP 工具生态

### 官方工具集

**内置工具定义** (`packages/global/support/mcp/type.d.ts`)
```typescript
// MCP 工具分类
export enum MCPToolCategoryEnum {
  dataAccess = 'data_access',        // 数据访问
  computation = 'computation',       // 计算处理
  communication = 'communication',   // 通信交互
  fileOperation = 'file_operation',  // 文件操作
  webService = 'web_service',        // 网络服务
  automation = 'automation',         // 自动化工具
  analysis = 'analysis',             // 分析工具
  integration = 'integration'        // 集成工具
}

// 官方工具库
export const OfficialMCPTools: MCPToolDefinition[] = [
  {
    id: 'fastgpt-core',
    name: 'FastGPT 核心工具',
    description: '访问FastGPT核心功能的官方工具集',
    category: MCPToolCategoryEnum.dataAccess,
    serverConfig: {
      command: 'node',
      args: ['./projects/mcp_server/dist/index.js'],
      env: {
        FASTGPT_BASE_URL: 'http://localhost:3000',
        FASTGPT_API_KEY: '{{userApiKey}}'
      }
    },
    tools: [
      'search_dataset',
      'create_app',
      'get_chat_history',
      'list_apps'
    ],
    version: '1.0.0',
    official: true
  },
  {
    id: 'web-search',
    name: '网络搜索工具',
    description: '通过搜索引擎获取实时信息',
    category: MCPToolCategoryEnum.webService,
    serverConfig: {
      command: 'npx',
      args: ['@modelcontextprotocol/server-web-search'],
      env: {
        SEARCH_API_KEY: '{{searchApiKey}}'
      }
    },
    tools: [
      'web_search',
      'get_page_content'
    ],
    version: '1.0.0',
    official: true
  },
  {
    id: 'filesystem',
    name: '文件系统工具',
    description: '安全的文件系统访问工具',
    category: MCPToolCategoryEnum.fileOperation,
    serverConfig: {
      command: 'npx',
      args: ['@modelcontextprotocol/server-filesystem', '/safe/directory'],
      env: {}
    },
    tools: [
      'read_file',
      'write_file',
      'list_directory',
      'create_directory'
    ],
    version: '1.0.0',
    official: true
  },
  {
    id: 'database',
    name: '数据库工具',
    description: '安全的数据库查询工具',
    category: MCPToolCategoryEnum.dataAccess,
    serverConfig: {
      command: 'npx',
      args: ['@modelcontextprotocol/server-database'],
      env: {
        DATABASE_URL: '{{databaseUrl}}'
      }
    },
    tools: [
      'query_database',
      'list_tables',
      'describe_table'
    ],
    version: '1.0.0',
    official: true
  }
]
```

### 第三方工具集成

**工具商店管理** (`packages/service/support/mcp/schema.ts`)
```typescript
const MCPServerSchema = new Schema({
  userId: {
    type: Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'Team',
    required: true
  },
  name: {
    type: String,
    required: true
  },
  description: {
    type: String,
    default: ''
  },
  category: {
    type: String,
    enum: Object.values(MCPToolCategoryEnum),
    default: MCPToolCategoryEnum.integration
  },
  serverConfig: {
    command: {
      type: String,
      required: true
    },
    args: [{
      type: String
    }],
    env: {
      type: Map,
      of: String,
      default: {}
    },
    workingDirectory: {
      type: String
    },
    timeout: {
      type: Number,
      default: 30000
    }
  },
  permissions: {
    allowedTools: [{
      type: String
    }],
    allowedResources: [{
      type: String
    }],
    rateLimits: {
      requestsPerMinute: {
        type: Number,
        default: 60
      },
      requestsPerHour: {
        type: Number,
        default: 1000
      }
    }
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'error'],
    default: 'inactive'
  },
  lastError: {
    type: String
  },
  statistics: {
    totalCalls: {
      type: Number,
      default: 0
    },
    successfulCalls: {
      type: Number,
      default: 0
    },
    failedCalls: {
      type: Number,
      default: 0
    },
    avgResponseTime: {
      type: Number,
      default: 0
    },
    lastUsed: {
      type: Date
    }
  }
}, {
  timestamps: true
})

// MCP工具使用日志
const MCPToolLogSchema = new Schema({
  userId: {
    type: Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  serverId: {
    type: Schema.Types.ObjectId,
    ref: 'MCPServer',
    required: true
  },
  toolName: {
    type: String,
    required: true
  },
  arguments: {
    type: Schema.Types.Mixed
  },
  result: {
    type: Schema.Types.Mixed
  },
  success: {
    type: Boolean,
    required: true
  },
  error: {
    type: String
  },
  duration: {
    type: Number,
    required: true
  },
  metadata: {
    workflowId: String,
    nodeId: String,
    sessionId: String
  }
}, {
  timestamps: true
})
```

## 📊 MCP 监控与管理

### 工具使用统计

**MCP 分析服务** (`packages/service/support/mcp/analytics.ts`)
```typescript
export class MCPAnalyticsService {
  
  // 生成工具使用报告
  async generateUsageReport(params: {
    userId?: string
    teamId?: string
    startDate: Date
    endDate: Date
    serverId?: string
  }): Promise<MCPUsageReport> {
    
    const { userId, teamId, startDate, endDate, serverId } = params
    
    // 构建查询条件
    const matchConditions: any = {
      createdAt: { $gte: startDate, $lte: endDate }
    }
    
    if (userId) matchConditions.userId = userId
    if (teamId) matchConditions.teamId = teamId
    if (serverId) matchConditions.serverId = serverId
    
    // 聚合统计
    const [toolStats, serverStats, errorStats] = await Promise.all([
      this.getToolStatistics(matchConditions),
      this.getServerStatistics(matchConditions),
      this.getErrorStatistics(matchConditions)
    ])
    
    return {
      period: { startDate, endDate },
      summary: {
        totalCalls: toolStats.reduce((sum, stat) => sum + stat.totalCalls, 0),
        successfulCalls: toolStats.reduce((sum, stat) => sum + stat.successfulCalls, 0),
        failedCalls: toolStats.reduce((sum, stat) => sum + stat.failedCalls, 0),
        averageResponseTime: this.calculateAverageResponseTime(toolStats),
        uniqueTools: toolStats.length,
        uniqueServers: serverStats.length
      },
      toolStatistics: toolStats,
      serverStatistics: serverStats,
      errorAnalysis: errorStats,
      recommendations: await this.generateRecommendations(toolStats, errorStats)
    }
  }
  
  // 获取工具统计
  private async getToolStatistics(matchConditions: any) {
    return await MCPToolLogModel.aggregate([
      { $match: matchConditions },
      {
        $group: {
          _id: {
            serverId: '$serverId',
            toolName: '$toolName'
          },
          totalCalls: { $sum: 1 },
          successfulCalls: { $sum: { $cond: ['$success', 1, 0] } },
          failedCalls: { $sum: { $cond: ['$success', 0, 1] } },
          avgResponseTime: { $avg: '$duration' },
          minResponseTime: { $min: '$duration' },
          maxResponseTime: { $max: '$duration' },
          lastUsed: { $max: '$createdAt' }
        }
      },
      {
        $lookup: {
          from: 'mcpservers',
          localField: '_id.serverId',
          foreignField: '_id',
          as: 'server'
        }
      },
      {
        $project: {
          serverId: '$_id.serverId',
          serverName: { $arrayElemAt: ['$server.name', 0] },
          toolName: '$_id.toolName',
          totalCalls: 1,
          successfulCalls: 1,
          failedCalls: 1,
          successRate: { $divide: ['$successfulCalls', '$totalCalls'] },
          avgResponseTime: 1,
          minResponseTime: 1,
          maxResponseTime: 1,
          lastUsed: 1
        }
      },
      { $sort: { totalCalls: -1 } }
    ])
  }
  
  // 性能监控
  async monitorPerformance(): Promise<MCPPerformanceMetrics> {
    const now = new Date()
    const oneHourAgo = new Date(now.getTime() - 60 * 60 * 1000)
    
    // 获取最近一小时的性能数据
    const recentLogs = await MCPToolLogModel.find({
      createdAt: { $gte: oneHourAgo }
    })
    
    // 计算性能指标
    const totalCalls = recentLogs.length
    const successfulCalls = recentLogs.filter(log => log.success).length
    const failedCalls = totalCalls - successfulCalls
    
    const responseTimes = recentLogs.map(log => log.duration)
    const avgResponseTime = responseTimes.reduce((sum, time) => sum + time, 0) / totalCalls
    
    // P95 响应时间
    responseTimes.sort((a, b) => a - b)
    const p95Index = Math.floor(responseTimes.length * 0.95)
    const p95ResponseTime = responseTimes[p95Index] || 0
    
    // 错误率趋势
    const errorRate = failedCalls / totalCalls
    const isHighErrorRate = errorRate > 0.05 // 5% 错误率阈值
    
    // 慢查询检测
    const slowCalls = recentLogs.filter(log => log.duration > 10000) // 10秒以上
    
    return {
      timestamp: now,
      totalCalls,
      successfulCalls,
      failedCalls,
      successRate: successfulCalls / totalCalls,
      avgResponseTime,
      p95ResponseTime,
      errorRate,
      alerts: [
        ...(isHighErrorRate ? [{
          type: 'high_error_rate',
          severity: 'warning',
          message: `错误率过高: ${(errorRate * 100).toFixed(2)}%`,
          threshold: '5%'
        }] : []),
        ...(slowCalls.length > 0 ? [{
          type: 'slow_calls',
          severity: 'info',
          message: `检测到 ${slowCalls.length} 个慢调用`,
          details: slowCalls.map(call => ({
            toolName: call.toolName,
            duration: call.duration
          }))
        }] : [])
      ]
    }
  }
  
  // 生成优化建议
  private async generateRecommendations(
    toolStats: any[],
    errorStats: any[]
  ): Promise<string[]> {
    
    const recommendations: string[] = []
    
    // 分析高错误率工具
    const highErrorRateTools = toolStats.filter(stat => 
      stat.successRate < 0.9 && stat.totalCalls > 10
    )
    
    if (highErrorRateTools.length > 0) {
      recommendations.push(
        `以下工具错误率较高，建议检查配置: ${highErrorRateTools.map(t => t.toolName).join(', ')}`
      )
    }
    
    // 分析响应时间
    const slowTools = toolStats.filter(stat => stat.avgResponseTime > 5000)
    if (slowTools.length > 0) {
      recommendations.push(
        `以下工具响应较慢，建议优化: ${slowTools.map(t => t.toolName).join(', ')}`
      )
    }
    
    // 分析使用频率
    const popularTools = toolStats
      .filter(stat => stat.totalCalls > 100)
      .sort((a, b) => b.totalCalls - a.totalCalls)
      .slice(0, 5)
    
    if (popularTools.length > 0) {
      recommendations.push(
        `最受欢迎的工具: ${popularTools.map(t => t.toolName).join(', ')}，可考虑进一步优化`
      )
    }
    
    return recommendations
  }
}
```

## 🚀 MCP 集成优势与展望

### 技术优势

1. **标准化接入** - 遵循 MCP 开放标准，确保互操作性
2. **安全隔离** - 工具运行在独立进程中，保障系统安全
3. **双向通信** - 支持复杂的工具交互和数据交换
4. **生态兼容** - 与现有 MCP 工具生态无缝集成
5. **扩展灵活** - 支持自定义工具和第三方服务

### 业务价值

1. **能力扩展** - 无限扩展 AI 应用的功能边界
2. **开发效率** - 标准化的工具开发和集成流程
3. **生态建设** - 促进工具生态的繁荣发展
4. **用户体验** - 统一的工具调用和管理体验
5. **成本控制** - 复用现有工具，降低开发成本

### 未来发展方向

#### 短期优化 (3-6个月)
- [ ] 增加更多官方工具支持
- [ ] 优化工具调用性能
- [ ] 完善工具市场和管理界面
- [ ] 增强安全防护机制

#### 中期规划 (6-12个月)
- [ ] 实现工具的智能推荐
- [ ] 支持工具的版本管理
- [ ] 构建工具性能优化引擎
- [ ] 开发可视化工具编排器

#### 长期愿景 (1-2年)
- [ ] 建设完整的工具生态系统
- [ ] 实现工具的自动化测试和部署
- [ ] 支持分布式工具执行
- [ ] 构建工具 AI 助手

---

FastGPT 的 MCP 集成代表了 AI 平台工具扩展能力的最前沿实践。通过深度集成 MCP 协议，FastGPT 不仅为用户提供了强大的工具扩展能力，更重要的是为整个 AI 工具生态的标准化发展做出了重要贡献。这种前瞻性的技术选择，将为 FastGPT 在未来的竞争中赢得重要优势。