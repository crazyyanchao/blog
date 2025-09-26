---
title: DeepSearch资料
tags: [LLM]
author: Yc-Ma
show_author_profile: true
key: 2025-09-05-DeepSearch资料
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

# 不同任务下 GraphRAG 效果对比

本文档对比分析了当前主流的GraphRAG相关项目，包括传统的GraphRAG实现和最新的HiRAG层次化知识检索增强生成框架。这些项目在综述总结、跨文档整合、易用性等方面各有特色，为不同场景下的知识图谱构建和问答提供了多样化的解决方案。

> GraphRAG 框架初步设计：[分层知识图谱](../core/README.md)

## 项目对比表

|        项目         | **综述总结能力** | **跨文档主题整合** | **易用性/开源支持** | **Token 成本** | **核心优化方向** | **适用场景** |
| ------------------- | --------------- | ------------------ | ------------------- | -------------- | ---------------- | ------------ |
| **Microsoft GraphRAG** | ✅ 强项 | ✅ 社区摘要 + 全局查询 | ✅ 高 | ✅ 已优化 | 全局总结、主题提炼、跨文档语义聚类 | 综述、报告、趋势分析 |
| **Youtu-GraphRAG** | ❌ 非核心优势 | ⚠️ 需手动设计 | ⚠️ 低 | ✅ 更优 | 多跳问答、复杂推理、匿名抗泄露、跨领域泛化 | 多跳问答、推理、匿名任务 |
| **KAG** | ⚠️ 中等（依赖图谱 schema 设计） | ✅ 自动 schema 对齐 + 子图融合 | ✅ 提供 docker-compose 一键起，文档较全 | ✅ 支持本地 7B/14B 模型，成本可控 | 领域知识图谱自动构建、图-文对齐、本地模型友好 | 金融、医疗、政务等领域的可解释 KgQA |
| **R2R** | ❌ 非核心优势 | ✅ 重排序 + 思维链，可跨段推理 | ✅ pip 安装即可，10 行代码跑通 | ✅ 可自选小型 Embedding+SLM，成本最低 | Retrieval→Ranking→Reason 全链路可插拔、轻量级部署 | 快速原型、实验对比、低资源场景下的可解释问答 |
| **HiRAG** | ✅ 强项（层次化知识结构） | ✅ 层次化知识组织 + 多维度检索 | ✅ 高（MIT开源，文档完善） | ✅ 已优化（层次化检索减少冗余） | 层次化知识组织、多维度评估、多LLM支持 | 复杂知识检索、多领域问答、高精度推理任务 |

## 项目详情

|       项目名        |                        链接                        |                                                       描述                                                        |
| ------------------ | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Microsoft GraphRAG | https://github.com/microsoft/graphrag             | 微软开源的GraphRAG实现，支持社区摘要和全局查询                                                                        |
| Youtu-GraphRAG     | https://github.com/TencentCloudADP/youtu-graphrag | 腾讯优图实验室的GraphRAG实现，专注于多跳问答和推理任务                                                                |
| KAG                | https://github.com/OpenSPG/KAG                    | 蚂蚁集团开源，面向领域知识图谱构建与问答，支持本地模型与知识对齐                                                        |
| R2R                | https://github.com/SciPhi-AI/R2R                  | 轻量级"RAG-to-Reason"框架，主打 Retrieval→Ranking→Reason 三步流程，可插拔 LLM/Embedding，内置自研重排序与思维链提示模板 |
| HiRAG          | https://github.com/hhy-huang/HiRAG                | 基于层次化知识的检索增强生成框架，在多个基准测试中显著优于传统GraphRAG方法，支持多LLM提供商 |


# Deep Research类项目对比

> Deep Research 框架初步设计：LLM【tongyi-deepresearch-30b-a3b】 + LangGraph【ReAct和IterResearch】=> Output

## 项目对比表

|        项目         | **多源搜索能力** | **LLM集成** | **递归任务分解** | **多代理协作** | **易用性/开源支持** | **工具生态** | **核心优化方向** | **适用场景** |
| ------------------- | --------------- | ----------- | --------------- | -------------- | ------------------- | ------------ | ---------------- | ------------ |
| **deep-research** | ✅ 强项 | ✅ 多LLM支持 | ❌ 单次处理 | ❌ 无 | ✅ 高 | ✅ 丰富 | 搜索引擎查询、网页提取、LLM分析 | 全面深度研究、报告生成 |
| **langchain-ai/open_deep_research** | ✅ 强项 | ✅ 多模型提供商 | ❌ 单次处理 | ❌ 无 | ✅ 高（LangChain生态） | ✅ MCP服务器 | 可配置代理、多工具集成 | 企业级深度研究 |
| **togethercomputer/open_deep_research** | ✅ 强项 | ✅ 多跳推理 | ❌ 单次处理 | ❌ 无 | ✅ 高 | ✅ 丰富 | 多跳推理、引用生成、人类研究过程模拟 | 复杂主题研究、学术分析 |
| **open-deep-research** (nickscamara) | ✅ 强项 | ✅ Firecrawl集成 | ❌ 单次处理 | ❌ 无 | ⚠️ 中等 | ⚠️ 基础 | 网络数据提取、AI推理 | 网络数据研究 |
| **deep-searcher** | ❌ 私有数据 | ✅ Python生态 | ❌ 单次处理 | ❌ 无 | ⚠️ 中等 | ⚠️ 基础 | 私有数据推理、本地搜索 | 企业内部研究 |
| **node-DeepResearch** | ✅ 持续搜索 | ✅ 多LLM | ❌ 单次处理 | ❌ 无 | ⚠️ 中等 | ⚠️ 基础 | 持续搜索、token预算管理 | 长期研究任务 |
| **browser-use** | ❌ 网站自动化 | ✅ AI代理 | ❌ 单次处理 | ❌ 无 | ✅ 高 | ✅ 丰富 | 网站自动化、AI代理访问 | 网站任务自动化 |
| **deepsearch-toolkit** | ❌ 平台交互 | ❌ 无 | ❌ 单次处理 | ❌ 无 | ⚠️ 专业 | ⚠️ 专业 | 知识探索、平台集成 | 科研平台集成 |
| **deep-research (u14app)** | ✅ 强项 | ✅ 多LLM | ❌ 单次处理 | ❌ 无 | ✅ 高 | ✅ SSE/MCP | SSE API、MCP服务器支持 | 实时研究、API集成 |
| **DeepSearch** | ✅ 多源搜索 | ✅ AI增强 | ❌ 单次处理 | ❌ 无 | ✅ 简单部署 | ⚠️ 基础 | 多源搜索、AI摘要、敏感词过滤 | Web搜索、信息聚合 |
| **MetaSearch** | ⚠️ 基础 | ✅ 多模态 | ❌ 单次处理 | ❌ 无 | ✅ 学习友好 | ⚠️ 基础 | 模块化RAG、多模态检索 | 学习RAG、快速原型 |
| **OpenMatch** | ❌ 检索优化 | ❌ 无 | ❌ 单次处理 | ❌ 无 | ✅ 学术研究 | ❌ 无 | 神经检索、传统检索模块 | 学术研究、定制检索 |
| **ROMA** | ✅ 强项 | ✅ 多LLM | ✅ 强项 | ✅ 核心功能 | ✅ 高（Apache-2.0） | ✅ 丰富 | 递归分解、多代理协作、企业级安全 | 复杂任务自动化、研究分析 |
| **Tongyi DeepResearch** | ✅ 强项 | ✅ 30B-A3B模型 | ✅ 端到端RL | ❌ 单次处理 | ✅ 高（Apache-2.0） | ✅ 丰富工具生态 | 大规模持续预训练、强化学习优化、多模式推理 | 长程深度研究、复杂信息检索、学术研究 |

## 项目详情

### 核心深度研究代理

|       项目名        |                        链接                        |                                                       描述                                                        |
| ------------------ | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| deep-research | https://github.com/treeleaves30760/deep-research | 全面的深度研究代理，结合搜索引擎查询、网页内容提取和LLM分析，生成详细报告，支持可定制的广度和深度 |
| langchain-ai/open_deep_research | https://github.com/langchain-ai/open_deep_research | LangChain官方深度研究代理，简单可配置，支持多种模型提供商、搜索工具和MCP服务器，性能与主流代理相当 |
| togethercomputer/open_deep_research | https://github.com/togethercomputer/open_deep_research | Together AI的深度研究工作流，提供多跳推理的复杂主题研究，生成全面有引用的内容，模拟人类研究过程 |
| deep-research (u14app) | https://github.com/u14app/deep-research | 支持任何LLM的深度研究代理，提供SSE API和MCP服务器支持，适合实时研究和API集成场景 |
| Tongyi DeepResearch | https://github.com/Alibaba-NLP/DeepResearch | 阿里巴巴通义实验室开发的领先开源深度研究代理，30.5B总参数量，每token仅激活3.3B参数，专为长程深度信息检索任务设计，支持ReAct和IterResearch两种推理模式 |

### 多源搜索和聚合工具

|       项目名        |                        链接                        |                                                       描述                                                        |
| ------------------ | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| DeepSearch | https://github.com/lokyshin/DeepSearch | 基于SearxNG的多源搜索引擎聚合工具，集成AI增强模块，支持敏感词过滤和Markdown渲染 |
| MetaSearch | https://github.com/Marstaos/MetaSearch | 教学项目，帮助学习如何运用大语言模型接口搭建RAG系统，提供模块化开发框架 |
| OpenMatch | https://arxiv.org/abs/2102.00166 | 用于神经信息检索研究的开源库，提供自包含的神经和传统信息检索模块 |

### 工作流和集成方案

|       项目名        |                        链接                        |                                                       描述                                                        |
| ------------------ | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Open-Deep-Research-workflow-on-Dify | https://github.com/hkxiaoyao/Open-Deep-Research-workflow-on-Dify | Deep Researcher工作流在Dify平台的复现方案，提供完整的工作流程示例和集成指导 |
| browser-use | https://github.com/browser-use/browser-use | 使网站对AI代理可访问的工具，轻松实现在线任务自动化，为深度研究提供网站交互能力 |
| deepsearch-toolkit | https://github.com/DS4SD/deepsearch-toolkit | 与Deep Search平台交互的专业工具包，专注于新知识探索和发现，适合科研平台集成 |

### 特定技术实现

|       项目名        |                        链接                        |                                                       描述                                                        |
| ------------------ | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| open-deep-research | https://github.com/nickscamara/open-deep-research | 开源深度研究克隆，使用Firecrawl提取大量网络数据并进行推理的AI代理，专注于网络数据研究 |
| deep-searcher | https://github.com/zilliztech/deep-searcher | 开源深度研究替代方案，专门用于在私有数据上进行推理和搜索，使用Python编写，适合企业内部使用 |
| node-DeepResearch | https://github.com/jina-ai/node-DeepResearch | 持续搜索、阅读网页、推理的Node.js实现，直到找到答案或超出token预算，适合长期研究任务 |
| ROMA | https://github.com/sentient-agi/ROMA | 递归开放元代理框架，构建高性能多代理系统，支持E2B沙箱、S3集成、WebSocket通信 |

