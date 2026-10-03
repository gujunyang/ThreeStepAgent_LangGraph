# ThreeStepAgent_LangGraph

基于 [LangGraph](https://github.com/langchain-ai/langgraph) 构建的三步式 Agent 实现。

## 简介

本项目展示了如何使用 LangGraph 组织一个三步式（Three Step）的智能体流程，包括状态管理、节点编排与工具调用。

## 特性

- 使用 LangGraph 定义有状态的 Agent 工作流
- 三步式流程编排
- 可扩展的节点与工具设计

## 环境要求

- Python 3.10+
- LangGraph
- LangChain

## 安装

```bash
git clone https://github.com/gujunyang/git.git
cd ThreeStepAgent_LangGraph
pip install -r requirements.txt
```

## 使用

复制 `.env.example` 为 `.env` 并填入 API Key，然后运行：

```bash
python agent.py
```

## 项目结构

```
ThreeStepAgent_LangGraph/
├── README.md
└── ...
```

## 许可证

MIT
