---
title: 自主代理（Autonomous Agents）
tags: [LLM,ChatGPT,GPT4,GPT3.5,语言模型,大语言模型,Prompt,Prompt工程,Cypher]
author: Yc-Ma
show_author_profile: true
key: 2023-05-16-自主代理（Autonomous Agents）
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

# 自主代理（Autonomous Agents）

## 简介

>&emsp;&emsp;自主代理（Autonomous Agents）是由人工智能驱动的程序，当给定目标时，它们能够为自己创建任务，完成任务，创建新任务，重新确定任务列表的优先级，完成新的顶级任务，循环（递归的思想）直到达到目标。

>&emsp;&emsp;可以给自主代理（Autonomous Agents）一个任务，比如发一个关于自主代理（Autonomous Agents）研究最新进展的邮件。他会先去理解分解这个任务目标，然后设定实施计划以及这几个计划的优先级，同时去辩证『冷静』的反思计划有没有漏洞，并将反思应用到执行过程中，然后就是自己不断的去换着关键词搜索总结最近的报道文章，然后是汇总、反思，看看有没有什么遗漏，最后组织成适合邮件的语言格式自动发送。

![自主智能体是如何工作](https://nimg.ws.126.net/?url=http%3A%2F%2Fdingyue.ws.126.net%2F2023%2F0424%2Fb98c9612j00rtm89r000yd200iw00d7g00g200b7.jpg&thumbnail=660x2147483647&quality=80&type=jpg)

## 核心技术

1. LLMs（大语言模型）：LLMs是最核心的能力，无论推理还是问答以及后续的Prompt工程，都强依赖于LLM的能力。

2. Memory（上下文记忆）：缓存任务的中间状态、保存会话的上下文信息，用于后续任务的处理。

3. Prompt Engineering（提示工程）：提供给LLMs使用的提示信息。

4. Tools（工具箱）：自主代理（Autonomous Agents）用来处理子任务的插件。

5. 递归的思想：使自主代理（Autonomous Agents）完全自主的核心设计思想。

![AutoGPTs building AutoGPTs building AutoGPTs...](https://camo.githubusercontent.com/8226dd8023ed52d438c66791ae849051fa7c8e1f874bb728ca9b2c3dbb1cd64b/68747470733a2f2f6170692e737461722d686973746f72792e636f6d2f7376673f7265706f733d546f72616e74756c696e6f2f6175746f2d67707426747970653d44617465)

## 让 AI 使用工具的案例

>&emsp;&emsp;下面是基于`LangChain`实现的`AutoGPT`类程序，可以让智能代理程序完成使用`搜索引擎`和`本地知识库`等工具的功能。

### 使用搜索引擎

>&emsp;&emsp;下面的案例展示了一个自主代理（Autonomous Agents）使用 `GPT3.5` 和工具箱（Tools）解决问题的运行过程。在这个案例中定义的可以使用的工具为`数学计算工具（llm-math）`和`Google搜索工具（serpapi）`，`Agents`拿到问题后拆分为下面四个子任务：
>&emsp;&emsp; 1. 找到经理名称，Action Input: "易方达基金总经理"。
>&emsp;&emsp; 2. 找到经理年龄，Action Input: "易方达基金总经理刘晓艳 年龄"。
>&emsp;&emsp; 3. 年龄运行数学计算，Action Input: 42^0.76。
>&emsp;&emsp; 4. 整合各个子任务结果信息并生成答案。
>前三个子任务逐个解决后，最后一个任务负责将答案进行整合输出一个完整的回复。

<br>

>**QA：易方达基金的总经理是谁？他今年的年龄是多少？他年龄的0.76次方是多少？**

>AGENTS：
**Entering new AgentExecutor chain...**
<font color=#008000 > I need to find out the name of the manager and his age</font>
<font color=#008000 >Action: Search</font>
<font color=#008000 >Action Input: "易方达基金总经理"</font>
Observation: <font color=#BD9A28 >易方达基金总经理刘晓艳：以专业精神与投资者长期长情常相伴- 媒体报道- 易方达基金管理有限公司</font>
Thought: <font color=#008000 >I need to find out the age of the manager</font>

><font color=#008000 >Action: Search</font>
<font color=#008000 >Action Input: "易方达基金总经理刘晓艳 年龄"</font>
Observation: <font color=#BD9A28 >用总裁助理兼市场部总经理刘晓艳(女)的话说：她是一个很公正、很铁面无私的人，很适合做督察长的职务。 张南的台面上摆着三个电脑显示器，据她介绍， ...</font>
Thought: <font color=#008000 >I need to calculate the age to the 0.76 power</font>

><font color=#008000 >Action: Calculator</font>
<font color=#008000 >Action Input: 42^0.76</font>
Observation: <font color=#008DA3 >Answer: 17.126533197047873</font>
Thought: <font color=#008000 >I now know the final answer</font>

><font color=#008000 >Final Answer: 易方达基金总经理刘晓艳今年42岁，他年龄的0.76次方是17.126533197047873。</font>

>**Finished chain.**

>备注：需要注意的是上面的回答，易方达基金总经理刘晓艳的年龄得到的结果是错误的【这种问题也可以结合本地知识库来优化】。

### 使用知识库

>&emsp;&emsp;下面的案例展示了一个自主代理（Autonomous Agents）使用 `GPT3.5` 和工具箱（Tools）解决问题的运行过程。在这个案例中定义的可以使用的工具为`知识图谱查询工具（cypher_search）`，`Agents`拿到问题后拆分为下面三个子任务：
>&emsp;&emsp; 1. 生成知识图谱查询代码Cypher。
>&emsp;&emsp; 2. 从知识图谱获取相关信息。
>&emsp;&emsp; 3. 整合结果信息并生成答案。

<br>

>**QA：嘉实物流产业基金，定位属于哪类基金？**

>**Entering new AgentExecutor chain...**
<font color=#008000 >{
"action": "Cypher search",
"action_input": "嘉实物流产业基金属于哪类基金？"
}</font>

>**Entering new LLMCypherGraphChain chain...**
<font color=#008000 >Generated Cypher statement:</font>
<font color=#008DA3 >MATCH p0=(n2:投资类型)<-[r1:投资类型]-(n0:基金产品)-[r0:基金中文简称]->(n1:基金中文简称)-[r2:别名]->(n3:基金中文简称别名)
WHERE n3.value='嘉实物流产业基金' RETURN n2.value AS value;</font>

>**Finished chain.**

>Observation: <font color=#008DA3 >投资类型：股票型</font>
Thought:<font color=#008000 >{
"action": "Final Answer",
"action_input": "嘉实物流产业基金投资类型属于股票型。"
}</font>

><font color=#008000 >Final Answer: 嘉实物流产业基金投资类型属于股票型。</font>

>**Finished chain.**



