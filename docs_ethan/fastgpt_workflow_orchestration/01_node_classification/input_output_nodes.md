# FastGPT 输入输出节点深度分析

## 🔌 输入输出节点概述

输入输出节点是 FastGPT 工作流编排系统中的**边界节点**，负责处理工作流与外部世界的数据交换。这些节点定义了用户如何与工作流交互，以及工作流如何向用户反馈结果。

### 核心设计理念

- **🎯 用户中心** - 以用户体验为中心设计交互界面
- **🔄 双向交互** - 支持数据输入和结果输出的双向流动
- **🎨 灵活配置** - 丰富的配置选项适应不同场景需求
- **⚡ 实时响应** - 支持实时交互和即时反馈
- **🔒 类型安全** - 强类型约束保证数据完整性

## 📊 输入输出节点分类

### 按交互方向分类

| 分类 | 节点类型 | 主要功能 | 使用场景 |
|------|----------|----------|----------|
| **数据输入** | `workflowStart`, `formInput`, `pluginInput` | 接收外部数据 | 工作流启动、用户输入 |
| **交互选择** | `userSelect`, `customFeedback` | 用户交互选择 | 决策点、反馈收集 |
| **结果输出** | `answerNode`, `pluginOutput` | 输出处理结果 | 结果展示、数据传递 |

### 按应用场景分类

| 场景 | 适用节点 | 典型配置 |
|------|----------|----------|
| **工作流启动** | `workflowStart` | 系统变量、用户输入、文件上传 |
| **用户决策** | `userSelect` | 选项列表、描述文本 |
| **数据收集** | `formInput` | 表单字段、验证规则 |
| **结果展示** | `answerNode` | 文本内容、格式化输出 |
| **插件接口** | `pluginInput/Output` | 参数定义、数据类型 |
| **反馈收集** | `customFeedback` | 反馈类型、收集选项 |

## 🚀 核心输入节点详解

### 1. 工作流开始节点 (workflowStart)

**节点标识**: `FlowNodeTypeEnum.workflowStart`  
**模板分类**: `systemInput`  
**核心功能**: 工作流的入口点，处理初始输入和系统变量

#### 输入接口定义

```typescript
interface WorkflowStartInputs {
  // 用户问题输入
  userChatInput: {
    key: 'userChatInput'
    renderTypeList: [FlowNodeInputTypeEnum.reference, FlowNodeInputTypeEnum.textarea]
    valueType: WorkflowIOValueTypeEnum.string
    label: '用户问题'
    required: true
    toolDescription: '用户提出的问题或需求'
  }
  
  // 文件输入（可选）
  files: {
    key: 'files'  
    renderTypeList: [FlowNodeInputTypeEnum.reference]
    valueType: WorkflowIOValueTypeEnum.arrayString
    label: '文件输入'
    description: '用户上传的文件列表'
  }
  
  // 系统变量（自动注入）
  variables: {
    userId: string        // 用户ID
    appId: string         // 应用ID  
    chatId: string        // 会话ID
    timestamp: number     // 时间戳
    userProfile: object   // 用户信息
  }
}
```

#### 输出接口定义

```typescript
interface WorkflowStartOutputs {
  // 用户输入内容
  userChatInput: {
    key: 'userChatInput'
    label: '用户问题'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.string
  }
  
  // 系统变量
  userId: {
    key: 'userId'
    label: '用户ID'
    type: FlowNodeOutputTypeEnum.static  
    valueType: WorkflowIOValueTypeEnum.string
  }
  
  // 文件列表
  files: {
    key: 'files'
    label: '文件列表'
    type: FlowNodeOutputTypeEnum.static
    valueType: WorkflowIOValueTypeEnum.arrayString
  }
}
```

#### 配置示例

```typescript
// 基础聊天启动配置
const basicChatStartConfig = {
  nodeId: 'workflowStart',
  name: '流程开始',
  intro: '用户输入，开始执行工作流',
  inputs: [
    {
      key: 'userChatInput',
      renderTypeList: [FlowNodeInputTypeEnum.textarea],
      valueType: WorkflowIOValueTypeEnum.string,
      label: '用户问题',
      required: true,
      placeholder: '请输入您的问题...',
      description: '用户的问题输入，支持多行文本'
    }
  ]
}

// 多模态输入配置
const multimodalStartConfig = {
  nodeId: 'workflowStart',
  name: '多模态输入',
  inputs: [
    {
      key: 'userChatInput',
      renderTypeList: [FlowNodeInputTypeEnum.textarea],
      valueType: WorkflowIOValueTypeEnum.string,
      label: '文本输入'
    },
    {
      key: 'files',
      renderTypeList: [FlowNodeInputTypeEnum.fileSelect],
      valueType: WorkflowIOValueTypeEnum.arrayString,
      label: '文件上传',
      fileTypes: ['image/*', 'application/pdf', 'text/*'],
      maxFiles: 5,
      maxSize: '10MB'
    }
  ]
}
```

#### 使用场景

**1. 简单问答场景**
```
用户输入问题 → workflowStart → AI处理 → 返回答案
```

**2. 文档分析场景**  
```
用户上传文档 + 提问 → workflowStart → 文档解析 → AI分析 → 返回结果
```

**3. 带上下文的对话**
```
用户输入 + 历史对话 → workflowStart → 上下文处理 → AI回复
```

### 2. 表单输入节点 (formInput)

**节点标识**: `FlowNodeTypeEnum.formInput`  
**模板分类**: `interactive`  
**核心功能**: 收集结构化的用户输入数据

#### 表单字段类型支持

```typescript
interface FormFieldTypes {
  // 基础输入类型
  input: {
    type: 'input'
    label: string
    key: string
    required?: boolean
    placeholder?: string
    defaultValue?: string
    validation?: RegExp
  }
  
  // 多行文本
  textarea: {
    type: 'textarea'
    label: string
    key: string
    rows?: number
    maxLength?: number
  }
  
  // 数字输入
  numberInput: {
    type: 'numberInput'
    label: string
    key: string
    min?: number
    max?: number
    step?: number
  }
  
  // 选择器
  select: {
    type: 'select'
    label: string
    key: string
    options: Array<{label: string, value: string}>
    multiple?: boolean
  }
  
  // 开关
  switch: {
    type: 'switch'
    label: string
    key: string
    defaultValue?: boolean
  }
  
  // 文件上传
  fileUpload: {
    type: 'fileUpload'
    label: string
    key: string
    accept?: string
    multiple?: boolean
    maxSize?: string
  }
}
```

#### 复杂表单配置示例

```typescript
// 用户信息收集表单
const userInfoFormConfig = {
  nodeId: 'formInput_userInfo',
  name: '用户信息收集',
  description: '收集用户的基本信息和需求',
  inputs: [
    {
      key: 'formFields',
      renderTypeList: [FlowNodeInputTypeEnum.custom],
      valueType: WorkflowIOValueTypeEnum.object,
      label: '表单字段配置',
      value: {
        fields: [
          {
            type: 'input',
            key: 'name',
            label: '姓名',
            required: true,
            placeholder: '请输入您的姓名',
            validation: '^[\\u4e00-\\u9fa5a-zA-Z\\s]{2,20}$'
          },
          {
            type: 'input', 
            key: 'email',
            label: '邮箱',
            required: true,
            placeholder: 'example@email.com',
            validation: '^[\\w-\\.]+@([\\w-]+\\.)+[\\w-]{2,4}$'
          },
          {
            type: 'select',
            key: 'department',
            label: '部门',
            required: true,
            options: [
              {label: '技术部', value: 'tech'},
              {label: '销售部', value: 'sales'},
              {label: '市场部', value: 'marketing'},
              {label: '人事部', value: 'hr'}
            ]
          },
          {
            type: 'textarea',
            key: 'requirements',
            label: '具体需求',
            placeholder: '请详细描述您的需求...',
            rows: 4,
            maxLength: 500
          },
          {
            type: 'numberInput',
            key: 'priority',
            label: '优先级',
            min: 1,
            max: 5,
            defaultValue: 3,
            step: 1
          },
          {
            type: 'switch',
            key: 'urgent',
            label: '是否紧急',
            defaultValue: false
          },
          {
            type: 'fileUpload',
            key: 'attachments',
            label: '相关文件',
            accept: '.pdf,.doc,.docx,.txt,.png,.jpg',
            multiple: true,
            maxSize: '5MB'
          }
        ]
      }
    }
  ],
  outputs: [
    {
      key: 'formData',
      label: '表单数据',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.object
    },
    {
      key: 'isValid',
      label: '验证状态',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.boolean
    }
  ]
}

// 产品反馈表单
const productFeedbackForm = {
  nodeId: 'formInput_feedback',
  name: '产品反馈表单',
  inputs: [
    {
      key: 'formFields',
      value: {
        title: '产品使用反馈',
        description: '您的反馈对我们很重要，请花几分钟时间填写',
        fields: [
          {
            type: 'select',
            key: 'product',
            label: '产品名称',
            required: true,
            options: [
              {label: 'FastGPT', value: 'fastgpt'},
              {label: 'ChatBot', value: 'chatbot'},
              {label: 'Knowledge Base', value: 'kb'}
            ]
          },
          {
            type: 'select',
            key: 'rating',
            label: '满意度评分',
            required: true,
            options: [
              {label: '⭐⭐⭐⭐⭐ 非常满意', value: 5},
              {label: '⭐⭐⭐⭐ 满意', value: 4},
              {label: '⭐⭐⭐ 一般', value: 3},
              {label: '⭐⭐ 不满意', value: 2},
              {label: '⭐ 非常不满意', value: 1}
            ]
          },
          {
            type: 'textarea',
            key: 'feedback',
            label: '详细反馈',
            placeholder: '请详细描述您的使用体验、遇到的问题或改进建议...',
            required: true,
            rows: 6,
            maxLength: 1000
          },
          {
            type: 'multipleSelect',
            key: 'features',
            label: '最常用的功能',
            options: [
              {label: '智能对话', value: 'chat'},
              {label: '知识库搜索', value: 'search'},
              {label: '工作流编排', value: 'workflow'},
              {label: '文件上传', value: 'upload'},
              {label: '团队协作', value: 'team'},
              {label: '数据分析', value: 'analytics'}
            ]
          },
          {
            type: 'input',
            key: 'contact',
            label: '联系方式（可选）',
            placeholder: '如需我们联系您，请留下邮箱或电话',
            validation: '^[\\w-\\.]+@([\\w-]+\\.)+[\\w-]{2,4}$|^1[3-9]\\d{9}$'
          }
        ],
        submitText: '提交反馈',
        resetText: '重置表单'
      }
    }
  ]
}
```

#### 表单验证机制

```typescript
interface FormValidation {
  // 字段级验证
  fieldValidation: {
    required: boolean           // 必填验证
    pattern: RegExp            // 正则表达式验证
    minLength: number          // 最小长度
    maxLength: number          // 最大长度
    min: number               // 最小值（数字）
    max: number               // 最大值（数字）
    customValidator: Function  // 自定义验证函数
  }
  
  // 表单级验证
  formValidation: {
    requiredFields: string[]   // 必填字段列表
    conditionalLogic: Array<{  // 条件逻辑验证
      condition: string        // 条件表达式
      action: 'show' | 'hide' | 'require' | 'disable'
      targetFields: string[]   // 目标字段
    }>
    crossFieldValidation: Array<{ // 跨字段验证
      fields: string[]         // 相关字段
      validator: Function      // 验证函数
      message: string         // 错误信息
    }>
  }
}

// 复杂验证示例
const complexValidationExample = {
  fieldValidation: {
    email: {
      required: true,
      pattern: /^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/,
      message: '请输入有效的邮箱地址'
    },
    password: {
      required: true,
      minLength: 8,
      pattern: /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/,
      message: '密码必须包含大小写字母、数字和特殊字符，长度至少8位'
    },
    phone: {
      pattern: /^1[3-9]\d{9}$/,
      message: '请输入有效的手机号码'
    }
  },
  conditionalLogic: [
    {
      condition: 'contact_method === "email"',
      action: 'require',
      targetFields: ['email']
    },
    {
      condition: 'contact_method === "phone"', 
      action: 'require',
      targetFields: ['phone']
    },
    {
      condition: 'user_type === "enterprise"',
      action: 'show',
      targetFields: ['company_name', 'company_size']
    }
  ],
  crossFieldValidation: [
    {
      fields: ['password', 'confirm_password'],
      validator: (values) => values.password === values.confirm_password,
      message: '两次输入的密码不一致'
    }
  ]
}
```

### 3. 用户选择节点 (userSelect)

**节点标识**: `FlowNodeTypeEnum.userSelect`  
**模板分类**: `interactive`  
**核心功能**: 向用户展示选项并收集选择结果

#### 选择类型支持

```typescript
interface UserSelectTypes {
  // 单选按钮
  radio: {
    type: 'radio'
    options: Array<{
      label: string
      value: string
      description?: string
      icon?: string
    }>
    defaultValue?: string
  }
  
  // 多选框
  checkbox: {
    type: 'checkbox'
    options: Array<{
      label: string
      value: string
      description?: string
    }>
    minSelect?: number
    maxSelect?: number
  }
  
  // 下拉选择
  dropdown: {
    type: 'dropdown'
    options: Array<{label: string, value: string}>
    multiple?: boolean
    searchable?: boolean
  }
  
  // 按钮选择
  buttonGroup: {
    type: 'buttonGroup'
    options: Array<{
      label: string
      value: string
      variant: 'primary' | 'secondary' | 'outline'
      icon?: string
    }>
    layout: 'horizontal' | 'vertical' | 'grid'
  }
  
  // 卡片选择
  cardSelect: {
    type: 'cardSelect'
    options: Array<{
      title: string
      value: string
      description: string
      image?: string
      features?: string[]
    }>
    columns?: number
  }
}
```

#### 高级选择配置示例

```typescript
// 服务类型选择
const serviceTypeSelect = {
  nodeId: 'userSelect_serviceType',
  name: '服务类型选择',
  description: '请选择您需要的服务类型',
  inputs: [
    {
      key: 'selectConfig',
      value: {
        type: 'cardSelect',
        title: '请选择服务类型',
        description: '根据您的需求选择最适合的服务方案',
        options: [
          {
            title: '基础咨询',
            value: 'consultation',
            description: '获得专业的咨询建议和解决方案',
            image: '/icons/consultation.svg',
            features: [
              '专业顾问一对一服务',
              '详细分析报告',
              '解决方案推荐',
              '后续跟进支持'
            ],
            price: '￥99/小时'
          },
          {
            title: '技术支持',
            value: 'technical',
            description: '解决技术问题和系统集成',
            image: '/icons/technical.svg',
            features: [
              '技术专家远程支持',
              '系统集成服务',
              '故障排查修复',
              '性能优化建议'
            ],
            price: '￥199/小时'
          },
          {
            title: '定制开发',
            value: 'development',
            description: '根据需求定制开发解决方案',
            image: '/icons/development.svg',
            features: [
              '需求分析设计',
              '定制化开发',
              '测试部署上线',
              '维护升级服务'
            ],
            price: '面议'
          }
        ],
        columns: 3,
        allowMultiple: false
      }
    }
  ],
  outputs: [
    {
      key: 'selectedService',
      label: '选择的服务',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.string
    },
    {
      key: 'serviceDetails',
      label: '服务详情',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.object
    }
  ]
}

// 多维度评估选择
const evaluationSelect = {
  nodeId: 'userSelect_evaluation',
  name: '多维度评估',
  inputs: [
    {
      key: 'selectConfig',
      value: {
        type: 'multiDimension',
        title: '请对以下方面进行评估',
        dimensions: [
          {
            name: '功能完整性',
            key: 'functionality',
            type: 'scale',
            scale: {
              min: 1,
              max: 5,
              labels: ['很差', '较差', '一般', '较好', '很好'],
              description: '产品功能是否满足您的需求'
            }
          },
          {
            name: '易用性',
            key: 'usability', 
            type: 'scale',
            scale: {
              min: 1,
              max: 5,
              labels: ['很难用', '较难用', '一般', '较易用', '很易用'],
              description: '产品是否容易学习和使用'
            }
          },
          {
            name: '性能表现',
            key: 'performance',
            type: 'scale', 
            scale: {
              min: 1,
              max: 5,
              labels: ['很慢', '较慢', '一般', '较快', '很快'],
              description: '产品响应速度和稳定性'
            }
          },
          {
            name: '推荐意愿',
            key: 'recommendation',
            type: 'nps',
            nps: {
              min: 0,
              max: 10,
              lowLabel: '完全不会推荐',
              highLabel: '非常愿意推荐',
              description: '您向朋友推荐此产品的可能性'
            }
          }
        ]
      }
    }
  ]
}
```

#### 条件选择逻辑

```typescript
interface ConditionalSelect {
  // 级联选择
  cascading: {
    levels: Array<{
      key: string
      label: string
      options: Array<{
        label: string
        value: string
        children?: Array<{label: string, value: string}>
      }>
      dependsOn?: string  // 依赖的上级选择
    }>
  }
  
  // 动态选项
  dynamicOptions: {
    source: 'api' | 'database' | 'variable'
    config: {
      url?: string          // API地址
      method?: string       // HTTP方法
      params?: object       // 请求参数
      transform?: Function  // 数据转换函数
    }
  }
  
  // 条件显示
  conditionalDisplay: {
    rules: Array<{
      condition: string     // 条件表达式
      action: 'show' | 'hide' | 'disable'
      targets: string[]     // 目标选项
    }>
  }
}

// 级联选择示例 - 地区选择
const regionCascadeSelect = {
  nodeId: 'userSelect_region',
  name: '地区选择',
  inputs: [
    {
      key: 'selectConfig',
      value: {
        type: 'cascading',
        levels: [
          {
            key: 'province',
            label: '省份',
            options: [
              {
                label: '北京市',
                value: 'beijing',
                children: [
                  {label: '东城区', value: 'dongcheng'},
                  {label: '西城区', value: 'xicheng'},
                  {label: '朝阳区', value: 'chaoyang'},
                  {label: '海淀区', value: 'haidian'}
                ]
              },
              {
                label: '上海市', 
                value: 'shanghai',
                children: [
                  {label: '黄浦区', value: 'huangpu'},
                  {label: '徐汇区', value: 'xuhui'},
                  {label: '长宁区', value: 'changning'},
                  {label: '静安区', value: 'jingan'}
                ]
              }
            ]
          },
          {
            key: 'city',
            label: '城市',
            dependsOn: 'province'
          },
          {
            key: 'district',
            label: '区县',
            dependsOn: 'city'
          }
        ]
      }
    }
  ]
}
```

## 💬 核心输出节点详解

### 1. 答案节点 (answerNode)

**节点标识**: `FlowNodeTypeEnum.answerNode`
**模板分类**: `function`
**核心功能**: 格式化输出工作流的最终结果

#### 输出格式类型

```typescript
interface AnswerFormats {
  // 纯文本格式
  text: {
    format: 'text'
    content: string
    maxLength?: number
    wordWrap?: boolean
  }
  
  // Markdown格式
  markdown: {
    format: 'markdown'
    content: string
    enableTOC?: boolean      // 目录
    enableCodeHighlight?: boolean  // 代码高亮
    theme?: 'default' | 'github' | 'dark'
  }
  
  // HTML格式
  html: {
    format: 'html'
    content: string
    sanitize?: boolean      // HTML安全过滤
    allowedTags?: string[]  // 允许的HTML标签
  }
  
  // JSON格式
  json: {
    format: 'json'
    data: object
    prettyPrint?: boolean   // 格式化输出
    expandLevel?: number    // 展开层级
  }
  
  // 卡片格式
  card: {
    format: 'card'
    title: string
    content: string
    image?: string
    actions?: Array<{
      label: string
      type: 'button' | 'link'
      action: string
    }>
  }
  
  // 列表格式
  list: {
    format: 'list'
    items: Array<{
      title: string
      description?: string
      icon?: string
      link?: string
    }>
    type: 'ordered' | 'unordered' | 'definition'
  }
  
  // 表格格式
  table: {
    format: 'table'
    headers: string[]
    rows: string[][]
    sortable?: boolean
    searchable?: boolean
    pagination?: boolean
  }
  
  // 图表格式
  chart: {
    format: 'chart'
    type: 'bar' | 'line' | 'pie' | 'scatter'
    data: object
    options?: object
  }
}
```

#### 复杂输出配置示例

```typescript
// 分析报告输出
const analysisReportAnswer = {
  nodeId: 'answerNode_report',
  name: '分析报告输出',
  inputs: [
    {
      key: 'answerText',
      renderTypeList: [FlowNodeInputTypeEnum.reference],
      valueType: WorkflowIOValueTypeEnum.string,
      label: '报告内容',
      description: '完整的分析报告内容'
    }
  ],
  outputs: [
    {
      key: 'answerText',
      label: '格式化报告',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.string
    }
  ],
  config: {
    format: 'markdown',
    template: `
# {{title}} 分析报告

## 📊 概览
{{overview}}

## 🔍 详细分析
{{details}}

## 📈 数据图表
{{charts}}

## 💡 结论与建议
{{recommendations}}

## 📋 附录
- 数据来源：{{dataSource}}
- 分析时间：{{timestamp}}
- 分析师：{{analyst}}
`,
    styling: {
      theme: 'professional',
      codeHighlight: true,
      tableStyle: 'striped',
      headingStyle: 'numbered'
    }
  }
}

// 交互式卡片输出
const interactiveCardAnswer = {
  nodeId: 'answerNode_card',
  name: '交互式卡片',
  config: {
    format: 'card',
    template: {
      layout: 'hero',
      components: [
        {
          type: 'header',
          content: '{{title}}',
          style: 'h1'
        },
        {
          type: 'image',
          src: '{{imageUrl}}',
          alt: '{{imageAlt}}',
          width: '100%',
          height: '200px'
        },
        {
          type: 'content',
          content: '{{description}}',
          style: 'body'
        },
        {
          type: 'metrics',
          data: [
            {label: '评分', value: '{{rating}}/5', icon: 'star'},
            {label: '价格', value: '¥{{price}}', icon: 'money'},
            {label: '库存', value: '{{stock}}件', icon: 'box'}
          ]
        },
        {
          type: 'actions',
          buttons: [
            {
              label: '立即购买',
              action: 'buy',
              style: 'primary',
              icon: 'shopping-cart'
            },
            {
              label: '加入收藏',
              action: 'favorite', 
              style: 'secondary',
              icon: 'heart'
            },
            {
              label: '查看详情',
              action: 'view',
              style: 'outline',
              icon: 'eye'
            }
          ]
        }
      ]
    }
  }
}

// 数据表格输出
const dataTableAnswer = {
  nodeId: 'answerNode_table',
  name: '数据表格输出',
  config: {
    format: 'table',
    features: {
      sortable: true,
      searchable: true,
      pagination: true,
      export: ['csv', 'excel', 'pdf'],
      filters: true
    },
    columns: [
      {
        key: 'id',
        label: 'ID',
        type: 'number',
        width: '80px',
        sortable: true
      },
      {
        key: 'name',
        label: '名称',
        type: 'text',
        searchable: true,
        sortable: true
      },
      {
        key: 'status',
        label: '状态',
        type: 'badge',
        render: (value) => ({
          text: value,
          color: value === 'active' ? 'green' : 'red'
        })
      },
      {
        key: 'createdAt',
        label: '创建时间',
        type: 'datetime',
        format: 'YYYY-MM-DD HH:mm:ss'
      },
      {
        key: 'actions',
        label: '操作',
        type: 'actions',
        buttons: [
          {label: '编辑', action: 'edit', icon: 'edit'},
          {label: '删除', action: 'delete', icon: 'trash', confirm: true}
        ]
      }
    ]
  }
}
```

#### 输出模板系统

```typescript
interface AnswerTemplate {
  // 模板变量系统
  variables: {
    // 系统变量
    system: {
      timestamp: string     // 当前时间戳
      userId: string       // 用户ID
      appName: string      // 应用名称
      version: string      // 版本号
    }
    
    // 工作流变量
    workflow: {
      [key: string]: any   // 工作流中的变量
    }
    
    // 自定义变量
    custom: {
      [key: string]: any   // 用户自定义变量
    }
  }
  
  // 模板函数
  functions: {
    // 文本处理函数
    text: {
      truncate: (text: string, length: number) => string
      capitalize: (text: string) => string
      format: (text: string, ...args: any[]) => string
    }
    
    // 数值处理函数
    number: {
      format: (num: number, decimals?: number) => string
      currency: (num: number, currency?: string) => string
      percentage: (num: number, decimals?: number) => string
    }
    
    // 日期处理函数
    date: {
      format: (date: Date, format: string) => string
      relative: (date: Date) => string
      add: (date: Date, amount: number, unit: string) => Date
    }
    
    // 数组处理函数
    array: {
      join: (arr: any[], separator: string) => string
      map: (arr: any[], transform: Function) => any[]
      filter: (arr: any[], predicate: Function) => any[]
    }
  }
}

// 高级模板示例
const advancedAnswerTemplate = `
# {{workflow.analysisTitle | capitalize}} 

> 生成时间：{{system.timestamp | date.format('YYYY年MM月DD日 HH:mm:ss')}}
> 分析对象：{{workflow.targetName}}

## 🎯 核心指标

{{#each workflow.metrics}}
- **{{this.name}}**: {{this.value | number.format(2)}} {{this.unit}}
  {{#if this.trend}}
  - 趋势：{{this.trend > 0 ? '📈 上升' : '📉 下降'}} {{this.trend | number.percentage}}
  {{/if}}
{{/each}}

## 📊 详细数据

{{workflow.detailData | array.map(function(item) {
  return '- ' + item.label + ': ' + item.value + ' (' + item.change + ')';
}) | array.join('\n')}}

## 💡 分析建议

{{#if workflow.score >= 80}}
✅ **表现优秀**：{{workflow.positivePoints | array.join('、')}}
{{else if workflow.score >= 60}}
⚠️ **需要改进**：{{workflow.improvementAreas | array.join('、')}}
{{else}}
❌ **急需优化**：{{workflow.criticalIssues | array.join('、')}}
{{/if}}

---
*本报告由 {{system.appName}} v{{system.version}} 自动生成*
`
```

### 2. 插件输出节点 (pluginOutput)

**节点标识**: `FlowNodeTypeEnum.pluginOutput`
**模板分类**: `function`
**核心功能**: 定义插件的输出接口和数据格式

#### 插件输出类型定义

```typescript
interface PluginOutputTypes {
  // 基础数据输出
  dataOutput: {
    key: string
    label: string
    description: string
    valueType: WorkflowIOValueTypeEnum
    required?: boolean
    defaultValue?: any
  }
  
  // 文件输出
  fileOutput: {
    key: string
    label: string
    fileType: string[]      // 支持的文件类型
    maxSize?: string        // 最大文件大小
    downloadable?: boolean  // 是否可下载
  }
  
  // 流式输出
  streamOutput: {
    key: string
    label: string
    streamType: 'text' | 'json' | 'binary'
    chunkSize?: number     // 块大小
    timeout?: number       // 超时时间
  }
  
  // 状态输出
  statusOutput: {
    key: string
    label: string
    statusType: 'success' | 'error' | 'warning' | 'info'
    message?: string       // 状态消息
    details?: object       // 详细信息
  }
}
```

#### 插件输出配置示例

```typescript
// 数据分析插件输出
const dataAnalysisPluginOutput = {
  nodeId: 'pluginOutput_analysis',
  name: '数据分析结果输出',
  inputs: [
    {
      key: 'analysisResults',
      renderTypeList: [FlowNodeInputTypeEnum.reference],
      valueType: WorkflowIOValueTypeEnum.object,
      label: '分析结果',
      required: true
    },
    {
      key: 'chartData',
      renderTypeList: [FlowNodeInputTypeEnum.reference],
      valueType: WorkflowIOValueTypeEnum.object,
      label: '图表数据'
    },
    {
      key: 'reportFile',
      renderTypeList: [FlowNodeInputTypeEnum.reference],
      valueType: WorkflowIOValueTypeEnum.string,
      label: '报告文件'
    }
  ],
  outputs: [
    {
      key: 'summary',
      label: '分析摘要',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.string,
      description: '数据分析的核心发现和洞察'
    },
    {
      key: 'metrics',
      label: '关键指标',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.object,
      description: '重要的数值指标和KPI'
    },
    {
      key: 'visualizations',
      label: '可视化图表',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.arrayObject,
      description: '生成的图表和可视化数据'
    },
    {
      key: 'downloadLink',
      label: '报告下载链接',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.string,
      description: '完整报告的下载地址'
    },
    {
      key: 'status',
      label: '处理状态',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.object,
      description: '插件执行状态和结果信息'
    }
  ],
  config: {
    outputFormat: {
      summary: {
        maxLength: 500,
        format: 'markdown'
      },
      metrics: {
        precision: 2,
        unit: 'auto'
      },
      visualizations: {
        formats: ['png', 'svg', 'json'],
        maxWidth: 1200,
        maxHeight: 800
      }
    },
    validation: {
      required: ['summary', 'status'],
      conditionalRequired: {
        'metrics': 'hasNumericalData === true',
        'visualizations': 'generateCharts === true'
      }
    }
  }
}

// 多媒体处理插件输出
const mediaProcessingOutput = {
  nodeId: 'pluginOutput_media',
  name: '多媒体处理输出',
  outputs: [
    {
      key: 'processedMedia',
      label: '处理后的媒体文件',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.arrayString,
      description: '处理完成的媒体文件URL列表'
    },
    {
      key: 'metadata',
      label: '媒体元数据',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.object,
      description: '媒体文件的详细信息'
    },
    {
      key: 'thumbnails',
      label: '缩略图',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.arrayString,
      description: '生成的缩略图URL'
    },
    {
      key: 'processingLog',
      label: '处理日志',
      type: FlowNodeOutputTypeEnum.static,
      valueType: WorkflowIOValueTypeEnum.string,
      description: '详细的处理过程日志'
    }
  ],
  config: {
    fileHandling: {
      supportedFormats: {
        input: ['jpg', 'png', 'gif', 'mp4', 'avi', 'mp3', 'wav'],
        output: ['jpg', 'png', 'webp', 'mp4', 'webm', 'mp3', 'wav']
      },
      maxFileSize: '100MB',
      maxFiles: 10,
      storage: {
        provider: 'minio',
        bucket: 'media-processing',
        retention: '30days'
      }
    }
  }
}
```

## 🔄 数据流与状态管理

### 输入输出数据流模式

```typescript
interface DataFlowPatterns {
  // 单向数据流
  unidirectional: {
    source: NodeOutput
    target: NodeInput
    transform?: (data: any) => any
    validate?: (data: any) => boolean
  }
  
  // 双向数据绑定
  bidirectional: {
    nodeA: NodeInterface
    nodeB: NodeInterface
    sync: 'realtime' | 'onchange' | 'manual'
  }
  
  // 多对多数据聚合
  aggregation: {
    sources: NodeOutput[]
    target: NodeInput
    aggregateFunction: (data: any[]) => any
    updateTrigger: 'any' | 'all' | 'threshold'
  }
  
  // 条件数据路由
  conditional: {
    source: NodeOutput
    routes: Array<{
      condition: string
      target: NodeInput
      transform?: Function
    }>
    defaultRoute?: NodeInput
  }
}
```

### 状态管理机制

```typescript
interface NodeStateManagement {
  // 节点状态
  nodeState: {
    status: 'waiting' | 'running' | 'completed' | 'error' | 'paused'
    progress?: number              // 进度百分比
    startTime?: Date              // 开始时间
    endTime?: Date                // 结束时间
    error?: {
      code: string
      message: string
      details?: any
    }
  }
  
  // 输入状态
  inputState: {
    [key: string]: {
      value: any                  // 当前值
      isValid: boolean           // 是否有效
      isDirty: boolean           // 是否已修改
      errors: string[]           // 验证错误
      lastUpdated: Date          // 最后更新时间
    }
  }
  
  // 输出状态
  outputState: {
    [key: string]: {
      value: any                 // 输出值
      isReady: boolean          // 是否就绪
      consumers: string[]        // 消费者节点
      lastGenerated: Date       // 最后生成时间
    }
  }
  
  // 交互状态
  interactionState: {
    isWaitingForUser: boolean    // 是否等待用户交互
    userResponse?: any           // 用户响应
    timeout?: number            // 超时时间
    retryCount?: number         // 重试次数
  }
}
```

## 🎨 UI/UX 设计模式

### 响应式布局

```typescript
interface ResponsiveLayoutConfig {
  // 断点设置
  breakpoints: {
    mobile: '0-768px'
    tablet: '769-1024px'
    desktop: '1025px+'
  }
  
  // 布局适配
  layouts: {
    mobile: {
      columns: 1
      spacing: 'compact'
      fontSize: 'small'
      buttonSize: 'large'
    }
    tablet: {
      columns: 2
      spacing: 'normal'
      fontSize: 'medium'
      buttonSize: 'medium'
    }
    desktop: {
      columns: 3
      spacing: 'comfortable'
      fontSize: 'large'
      buttonSize: 'normal'
    }
  }
}
```

### 无障碍访问支持

```typescript
interface AccessibilityFeatures {
  // 键盘导航
  keyboardNavigation: {
    tabOrder: string[]           // Tab键顺序
    shortcuts: {
      [key: string]: string     // 快捷键映射
    }
    focusManagement: boolean     // 焦点管理
  }
  
  // 屏幕阅读器支持
  screenReader: {
    ariaLabels: {
      [key: string]: string     // ARIA标签
    }
    landmarks: string[]          // 页面地标
    announcements: string[]      // 状态公告
  }
  
  // 视觉辅助
  visualAids: {
    highContrast: boolean        // 高对比度模式
    largeText: boolean          // 大字体模式
    colorBlindSupport: boolean   // 色盲支持
  }
}
```

---

输入输出节点作为 FastGPT 工作流编排系统的**用户界面层**，承担着至关重要的交互职责。通过精心设计的节点类型、灵活的配置选项和强大的数据处理能力，为用户提供了丰富而直观的交互体验，是构建用户友好AI应用的基础。