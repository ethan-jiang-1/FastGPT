# FastGPT 插件系统架构深度分析

## 🔌 插件系统架构概述

FastGPT 实现了**企业级插件生态系统**，支持多种插件类型和分发方式。通过精心设计的分层架构，提供了**安全隔离**、**版本管理**、**成本控制**和**动态扩展**能力。

### 核心架构特点

- **多源插件支持** - 个人、系统、商业和社区插件
- **工作流集成** - 插件作为工作流节点无缝集成
- **沙箱执行** - 安全的代码执行环境
- **成本追踪** - 基于积分的使用量管理
- **版本控制** - 插件版本管理和升级

## 🏗️ 插件系统架构

### 1. 插件分类架构

**文件路径**: `packages/global/core/app/plugin/constants.ts`

```typescript
// 插件来源类型
export enum PluginSourceEnum {
  personal = 'personal',        // 个人创建的应用插件（ObjectId 标识）
  systemTool = 'systemTool',    // FastGPT 核心系统工具（代码标识）
  commercial = 'commercial',    // 商业/专业版插件
  community = 'community'       // 社区插件（已弃用，被 systemTool 替代）
}

// 插件类型定义
export type PluginTypeEnum = 
  | 'personal'      // 用户自定义插件
  | 'systemTool'    // 系统内置工具
  | 'commercial'    // 付费商业插件

// 插件组类型
export type TGroupType = 
  | 'ai-model'      // AI 模型类插件
  | 'document'      // 文档处理插件
  | 'multimedia'    // 多媒体处理插件
  | 'communication' // 通信集成插件
  | 'utility'       // 实用工具插件
```

### 2. 插件节点类型

```typescript
// 插件工作流节点类型
export enum FlowNodeTypeEnum {
  pluginModule = 'pluginModule',      // 自定义插件工作流
  pluginInput = 'pluginInput',        // 插件输入配置
  pluginOutput = 'pluginOutput',      // 插件输出配置
  pluginConfig = 'pluginConfig',      // 系统插件配置
  tool = 'tool',                      // 独立工具
  toolSet = 'toolSet'                 // 工具集合
}

// 插件执行状态
export enum PluginStatusEnum {
  waiting = 'waiting',        // 等待执行
  running = 'running',        // 正在执行
  completed = 'completed',    // 执行完成
  failed = 'failed',         // 执行失败
  stopped = 'stopped'        // 手动停止
}
```

## 📦 可用插件与功能

### 1. AI 模型插件

**大语言模型插件**:
```bash
plugins/model/
├── llm-Baichuan2/          # 百川2语言模型
├── llm-ChatGLM2/           # 智谱ChatGLM2模型
├── llm-Qwen/               # 阿里通义千问模型
└── llm-DeepSeek/           # DeepSeek模型
```

**多模态模型插件**:
```bash
plugins/multimodal/
├── ocr-surya/              # Surya OCR文字识别
├── tts-cosevoice/          # CoseVoice语音合成
├── stt-sensevoice/         # SenseVoice语音识别
└── rerank-bge/             # BGE重排序模型
    ├── bge-reranker-base/
    ├── bge-reranker-large/
    └── bge-reranker-v2-m3/
```

### 2. 文档处理插件

**PDF处理插件**:
```bash
plugins/document/
├── pdf-marker/             # PDF标记和注释
├── pdf-mineru/             # MinerU PDF解析
├── pdf-mistral/            # Mistral PDF处理
└── pdf-extractor/          # PDF内容提取
```

**文档插件配置示例** (`plugins/pdf-marker/Dockerfile`):
```dockerfile
FROM python:3.11-slim

# 系统依赖
RUN apt-get update && apt-get install -y \
    libgl1-mesa-glx \
    libglib2.0-0 \
    libsm6 \
    libxext6 \
    libxrender-dev \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Python依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 插件代码
WORKDIR /app
COPY . .

# 插件配置
ENV PLUGIN_TYPE=pdf-marker
ENV PLUGIN_VERSION=1.0.0
EXPOSE 3001

# 启动命令
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "3001"]
```

### 3. 网络爬虫插件

**完整网络爬虫解决方案** (`plugins/webcrawler/`):

```python
# plugins/webcrawler/src/spider.py
class WebSpider:
    def __init__(self):
        self.session = requests.Session()
        self.session.headers.update({
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
        })
    
    async def fetch_url(self, url: str, options: dict = None) -> dict:
        """
        获取网页内容
        
        Args:
            url: 目标URL
            options: 爬取选项（超时、重试等）
            
        Returns:
            网页内容和元数据
        """
        try:
            response = await self.session.get(url, timeout=30)
            response.raise_for_status()
            
            # 内容解析
            soup = BeautifulSoup(response.content, 'html.parser')
            
            return {
                'url': url,
                'title': soup.title.string if soup.title else '',
                'content': self.extract_main_content(soup),
                'metadata': {
                    'status_code': response.status_code,
                    'content_type': response.headers.get('content-type'),
                    'last_modified': response.headers.get('last-modified'),
                    'size': len(response.content)
                }
            }
            
        except Exception as e:
            return {
                'url': url,
                'error': str(e),
                'status': 'failed'
            }
    
    def extract_main_content(self, soup):
        """提取主要内容"""
        # 移除脚本和样式
        for script in soup(["script", "style"]):
            script.decompose()
        
        # 智能内容提取
        main_content = soup.find('main') or soup.find('article')
        if not main_content:
            main_content = soup.find('div', class_=lambda x: x and 'content' in x.lower())
        
        return main_content.get_text(strip=True) if main_content else soup.get_text(strip=True)
```

**搜索引擎集成** (`plugins/webcrawler/src/search_engines/`):
```python
# 百度搜索引擎
class BaiduSearchEngine:
    def __init__(self):
        self.base_url = "https://www.baidu.com/s"
    
    async def search(self, query: str, limit: int = 10) -> List[dict]:
        """
        百度搜索
        
        Args:
            query: 搜索关键词
            limit: 结果数量限制
            
        Returns:
            搜索结果列表
        """
        params = {
            'wd': query,
            'rn': limit,
            'ie': 'utf-8'
        }
        
        response = await self.session.get(self.base_url, params=params)
        soup = BeautifulSoup(response.content, 'html.parser')
        
        results = []
        for result in soup.find_all('div', class_='result'):
            title_element = result.find('h3')
            url_element = result.find('a')
            desc_element = result.find('span', class_='content-right_8Zs40')
            
            if title_element and url_element:
                results.append({
                    'title': title_element.get_text(strip=True),
                    'url': url_element.get('href'),
                    'description': desc_element.get_text(strip=True) if desc_element else '',
                    'source': 'baidu'
                })
        
        return results[:limit]
```

### 4. 企业集成插件

**飞书集成插件模板** (`packages/templates/src/plugin-feishu/`):

```typescript
// 飞书插件配置
export const feishuPluginTemplate: AppTemplate = {
  id: 'plugin-feishu',
  name: '飞书机器人',
  description: '与飞书进行集成，支持消息发送、日程管理、文档操作等功能',
  avatar: '/imgs/app/templates/feishu.svg',
  type: AppTypeEnum.plugin,
  modules: [
    {
      nodeId: 'pluginInput',
      name: '插件开始',
      intro: '飞书插件的开始节点，接收来自飞书的消息和事件',
      flowNodeType: FlowNodeTypeEnum.pluginInput,
      position: { x: 500, y: 200 },
      version: '481',
      inputs: [
        {
          key: 'message',
          renderTypeList: [FlowNodeInputTypeEnum.reference],
          valueType: WorkflowIOValueTypeEnum.string,
          label: '用户消息',
          description: '来自飞书用户的消息内容'
        },
        {
          key: 'event_type',
          renderTypeList: [FlowNodeInputTypeEnum.reference],
          valueType: WorkflowIOValueTypeEnum.string,
          label: '事件类型',
          description: '飞书事件类型（消息、@机器人、日程等）'
        }
      ],
      outputs: [
        {
          id: 'message',
          key: 'message',
          label: '用户消息',
          valueType: WorkflowIOValueTypeEnum.string,
          type: FlowNodeOutputTypeEnum.static
        }
      ]
    },
    {
      nodeId: 'feishuAction',
      name: '飞书操作',
      intro: '执行飞书相关操作，如发送消息、创建日程等',
      flowNodeType: FlowNodeTypeEnum.httpRequest468,
      position: { x: 800, y: 200 },
      version: '481',
      inputs: [
        {
          key: 'action',
          renderTypeList: [FlowNodeInputTypeEnum.select],
          valueType: WorkflowIOValueTypeEnum.string,
          label: '操作类型',
          list: [
            { label: '发送消息', value: 'send_message' },
            { label: '创建日程', value: 'create_event' },
            { label: '获取用户信息', value: 'get_user' }
          ]
        },
        {
          key: 'params',
          renderTypeList: [FlowNodeInputTypeEnum.reference],
          valueType: WorkflowIOValueTypeEnum.object,
          label: '操作参数',
          description: '执行操作所需的参数'
        }
      ]
    }
  ],
  edges: [
    {
      source: 'pluginInput',
      target: 'feishuAction',
      sourceHandle: 'message',
      targetHandle: 'params'
    }
  ]
}
```

## 🔐 插件安全与沙箱

### 1. 沙箱执行环境

**沙箱服务架构** (`projects/sandbox/`):

```python
# sandbox/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import seccomp
import subprocess
import tempfile
import os

app = FastAPI(title="FastGPT Sandbox Service")

class CodeExecutionRequest(BaseModel):
    code: str
    language: str
    timeout: int = 30
    memory_limit: int = 128  # MB

class CodeExecutionResponse(BaseModel):
    success: bool
    output: str
    error: str = None
    execution_time: float
    memory_usage: int

@app.post("/execute", response_model=CodeExecutionResponse)
async def execute_code(request: CodeExecutionRequest):
    """
    安全执行用户代码
    """
    try:
        # 创建临时文件
        with tempfile.NamedTemporaryFile(mode='w', suffix=f'.{request.language}', delete=False) as f:
            f.write(request.code)
            temp_file = f.name
        
        # 配置seccomp安全策略
        seccomp_filter = seccomp.SyscallFilter(defaction=seccomp.KILL)
        seccomp_filter.add_rule(seccomp.ALLOW, "read")
        seccomp_filter.add_rule(seccomp.ALLOW, "write")
        seccomp_filter.add_rule(seccomp.ALLOW, "exit")
        seccomp_filter.add_rule(seccomp.ALLOW, "exit_group")
        
        # 执行代码
        start_time = time.time()
        process = subprocess.Popen(
            [get_interpreter(request.language), temp_file],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            timeout=request.timeout,
            preexec_fn=lambda: seccomp_filter.load()
        )
        
        stdout, stderr = process.communicate()
        execution_time = time.time() - start_time
        
        # 清理临时文件
        os.unlink(temp_file)
        
        return CodeExecutionResponse(
            success=process.returncode == 0,
            output=stdout.decode('utf-8'),
            error=stderr.decode('utf-8') if stderr else None,
            execution_time=execution_time,
            memory_usage=get_memory_usage(process.pid)
        )
        
    except subprocess.TimeoutExpired:
        return CodeExecutionResponse(
            success=False,
            output="",
            error="代码执行超时",
            execution_time=request.timeout,
            memory_usage=0
        )
    except Exception as e:
        return CodeExecutionResponse(
            success=False,
            output="",
            error=str(e),
            execution_time=0,
            memory_usage=0
        )

def get_interpreter(language: str) -> str:
    """获取语言解释器"""
    interpreters = {
        'python': 'python3',
        'javascript': 'node',
        'bash': 'bash'
    }
    return interpreters.get(language, 'python3')
```

### 2. 权限控制系统

**插件权限验证** (`packages/service/core/app/plugin/controller.ts`):

```typescript
// 插件权限验证
export const authPluginByTmbId = async ({
  pluginId,
  teamId,
  tmbId,
  per = ReadPermissionVal
}: {
  pluginId: string
  teamId: string
  tmbId: string
  per?: number
}): Promise<PluginItemType> => {
  try {
    // 1. 获取插件信息
    const plugin = await getPluginRuntimeById({
      pluginId,
      teamId
    })

    if (!plugin) {
      throw new Error('插件不存在')
    }

    // 2. 检查插件状态
    if (!plugin.isActive) {
      throw new Error('插件未激活')
    }

    // 3. 验证团队权限
    const hasPermission = await checkTeamPluginPermission({
      teamId,
      pluginId,
      per
    })

    if (!hasPermission) {
      throw new Error('权限不足')
    }

    // 4. 检查成员权限
    const memberPermission = await getMemberPluginPermission({
      tmbId,
      pluginId
    })

    if (memberPermission < per) {
      throw new Error('成员权限不足')
    }

    return plugin

  } catch (error) {
    console.error('插件权限验证失败:', error)
    throw error
  }
}

// 插件成本计算
export const computedPluginUsage = async ({
  plugin,
  childrenUsage,
  error
}: {
  plugin: PluginItemType
  childrenUsage: UsageItemType[]
  error?: boolean
}): Promise<number> => {
  let totalPoints = 0

  // 1. 基础使用成本
  if (plugin.currentCost && !error) {
    totalPoints += plugin.currentCost
  }

  // 2. 子流程成本累加
  for (const usage of childrenUsage) {
    totalPoints += usage.totalPoints || 0
  }

  // 3. Token费用计算
  if (plugin.hasTokenFee) {
    const tokenUsage = childrenUsage.reduce((sum, item) => {
      return sum + (item.tokens || 0)
    }, 0)
    
    totalPoints += Math.ceil(tokenUsage / 1000) * 0.1 // 每1000 token 0.1积分
  }

  // 4. 错误情况不收费
  if (error) {
    totalPoints = 0
  }

  return totalPoints
}
```

### 3. 密钥管理系统

**密钥管理接口** (`projects/app/src/pageComponents/app/plugin/SecretInputModal.tsx`):

```typescript
interface SecretConfig {
  key: string                    // 密钥标识
  label: string                 // 显示名称
  type: 'input' | 'textarea'    // 输入类型
  placeholder?: string          // 占位符
  required?: boolean           // 是否必需
  systemManaged?: boolean      // 系统管理的密钥
}

interface PluginSecretManager {
  // 获取插件密钥配置
  getSecretConfig(pluginId: string): SecretConfig[]
  
  // 设置用户密钥
  setUserSecret(teamId: string, pluginId: string, key: string, value: string): Promise<void>
  
  // 获取用户密钥
  getUserSecret(teamId: string, pluginId: string, key: string): Promise<string | null>
  
  // 删除用户密钥
  deleteUserSecret(teamId: string, pluginId: string, key: string): Promise<void>
  
  // 验证密钥有效性
  validateSecret(pluginId: string, secrets: Record<string, string>): Promise<boolean>
}

// 密钥加密存储
const encryptSecret = (value: string): string => {
  const key = process.env.AES256_SECRET_KEY!
  const cipher = crypto.createCipher('aes256', key)
  let encrypted = cipher.update(value, 'utf8', 'hex')
  encrypted += cipher.final('hex')
  return encrypted
}

const decryptSecret = (encryptedValue: string): string => {
  const key = process.env.AES256_SECRET_KEY!
  const decipher = crypto.createDecipher('aes256', key)
  let decrypted = decipher.update(encryptedValue, 'hex', 'utf8')
  decrypted += decipher.final('utf8')
  return decrypted
}
```

## 🔄 插件通信协议

### 1. MCP (Model Context Protocol) 集成

**MCP工具管理** (`projects/app/src/pages/api/core/app/mcpTools/`):

```typescript
// MCP工具创建
export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  try {
    const {
      appId,
      name,
      description,
      serverUrl,
      toolName,
      arguments: toolArgs,
      timeout = 30000
    } = req.body

    // 身份验证
    const { teamId, tmbId } = await authCert({
      req,
      authToken: true,
      per: WritePermissionVal
    })

    // 验证应用权限
    await authApp({
      appId,
      per: WritePermissionVal,
      teamId,
      tmbId
    })

    // 创建MCP工具配置
    const mcpTool = await MongoMCPTool.create({
      teamId,
      tmbId,
      appId,
      name,
      description,
      serverUrl,
      toolName,
      arguments: toolArgs,
      timeout,
      isActive: true
    })

    jsonRes(res, {
      data: mcpTool._id
    })

  } catch (error) {
    jsonRes(res, {
      code: 500,
      error: error.message
    })
  }
}

// MCP工具调用
const callMCPTool = async ({
  serverUrl,
  toolName,
  arguments: args,
  timeout = 30000
}: {
  serverUrl: string
  toolName: string
  arguments: Record<string, any>
  timeout?: number
}): Promise<any> => {
  try {
    // 建立MCP连接
    const mcpClient = new MCPClient(serverUrl)
    await mcpClient.connect()

    // 调用工具
    const result = await mcpClient.callTool({
      name: toolName,
      arguments: args
    }, { timeout })

    await mcpClient.disconnect()
    return result

  } catch (error) {
    console.error('MCP工具调用失败:', error)
    throw new Error(`MCP工具调用失败: ${error.message}`)
  }
}
```

### 2. HTTP插件协议

**HTTP插件集成** (`projects/app/src/pages/api/core/app/httpPlugin/`):

```typescript
// HTTP插件配置
interface HttpPluginConfig {
  name: string
  description: string
  baseUrl: string
  apiSchemaStr: string          // OpenAPI schema JSON字符串
  customHeaders?: string        // 自定义请求头JSON字符串
  timeout?: number             // 超时时间（毫秒）
}

// HTTP插件调用
const callHttpPlugin = async ({
  config,
  endpoint,
  method,
  params,
  headers = {}
}: {
  config: HttpPluginConfig
  endpoint: string
  method: string
  params: Record<string, any>
  headers?: Record<string, string>
}): Promise<any> => {
  try {
    // 解析自定义头部
    const customHeaders = config.customHeaders 
      ? JSON.parse(config.customHeaders) 
      : {}

    // 合并请求头
    const requestHeaders = {
      'Content-Type': 'application/json',
      'User-Agent': 'FastGPT-Plugin/1.0',
      ...customHeaders,
      ...headers
    }

    // 构建请求URL
    const url = `${config.baseUrl.replace(/\/$/, '')}/${endpoint.replace(/^\//, '')}`

    // 发送HTTP请求
    const response = await fetch(url, {
      method: method.toUpperCase(),
      headers: requestHeaders,
      body: method.toUpperCase() !== 'GET' ? JSON.stringify(params) : undefined,
      timeout: config.timeout || 30000
    })

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`)
    }

    const result = await response.json()
    return result

  } catch (error) {
    console.error('HTTP插件调用失败:', error)
    throw new Error(`HTTP插件调用失败: ${error.message}`)
  }
}

// OpenAPI Schema解析
const parseOpenAPISchema = (apiSchemaStr: string): OpenAPISchema => {
  try {
    const schema = JSON.parse(apiSchemaStr)
    
    // 验证OpenAPI格式
    if (!schema.openapi && !schema.swagger) {
      throw new Error('无效的OpenAPI Schema')
    }

    return {
      info: schema.info || {},
      servers: schema.servers || [],
      paths: schema.paths || {},
      components: schema.components || {}
    }

  } catch (error) {
    throw new Error(`OpenAPI Schema解析失败: ${error.message}`)
  }
}
```

## ⚙️ 插件配置与自定义

### 1. 插件数据库架构

**系统插件配置表** (`packages/service/core/app/plugin/systemPluginSchema.ts`):

```typescript
// 系统插件配置Schema
const SystemPluginConfigSchema = new Schema({
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'teams',
    required: true
  },
  pluginId: {
    type: String,
    required: true
  },
  originCost: {
    type: Number,
    required: true,
    default: 0
  },
  currentCost: {
    type: Number,
    required: true,
    default: 0
  },
  hasTokenFee: {
    type: Boolean,
    default: false
  },
  isActive: {
    type: Boolean,
    required: true,
    default: true
  },
  pluginOrder: {
    type: Number,
    default: 0
  },
  customConfig: {
    type: Object,
    default: {}
  },
  inputListVal: {
    type: Object,
    default: {}
  }
})

// 插件组配置Schema
const PluginGroupSchema = new Schema({
  teamId: {
    type: Schema.Types.ObjectId,
    ref: 'teams',
    required: true
  },
  groupId: {
    type: String,
    required: true
  },
  groupAvatar: {
    type: String,
    default: ''
  },
  groupName: {
    type: String,
    required: true
  },
  groupTypes: {
    type: [String],
    required: true
  },
  groupOrder: {
    type: Number,
    default: 0
  }
})

// 索引优化
SystemPluginConfigSchema.index({ teamId: 1, pluginId: 1 }, { unique: true })
PluginGroupSchema.index({ teamId: 1, groupId: 1 }, { unique: true })
```

### 2. 插件模板系统

**模板注册机制** (`packages/templates/register.ts`):

```typescript
// 插件模板注册
export const registerPluginTemplates = () => {
  const templates = new Map<string, AppTemplate>()

  // DALLE3图像生成插件
  templates.set('plugin-dalle', {
    id: 'plugin-dalle',
    templateType: AppTemplateTypeEnum.plugin,
    name: 'DALLE3 图像生成',
    description: '使用OpenAI DALLE3模型生成高质量图像',
    avatar: '/imgs/app/templates/dalle.svg',
    tags: ['图像生成', 'AI绘画', 'OpenAI'],
    type: AppTypeEnum.plugin,
    modules: [
      {
        nodeId: 'pluginInput',
        name: '插件输入',
        flowNodeType: FlowNodeTypeEnum.pluginInput,
        inputs: [
          {
            key: 'prompt',
            renderTypeList: [FlowNodeInputTypeEnum.reference],
            valueType: WorkflowIOValueTypeEnum.string,
            label: '图像描述',
            description: '描述您想要生成的图像'
          },
          {
            key: 'size',
            renderTypeList: [FlowNodeInputTypeEnum.select],
            valueType: WorkflowIOValueTypeEnum.string,
            label: '图像尺寸',
            list: [
              { label: '1024x1024', value: '1024x1024' },
              { label: '1792x1024', value: '1792x1024' },
              { label: '1024x1792', value: '1024x1792' }
            ],
            value: '1024x1024'
          }
        ]
      },
      {
        nodeId: 'dalleGenerate',
        name: 'DALLE3生成',
        flowNodeType: FlowNodeTypeEnum.httpRequest468,
        inputs: [
          {
            key: 'url',
            valueType: WorkflowIOValueTypeEnum.string,
            value: 'https://api.openai.com/v1/images/generations'
          },
          {
            key: 'method',
            valueType: WorkflowIOValueTypeEnum.string,
            value: 'POST'
          }
        ]
      }
    ]
  })

  // GitHub Issues插件
  templates.set('plugin-github', {
    id: 'plugin-github',
    templateType: AppTemplateTypeEnum.plugin,
    name: 'GitHub Issues管理',
    description: '管理GitHub仓库的Issues，包括创建、查询、更新等操作',
    avatar: '/imgs/app/templates/github.svg',
    tags: ['代码管理', 'GitHub', 'Issues'],
    type: AppTypeEnum.plugin,
    modules: [
      {
        nodeId: 'pluginInput',
        name: '插件输入',
        flowNodeType: FlowNodeTypeEnum.pluginInput,
        inputs: [
          {
            key: 'action',
            renderTypeList: [FlowNodeInputTypeEnum.select],
            valueType: WorkflowIOValueTypeEnum.string,
            label: '操作类型',
            list: [
              { label: '创建Issue', value: 'create' },
              { label: '查询Issues', value: 'list' },
              { label: '更新Issue', value: 'update' },
              { label: '关闭Issue', value: 'close' }
            ]
          },
          {
            key: 'repo',
            renderTypeList: [FlowNodeInputTypeEnum.reference],
            valueType: WorkflowIOValueTypeEnum.string,
            label: '仓库名称',
            description: '格式：owner/repo'
          }
        ]
      }
    ]
  })

  return templates
}
```

### 3. 插件UI组件

**插件输入输出配置UI** (`projects/app/src/pageComponents/app/detail/WorkflowComponents/Flow/nodes/NodePluginIO/`):

```typescript
// 插件输入配置组件
const PluginInputConfig: React.FC<{
  inputs: FlowNodeInputItemType[]
  onUpdate: (inputs: FlowNodeInputItemType[]) => void
}> = ({ inputs, onUpdate }) => {
  const { t } = useTranslation()

  const addInput = () => {
    const newInput: FlowNodeInputItemType = {
      key: `input_${Date.now()}`,
      renderTypeList: [FlowNodeInputTypeEnum.reference],
      valueType: WorkflowIOValueTypeEnum.string,
      label: '新输入',
      description: ''
    }
    onUpdate([...inputs, newInput])
  }

  const updateInput = (index: number, field: string, value: any) => {
    const updatedInputs = [...inputs]
    updatedInputs[index] = {
      ...updatedInputs[index],
      [field]: value
    }
    onUpdate(updatedInputs)
  }

  const removeInput = (index: number) => {
    const updatedInputs = inputs.filter((_, i) => i !== index)
    onUpdate(updatedInputs)
  }

  return (
    <Box>
      <HStack mb={4}>
        <Text fontSize="lg" fontWeight="medium">
          {t('plugin.input_config')}
        </Text>
        <Button size="sm" onClick={addInput}>
          {t('common.add')}
        </Button>
      </HStack>

      {inputs.map((input, index) => (
        <Box key={input.key} p={4} border="1px" borderColor="gray.200" borderRadius="md" mb={3}>
          <HStack mb={3}>
            <FormControl>
              <FormLabel>{t('plugin.input_key')}</FormLabel>
              <Input
                value={input.key}
                onChange={(e) => updateInput(index, 'key', e.target.value)}
              />
            </FormControl>
            <FormControl>
              <FormLabel>{t('plugin.input_label')}</FormLabel>
              <Input
                value={input.label}
                onChange={(e) => updateInput(index, 'label', e.target.value)}
              />
            </FormControl>
            <IconButton
              aria-label="删除输入"
              icon={<DeleteIcon />}
              size="sm"
              colorScheme="red"
              onClick={() => removeInput(index)}
            />
          </HStack>

          <FormControl mb={3}>
            <FormLabel>{t('plugin.input_description')}</FormLabel>
            <Textarea
              value={input.description || ''}
              onChange={(e) => updateInput(index, 'description', e.target.value)}
              placeholder={t('plugin.input_description_placeholder')}
            />
          </FormControl>

          <HStack>
            <FormControl>
              <FormLabel>{t('plugin.value_type')}</FormLabel>
              <Select
                value={input.valueType}
                onChange={(e) => updateInput(index, 'valueType', e.target.value)}
              >
                <option value={WorkflowIOValueTypeEnum.string}>字符串</option>
                <option value={WorkflowIOValueTypeEnum.number}>数字</option>
                <option value={WorkflowIOValueTypeEnum.boolean}>布尔值</option>
                <option value={WorkflowIOValueTypeEnum.object}>对象</option>
                <option value={WorkflowIOValueTypeEnum.arrayString}>字符串数组</option>
              </Select>
            </FormControl>

            <FormControl>
              <FormLabel>{t('plugin.render_type')}</FormLabel>
              <Select
                value={input.renderTypeList[0]}
                onChange={(e) => updateInput(index, 'renderTypeList', [e.target.value])}
              >
                <option value={FlowNodeInputTypeEnum.reference}>变量引用</option>
                <option value={FlowNodeInputTypeEnum.input}>手动输入</option>
                <option value={FlowNodeInputTypeEnum.select}>下拉选择</option>
                <option value={FlowNodeInputTypeEnum.slider}>滑块</option>
              </Select>
            </FormControl>
          </HStack>
        </Box>
      ))}
    </Box>
  )
}
```

## 🏪 插件市场与分发

### 1. 当前分发机制

```typescript
// 插件分发策略
export enum PluginDistributionType {
  FILE_BASED = 'file_based',           // 基于文件的模板分发
  DATABASE_MANAGED = 'database_managed', // 数据库管理的插件
  DOCKER_REGISTRY = 'docker_registry',   // Docker镜像仓库分发
  HTTP_API = 'http_api'                 // HTTP API集成
}

// 分发渠道
const distributionChannels = {
  // 1. 文件模板分发（当前主要方式）
  templates: {
    path: 'packages/templates/src/',
    registration: 'packages/templates/register.ts',
    versioning: 'git-based'
  },

  // 2. 数据库配置分发
  database: {
    collection: 'system_plugin_configs',
    schema: 'SystemPluginConfigSchema',
    management: 'admin-ui'
  },

  // 3. Docker镜像分发
  docker: {
    registry: 'ghcr.io/labring/fastgpt-plugins/',
    models: 'plugins/model/',
    automation: '.github/workflows/'
  },

  // 4. HTTP插件集成
  http: {
    api: 'projects/app/src/pages/api/core/app/httpPlugin/',
    openapi: 'OpenAPI 3.0 Schema',
    authentication: 'API Key / OAuth2'
  }
}
```

### 2. 插件版本管理

```typescript
// 插件版本控制
interface PluginVersion {
  pluginId: string
  version: string
  releaseDate: Date
  changelog: string[]
  compatibility: {
    minFastGPTVersion: string
    maxFastGPTVersion?: string
  }
  dependencies: {
    [key: string]: string
  }
  deprecated?: boolean
  securityUpdates?: string[]
}

// 版本管理API
export const getPluginVersionList = async (pluginId: string): Promise<PluginVersion[]> => {
  try {
    // 从数据库或API获取版本列表
    const versions = await MongoPluginVersion
      .find({ pluginId })
      .sort({ releaseDate: -1 })
      .lean()

    return versions.map(version => ({
      ...version,
      compatibility: JSON.parse(version.compatibility),
      dependencies: JSON.parse(version.dependencies || '{}'),
      changelog: JSON.parse(version.changelog || '[]')
    }))

  } catch (error) {
    console.error('获取插件版本列表失败:', error)
    return []
  }
}

// 插件升级检查
export const checkPluginUpdates = async (teamId: string): Promise<PluginUpdate[]> => {
  const installedPlugins = await getTeamInstalledPlugins(teamId)
  const updates: PluginUpdate[] = []

  for (const plugin of installedPlugins) {
    const latestVersion = await getLatestPluginVersion(plugin.pluginId)
    
    if (latestVersion && isVersionNewer(latestVersion.version, plugin.version)) {
      updates.push({
        pluginId: plugin.pluginId,
        currentVersion: plugin.version,
        latestVersion: latestVersion.version,
        updateType: getUpdateType(plugin.version, latestVersion.version),
        securityUpdate: latestVersion.securityUpdates && latestVersion.securityUpdates.length > 0
      })
    }
  }

  return updates
}
```

### 3. 插件安装流程

```typescript
// 插件安装管理器
export class PluginInstaller {
  async installPlugin(props: {
    teamId: string
    tmbId: string
    pluginId: string
    version?: string
    config?: Record<string, any>
  }): Promise<InstallResult> {
    const { teamId, tmbId, pluginId, version = 'latest', config = {} } = props

    try {
      // 1. 验证权限
      await this.checkInstallPermission(teamId, tmbId)

      // 2. 获取插件信息
      const pluginInfo = await this.getPluginInfo(pluginId, version)
      
      // 3. 检查依赖
      await this.checkDependencies(pluginInfo.dependencies)

      // 4. 验证兼容性
      await this.checkCompatibility(pluginInfo.compatibility)

      // 5. 安装插件
      const installation = await this.performInstallation({
        teamId,
        tmbId,
        plugin: pluginInfo,
        config
      })

      // 6. 配置插件
      await this.configurePlugin(installation.id, config)

      // 7. 激活插件
      await this.activatePlugin(installation.id)

      return {
        success: true,
        installationId: installation.id,
        message: '插件安装成功'
      }

    } catch (error) {
      console.error('插件安装失败:', error)
      return {
        success: false,
        error: error.message
      }
    }
  }

  private async performInstallation(props: {
    teamId: string
    tmbId: string
    plugin: PluginInfo
    config: Record<string, any>
  }): Promise<PluginInstallation> {
    const { teamId, tmbId, plugin, config } = props

    // 创建安装记录
    const installation = await MongoPluginInstallation.create({
      teamId,
      tmbId,
      pluginId: plugin.id,
      version: plugin.version,
      status: 'installing',
      config,
      installedAt: new Date()
    })

    // 根据插件类型执行不同的安装逻辑
    switch (plugin.type) {
      case 'docker':
        await this.installDockerPlugin(plugin, installation)
        break
      case 'http':
        await this.installHttpPlugin(plugin, installation)
        break
      case 'template':
        await this.installTemplatePlugin(plugin, installation)
        break
      default:
        throw new Error(`不支持的插件类型: ${plugin.type}`)
    }

    // 更新安装状态
    installation.status = 'installed'
    await installation.save()

    return installation
  }

  private async installDockerPlugin(plugin: PluginInfo, installation: PluginInstallation): Promise<void> {
    // Docker插件安装逻辑
    const dockerConfig = {
      image: plugin.dockerImage,
      tag: plugin.version,
      environment: {
        PLUGIN_ID: plugin.id,
        TEAM_ID: installation.teamId,
        API_KEY: await this.generatePluginApiKey(installation)
      },
      networks: ['fastgpt'],
      restart: 'unless-stopped'
    }

    // 拉取Docker镜像
    await this.pullDockerImage(dockerConfig.image, dockerConfig.tag)

    // 启动容器
    await this.startDockerContainer(dockerConfig)
  }
}
```

---

FastGPT 的插件系统展现了企业级软件的成熟设计理念，通过**安全隔离**、**版本控制**、**权限管理**和**灵活扩展**，构建了一个功能强大且安全可靠的插件生态系统。无论是内置系统工具还是第三方插件，都能在统一的框架下安全运行并提供丰富的功能扩展。