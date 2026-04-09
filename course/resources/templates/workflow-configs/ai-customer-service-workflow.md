---
title: AI客服工作流配置模板
type: template
tags: [模板, 工作流, AI客服, YAML]
---

# AI客服工作流配置模板

> 智能客服场景的标准工作流配置

---

## 工作流基本信息

| 字段 | 内容 |
|------|------|
| 名称 | AI客服-{$TEAM_NAME} |
| 描述 | 智能客服自动回复+转人工工作流 |
| 适用行业 | 通用 |
| 复杂度 | 中等 |

---

## 工作流结构图

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ 消息入口 │ → │ 意图识别 │ → │ 知识库  │ → │ AI回复  │
│ (绿色)   │    │ (蓝色)  │    │ 检索   │    │ (蓝色)  │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
                                       ↓
                               ┌─────────┐    ┌─────────┐
                               │ 未识别   │ → │ 转人工  │
                               │ (蓝色)   │    │ (紫色)  │
                               └─────────┘    └─────────┘
```

---

## 完整配置（YAML格式）

```yaml
workflow:
  name: AI客服-{$TEAM_NAME}
  version: 1.0
  description: 智能客服自动回复+转人工工作流

# 触发配置
trigger:
  type: message
  channel: feishu
  condition: always

# 节点配置
nodes:
  - id: node_1
    name: 消息入口
    type: trigger
    color: green
    config:
      channel: feishu
      listen_events: [im.message.receive_v1]
      
  - id: node_2
    name: 意图识别
    type: process
    color: blue
    config:
      skill: intent_detection
      intents:
        - name: 使用咨询
          keywords: [怎么, 如何, 设置, 使用]
          confidence_threshold: 0.8
        - name: 故障排查
          keywords: [无法, 坏了, 故障, 问题]
          confidence_threshold: 0.8
        - name: 售后政策
          keywords: [保修, 退换, 售后, 维修]
          confidence_threshold: 0.8
        - name: 订单查询
          keywords: [订单, 物流, 快递, 到哪了]
          confidence_threshold: 0.8
      default_intent: 转人工
      
  - id: node_3
    name: 知识库检索
    type: process
    color: blue
    config:
      skill: knowledge_base
      knowledge_base_id: customer_service_kb
      top_k: 3
      match_threshold: 0.7
      
  - id: node_4
    name: AI回复
    type: process
    color: blue
    config:
      skill: llm
      model: gpt-4
      prompt_template: |
        用户问题：{{user_input}}
        意图类型：{{intent}}
        检索结果：{{retrieved_content}}
        
        请根据以上信息，给出专业的回复。
        回复要求：
        1. 简洁明了
        2. 提供具体解决方案
        3. 如需转人工，明确告知用户
      
  - id: node_5
    name: 发送回复
    type: output
    color: purple
    config:
      skill: feishu_message
      channel: feishu
      message_type: text
      
  - id: node_6
    name: 转人工
    type: output
    color: purple
    config:
      skill: ticket_create
      ticket_system: zendesk
      priority: normal
      assign_to: customer_service_team
      
  - id: node_7
    name: 通知用户
    type: output
    color: purple
    config:
      skill: feishu_message
      message_template: |
        您的问题已转交给人工客服，请稍候...
      
  - id: node_8
    name: 满意度调查
    type: output
    color: purple
    config:
      skill: feishu_message
      message_template: |
        请问您的问题是否已解决？
        👍 已解决
        👎 未解决

# 连接配置
connections:
  - from: node_1
    to: node_2
    
  - from: node_2
    to: node_3
    condition: intent识别的置信度 >= 0.8
    
  - from: node_3
    to: node_4
    
  - from: node_4
    to: node_5
    
  - from: node_2
    to: node_6
    condition: intent识别的置信度 < 0.8 OR intent == 转人工
    
  - from: node_6
    to: node_7
    
  - from: node_5
    to: node_8

# 异常处理
error_handling:
  knowledge_not_found:
    action: fallback_to_ai
    message: 未找到相关答案，让我来帮您解答...
    
  api_error:
    action: retry
    max_retries: 3
    retry_interval: 1s
    
  timeout:
    action: transfer_to_human
    message: 系统繁忙，请稍后再试或转人工服务
```

---

## 配置说明

### 关键参数说明

| 参数 | 说明 | 推荐值 |
|------|------|-------|
| confidence_threshold | 意图识别置信度阈值 | 0.8 |
| top_k | 知识库检索返回条数 | 3 |
| match_threshold | 知识库匹配阈值 | 0.7 |
| max_retries | API错误重试次数 | 3 |

### 自定义项

使用前需要修改以下内容：

```yaml
# 1. 修改工作流名称
name: AI客服-{$TEAM_NAME}  # 替换为你的团队名

# 2. 配置知识库ID
knowledge_base_id: customer_service_kb  # 替换为你的知识库ID

# 3. 配置渠道
channel: feishu  # 可选：feishu, wecom, dingtalk

# 4. 配置AI模型
model: gpt-4  # 可选：gpt-4, gpt-3.5-turbo, claude
```

---

## 知识库结构建议

```
客户服务中心知识库/
├── 使用咨询/
│   ├── 产品使用指南/
│   ├── 基础操作教程/
│   └── 常见问题FAQ/
├── 故障排查/
│   ├── 网络问题/
│   ├── 设备问题/
│   └── 软件问题/
├── 售后政策/
│   ├── 保修政策/
│   ├── 退换货流程/
│   └── 维修服务/
└── 订单查询/
    ├── 物流查询/
    ├── 订单修改/
    └── 发票开具/
```

---

## 部署检查清单

- [ ] 已创建知识库并导入内容
- [ ] 已配置飞书通道
- [ ] 已配置意图识别
- [ ] 已测试各分支流程
- [ ] 已配置异常处理
- [ ] 已测试转人工流程

---

## 相关资源

- [[../case-studies/案例集|案例集]]
- [[../../modules/m4-commercial-deployment|M4 商业落地]]
