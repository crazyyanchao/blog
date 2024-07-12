---
title: 使用Neo4j和LangChain实现“从本地到全局”的GraphRAG
tags: [Neo4j,LangChain,GraphRAG]
author: Yc-Ma
show_author_profile: true
key: 2024-07-11-使用Neo4j和LangChain实现“从本地到全局”的GraphRAG
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

# 使用Neo4j和LangChain实现“从本地到全局”的GraphRAG

&emsp;**GraphRAG**是一种基于知识图谱的检索增强技术。它使用多来源数据构建图模型的知识表达，将实体和关系之间的联系以图的形式展示，然后利用大语言模型进行检索增强。这种方法能更高效准确地检索相关信息，并为**LLM**生成响应提供更好的上下文。微软和领英的技术人员已经科学的验证了这种技术相较于基线 RAG 的优势，并发表了相关论文。

- [LinkedIn 最新研究：图+向量数据库，客服解答时间缩短 64%](https://xie.infoq.cn/article/0940b8a7d0746b0b902cb46a0)
- [From Local to Global: A Graph RAG Approach to Query-Focused Summarization](https://arxiv.org/abs/2404.16130)

&emsp;目前微软开源的[Microsoft GraphRAG](https://github.com/microsoft/graphrag)项目支持对文档进行分块、向量化、抽取实体和关系然后保存为本地知识图谱文件，暂不支持将数据存储到 Neo4j。Noej4 的开源项目[LLM Graph Builder](https://github.com/neo4j-labs/llm-graph-builder)分为前后端，支持上传图片、文档等资料生成知识图谱然后进行问答，暂时没有做社区摘要和实体向量化功能。本篇文章是基于这些项目基础上，使用 Neo4j 和 Langchain 单独实现的“从本地到全局”的 GraphRAG，可以理解为将 Microsoft GraphRAG 社区摘要的功能单独添加到 LLM Graph Builder 项目。另外关于 LLM Graph Builder 是一个新项目，官网开发者也在快速迭代正计划集成 GraphRAG 的各个方面功能，未来应该会有分层聚类生成社区摘要、实体向量化等功能。

![LLM Graph Builder Issues](https://cdn.jsdelivr.net/gh/filess/img8@main/2024/07/11/1720686343384-eb64a663-592f-43a5-b30b-23b71127090c.jpg)

![开发者回复](https://cdn.jsdelivr.net/gh/filess/img0@main/2024/07/11/1720686328327-4df79e86-0b7a-492d-9c1e-3fa837730c8f.jpg)

&emsp;为了提高基于检索增强生成（RAG）技术的准确性，可以结合使用文本抽取、图分析、大型语言模型（LLM）的提示词技术和信息摘要能力。本篇文章翻译整理原文：[Implementing ‘From Local to Global’ GraphRAG with Neo4j and LangChain: Constructing the Graph](https://medium.com/neo4j/implementing-from-local-to-global-graphrag-with-neo4j-and-langchain-constructing-the-graph-73924cc5bab4)。

&emsp;在这篇博文中，我们将深入探讨微软的“[From Local to Global GraphRAG](https://arxiv.org/abs/2404.16130)”的文章以及具体实现。我们将主要介绍知识图谱的构建和摘要部分，基于 GraphRAG 的检索应用在下一篇博客文章会具体介绍。微软的研究人员同时提供了[Microsoft GraphRAG](https://www.microsoft.com/en-us/research/project/graphrag/)的项目页面。

&emsp;上面提到的文章中采用的方法非常有趣。据我所知，它包括使用知识图作为管道中的一个步骤，用于压缩和组合来自多个来源的信息。从文本中提取实体和关系并不是什么新鲜事。然而，作者引入了一个新颖的想法(至少对我来说)，将压缩的图结构和信息总结为自然语言文本。管道从文档中的输入文本开始，然后对其进行处理以生成图。然后，图被转换回自然语言文本，生成的文本包含关于特定实体或图社区的浓缩信息，这些信息以前分布在多个文档中。

![Microsoft在GraphRAG论文中实现的高级索引管道 — 图片来自论文作者](https://cdn.jsdelivr.net/gh/filess/img16@main/2024/07/11/1720681046971-d751a68d-7891-4592-ba24-fde4d249be6d.png)

&emsp;在非常高的级别上，GraphRAG 管道的输入是包含各种信息的源文档。使用 LLM 处理文档，以提取有关出现在论文中的实体及其关系的结构化信息。然后使用提取的结构化信息构建知识图。

&emsp;使用知识图谱数据表示的优点是它可以快速直接地组合来自多个文档或数据源的有关特定实体的信息。如前所述，知识图谱并不是唯一的数据表示。构建知识图谱后，他们使用图谱算法和 LLM 提示的组合来生成知识图谱中实体社区的自然语言摘要。

&emsp;这些摘要包含了针对特定实体和社区的跨多个数据源和文档的浓缩信息。为了更详细地了解该流程，我们可以参考原始论文中提供的分步描述。

![流程中的步骤 — 图片来自GraphRAG 论文，根据 CC BY 4.0 许可](https://cdn.jsdelivr.net/gh/filess/img19@main/2024/07/11/1720681457617-02da9cd0-81d3-4606-8edc-d6e642f45848.png)

&emsp;下面是使用 Neo4j 和 LangChain 复现 Microsoft GraphRAG 方法的具体步骤，[本文涉及的实验代码](https://github.com/tomasonjo/blogs/blob/master/llm/ms_graphrag.ipynb)。

## 重点摘要

### 索引—图生成

- **源文档到文本块**：源文档被分割成更小的文本块进行处理。
- **文本块到元素实例**：分析每个文本块以提取实体和关系，生成表示这些元素的元组列表。
- **元素实例到元素摘要**：提取的实体和关系由 LLM 总结为每个元素的描述性文本块。
- **元素摘要到图社区**：这些实体摘要形成一个图，然后使用[Leiden](https://neo4j.com/docs/graph-data-science/current/algorithms/leiden/)等算法将其划分为社区，以实现分层结构。
- **从图社区到社区摘要**：使用 LLM 生成每个社区的摘要，以了解数据集的全局主题结构和语义。

### 检索—回答

- **社区摘要到全局答案**：社区摘要用于通过生成中间答案来回答用户查询，然后将其汇总为最终的全局答案。

## 设置 Neo4j 环境

&emsp;我们将使用 Neo4j 作为底层图形存储。最简单的入门方法是使用[Neo4j Sandbox](https://sandbox.neo4j.com/onboarding?usecase=blank-sandbox)的免费实例，它提供安装了 Graph Data Science 插件的 Neo4j 数据库的云实例。或者，您可以通过下载 Neo4j Desktop 应用程序并创建本地数据库实例来设置 Neo4j 数据库的本地实例。如果您使用的是本地版本，请确保同时安装 APOC 和 GDS 插件。对于生产设置，您可以使用付费的托管 AuraDS（数据科学）实例，它提供 GDS 插件。

&emsp;我们首先创建一个[Neo4jGraph](https://python.langchain.com/v0.2/docs/integrations/graphs/neo4j_cypher/)实例，这是我们添加到 LangChain 的便利包装器：

```python
from langchain_community.graphs import Neo4jGraph

os.environ["NEO4J_URI"] = "bolt://44.202.208.177:7687"
os.environ["NEO4J_USERNAME"] = "neo4j"
os.environ["NEO4J_PASSWORD"] = "mast-codes-trails"

graph = Neo4jGraph(refresh_schema=False)
```

## 数据集

&emsp;我们将使用我之前使用 Diffbot 的 API 创建的新闻文章数据集。我已将其上传到我的 GitHub 以便于重复使用：

```python
news = pd.read_csv(
    "https://raw.githubusercontent.com/tomasonjo/blog-datasets/main/news_articles.csv"
)
news["tokens"] = [
    num_tokens_from_string(f"{row['title']} {row['text']}")
    for i, row in news.iterrows()
]
news.head()
```

检查数据集中的前几行。

![样例数据](https://cdn.jsdelivr.net/gh/filess/img10@main/2024/07/11/1720687126978-9886d576-f193-4d35-aec5-44f4ea85a74e.png)

使用 tiktoken 库来获取文章的标题和正文，计算 Token 数量。

## 文本分块

&emsp;文本分块步骤至关重要，对后续结果有重大影响。论文作者发现，使用较小的文本块可以整体提取更多实体。

![根据文本块大小提取实体的数量 — 图片来自GraphRAG 论文，根据 CC BY 4.0 许可](https://cdn.jsdelivr.net/gh/filess/img13@main/2024/07/11/1720687312069-6371db6d-3c54-40ad-bb6b-59d0844d4e75.png)

&emsp;如您所见，使用 2,400 个标记的文本块会比使用 600 个标记时提取的实体更少。此外，他们还发现 LLM 可能无法在第一次运行时提取所有实体。在这种情况下，他们引入了一种启发式方法来多次执行提取。我们将在下一节中进一步讨论这一点。

然而，总是有权衡的。使用较小的文本块可能会导致丢失文档中特定实体的上下文和共指。例如，如果文档在不同的句子中提到“约翰”和“他”，将文本分成较小的块可能会让人不清楚“他”指的是约翰。一些共指问题可以使用重叠文本分块策略来解决，但不是全部。

让我们检查一下文章文本的大小：

```python
sns.histplot(news["tokens"], kde=False)
plt.title('Distribution of chunk sizes')
plt.xlabel('Token count')
plt.ylabel('Frequency')
plt.show()
```

![](https://cdn.jsdelivr.net/gh/filess/img11@main/2024/07/11/1720687398855-1ee9c170-1c87-4fd3-9036-2360bcb8a903.png)

文章标记计数的分布大致呈正态分布，峰值约为 400 个标记。块的频率逐渐增加到这个峰值，然后对称地减少，这表明大多数文本块都在 400 个标记附近。

由于这种分布，我们不会在此处执行任何文本分块以避免共指问题。默认情况下，GraphRAG 项目使用 300 个标记的[块大小](https://github.com/microsoft/graphrag/blob/main/docsite/posts/config/env_vars.md#data-chunking)，其中 100 个标记重叠。

## 提取节点和关系

下一步是从文本块构建知识。对于此用例，我们使用 LLM 从文本中提取节点和关系形式的结构化信息。您可以检查作者在论文中使用的[LLM 提示](https://github.com/microsoft/graphrag/blob/main/graphrag/prompt_tune/prompt/entity_relationship.py#L6)。他们有 LLM 提示，我们可以根据需要预定义节点标签，但默认情况下，这是可选的。此外，原始文档中提取的关系实际上没有类型，只有描述。我想这种选择的原因是允许 LLM 提取和保留更丰富、更细微的信息作为关系。但是，如果没有关系类型规范（描述可以进入属性），很难拥有干净的知识图谱。

在我们的实现中，我们将使用 LangChain 库中提供的 LLMGraphTransformer。[LLMGraphTransformer](https://python.langchain.com/v0.1/docs/use_cases/graph/constructing/)不是像本文中的实现那样使用纯提示工程，而是使用内置函数调用支持来提取结构化信息（LangChain 中的结构化输出 LLM）。您可以检查[系统提示](https://github.com/langchain-ai/langchain/blob/master/libs/experimental/langchain_experimental/graph_transformers/llm.py#L69)：

```python
from langchain_experimental.graph_transformers import LLMGraphTransformer
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(temperature=0, model_name="gpt-4o")

llm_transformer = LLMGraphTransformer(
  llm=llm,
  node_properties=["description"],
  relationship_properties=["description"]
)

def process_text(text: str) -> List[GraphDocument]:
    doc = Document(page_content=text)
    return llm_transformer.convert_to_graph_documents([doc])
```

在此示例中，我们使用 GPT-4o 进行图形提取。作者特别指示 LLM [提取实体和关系及其描述](https://github.com/microsoft/graphrag/blob/main/graphrag/prompt_tune/template/entity_extraction.py)。使用 LangChain 实现，您可以使用 node_properties 和 relationship_properties 属性来指定希望 LLM 提取哪些节点或关系属性。

LLMGraphTransformer 实现的不同之处在于，所有节点或关系属性都是可选的，因此并非所有节点都具有该 description 属性。如果我们愿意，我们可以定义自定义提取以具有强制 description 属性，但在本实现中我们将跳过该操作。

我们将并行化请求以加快图形提取速度并将结果存储到 Neo4j：

```python
MAX_WORKERS = 10
NUM_ARTICLES = 2000
graph_documents = []

with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
    # Submitting all tasks and creating a list of future objects
    futures = [
        executor.submit(process_text, f"{row['title']} {row['text']}")
        for i, row in news.head(NUM_ARTICLES).iterrows()
    ]

    for future in tqdm(
        as_completed(futures), total=len(futures), desc="Processing documents"
    ):
        graph_document = future.result()
        graph_documents.extend(graph_document)

graph.add_graph_documents(
    graph_documents,
    baseEntityLabel=True,
    include_source=True
)
```

在此示例中，我们从 2,000 篇文章中提取了图谱信息，并将结果存储到 Neo4j。我们提取了大约 13,000 个实体和 16,000 个关系。以下是图谱中提取的文档的示例。

![文档（蓝色）指向提取的实体和关系](https://cdn.jsdelivr.net/gh/filess/img2@main/2024/07/11/1720687719216-c9e84683-166f-4a61-8369-0dce1fb6b62a.png)

使用 GPT-4o 完成提取大约需要 35（+/- 5）分钟，费用约为 30 美元。

在此步骤中，作者引入了启发式方法来决定是否在多个过程中提取图形信息。为简单起见，我们只进行一次。但是，如果我们想进行多次，我们可以将第一次提取结果作为对话历史记录，并简单地指示 LLM 缺少许多实体，它应该提取更多实体，就像 GraphRAG 作者所做的那样。

之前，我提到了文本块大小的重要性以及它如何影响提取的实体数量。由于我们没有执行任何额外的文本分块，因此我们可以根据文本块大小评估提取实体的分布：

```python
entity_dist = graph.query(
    """
MATCH (d:Document)
RETURN d.text AS text,
       count {(d)-[:MENTIONS]->()} AS entity_count
"""
)
entity_dist_df = pd.DataFrame.from_records(entity_dist)
entity_dist_df["token_count"] = [
    num_tokens_from_string(str(el)) for el in entity_dist_df["text"]
]
# Scatter plot with regression line
sns.lmplot(
    x="token_count",
    y="entity_count",
    data=entity_dist_df,
    line_kws={"color": "red"}
)
plt.title("Entity Count vs Token Count Distribution")
plt.xlabel("Token Count")
plt.ylabel("Entity Count")
plt.show()
```

![](https://cdn.jsdelivr.net/gh/filess/img4@main/2024/07/11/1720690944675-2eaf221b-a285-4a77-9285-b681c2388c21.png)

散点图显示，虽然存在正趋势（红线表示），但关系是次线性的。即使标记数增加，大多数数据点仍聚集在较低的实体数上。这表明提取的实体数量并不与文本块的大小成比例。虽然存在一些异常值，但一般模式表明，较高的标记数并不一定导致较高的实体数。这验证了作者的发现，即较低的文本块大小将提取更多信息。

我还认为检查构造图的节点度分布会很有趣。以下代码检索并可视化节点度分布：

```python
degree_dist = graph.query(
    """
MATCH (e:__Entity__)
RETURN count {(e)-[:!MENTIONS]-()} AS node_degree
"""
)
degree_dist_df = pd.DataFrame.from_records(degree_dist)

# Calculate mean and median
mean_degree = np.mean(degree_dist_df['node_degree'])
percentiles = np.percentile(degree_dist_df['node_degree'], [25, 50, 75, 90])
# Create a histogram with a logarithmic scale
plt.figure(figsize=(12, 6))
sns.histplot(degree_dist_df['node_degree'], bins=50, kde=False, color='blue')
# Use a logarithmic scale for the x-axis
plt.yscale('log')
# Adding labels and title
plt.xlabel('Node Degree')
plt.ylabel('Count (log scale)')
plt.title('Node Degree Distribution')
# Add mean, median, and percentile lines
plt.axvline(mean_degree, color='red', linestyle='dashed', linewidth=1, label=f'Mean: {mean_degree:.2f}')
plt.axvline(percentiles[0], color='purple', linestyle='dashed', linewidth=1, label=f'25th Percentile: {percentiles[0]:.2f}')
plt.axvline(percentiles[1], color='orange', linestyle='dashed', linewidth=1, label=f'50th Percentile: {percentiles[1]:.2f}')
plt.axvline(percentiles[2], color='yellow', linestyle='dashed', linewidth=1, label=f'75th Percentile: {percentiles[2]:.2f}')
plt.axvline(percentiles[3], color='brown', linestyle='dashed', linewidth=1, label=f'90th Percentile: {percentiles[3]:.2f}')
# Add legend
plt.legend()
# Show the plot
plt.show()
```

![](https://cdn.jsdelivr.net/gh/filess/img13@main/2024/07/11/1720691038533-c53b5aff-a307-472f-821c-996a490d0efb.png)

节点度分布遵循幂律模式，表明大多数节点的连接很少，而少数节点的连接高度紧密。平均度为 2.45，中位数为 1.00，表明超过一半的节点只有一个连接。大多数节点（75%）有两个或更少的连接，90% 的节点有五个或更少的连接。这种分布是许多现实世界网络的典型特征，其中少数枢纽有许多连接，而大多数节点的连接很少。

由于节点和关系描述都不是强制属性，我们还将检查提取了多少个：

```python
graph.query("""
MATCH (n:`__Entity__`)
RETURN "node" AS type,
       count(*) AS total_count,
       count(n.description) AS non_null_descriptions
UNION ALL
MATCH (n)-[r:!MENTIONS]->()
RETURN "relationship" AS type,
       count(*) AS total_count,
       count(r.description) AS non_null_descriptions
""")
```

结果显示，在 12,994 个节点中，有 5,926 个节点（占 45.6%）具有描述属性。另一方面，在 15,921 个关系中，只有 5,569 个关系（占 35%）具有此类属性。

请注意，由于 LLM 的概率性质，数字可能会因不同的运行和不同的源数据、LLM 和提示而有所不同。

## 实体解析

实体解析（去重）在构建知识图谱时至关重要，因为它可以确保每个实体都以唯一且准确的形式呈现，从而防止出现重复并合并指向同一现实世界实体的记录。此过程对于维护图谱中的数据完整性和一致性至关重要。如果没有实体解析，知识图谱将受到数据碎片化和不一致的影响，从而导致错误和不可靠的见解。

![潜在实体重复](https://cdn.jsdelivr.net/gh/filess/img12@main/2024/07/11/1720691136978-09a73b4a-1b1d-44a3-b15b-44c76ff5b77a.png)

该图演示了单个现实世界实体如何在不同的文档中以略有不同的名称出现，从而在我们的图表中出现。

此外，如果没有实体解析，数据稀疏就会成为一个重大问题。来自各种来源的不完整或部分数据可能会导致信息分散且不连贯，从而难以形成对实体的连贯和全面的理解。准确的实体解析通过整合数据、填补空白并创建每个实体的统一视图来解决此问题。

![使用实体解析技术前后对比图的连通性](https://cdn.jsdelivr.net/gh/filess/img8@main/2024/07/11/1720691273026-16426082-8127-466f-9203-c5f681621658.png)

可视化的左侧部分呈现了一个稀疏且不连通的图。然而，如右侧所示，这样的图可以通过高效的实体解析变得连通。

> 总体而言，实体解析提高了数据检索和集成的效率，提供了跨不同来源的信息的统一视图。它最终使基于可靠且完整的知识图谱的问答更加有效。

不幸的是，GraphRAG 论文的作者没有 在他们的 repo 中包含任何实体解析代码，尽管他们在论文中提到了这一点。省略此代码的原因之一可能是很难为任何给定域实现稳健且性能良好的实体解析。在处理预定义类型的节点时，您可以为不同的节点实现自定义启发式方法（当它们未预定义时，它们不够一致，如公司、组织、企业等）。但是，如果节点标签或类型事先不知道，就像我们的情况一样，这会成为一个更难的问题。尽管如此，我们将在这里的项目中实现一个版本的实体解析，将文本嵌入和图形算法与词距离和 LLM 相结合。

![实体解析流程](https://cdn.jsdelivr.net/gh/filess/img0@main/2024/07/11/1720691365511-9f980df7-6389-408d-ad3a-dff39a5e5ddb.png)

我们的实体解析流程包括以下步骤：

1. 图中的实体——从图中的所有实体开始。
2. K-最近图——构建 k-最近邻图，根据文本嵌入连接相似的实体。
3. 弱连通分量 — 识别 k-最近图中的弱连通分量，对可能相似的实体进行分组。识别这些分量后，添加一个词距过滤步骤。
4. LLM 评估——使用 LLM 评估这些组件并决定是否应合并每个组件内的实体，从而对实体解析做出最终决定（例如，合并“硅谷银行”和“Silicon_Valley_Bank”，同时拒绝不同日期的合并，如“2023 年 9 月 16 日”和“2023 年 9 月 2 日”）。

我们首先计算实体的名称和描述属性的文本嵌入。我们可以使用 LangChain 中集成 from_existing_graph 的方法[Neo4jVector](https://python.langchain.com/v0.2/docs/integrations/vectorstores/neo4jvector/)来实现这一点：

```python
vector = Neo4jVector.from_existing_graph(
    OpenAIEmbeddings(),
    node_label='__Entity__',
    text_node_properties=['id', 'description'],
    embedding_node_property='embedding'
)
```

我们可以使用这些嵌入来根据这些嵌入的余弦距离找到相似的潜在候选者。我们将使用[图数据科学 (GDS) 库](https://neo4j.com/docs/graph-data-science/current/)中提供的图算法；因此，我们可以使用[GDS Python 客户端](https://neo4j.com/docs/graph-data-science-client/current/)以 Pythonic 方式轻松使用：

```python
from graphdatascience import GraphDataScience

gds = GraphDataScience(
    os.environ["NEO4J_URI"],
    auth=(os.environ["NEO4J_USERNAME"], os.environ["NEO4J_PASSWORD"])
)
```

如果您不熟悉 GDS 库，我们必须首先投影内存图，然后才能执行任何图算法。

![图数据科学算法执行工作流程](https://cdn.jsdelivr.net/gh/filess/img0@main/2024/07/11/1720691617583-d46d8d3e-cb5e-4543-ba31-d61b1c8facbb.png)

首先，将 Neo4j 存储的图投影到内存图中，以便更快地进行处理和分析。接下来，在内存图上执行图算法。或者，可以将算法的结果存储回 Neo4j 数据库。有关更多信息，请参阅图数据科学 (GDS) 库文档。

为了创建 k 最近邻图，我们将投影所有实体及其文本嵌入：

```python
G, result = gds.graph.project(
    "entities",                   # Graph name
    "__Entity__",                 # Node projection
    "*",                          # Relationship projection
    nodeProperties=["embedding"]  # Configuration parameters
)`
```

现在图已投影到名称下 entities，我们可以执行图算法了。我们将从构建[k 最近邻图](https://neo4j.com/docs/graph-data-science/current/algorithms/knn/)开始。影响 k 最近邻图稀疏或密集程度的两个最重要的参数是和 similarityCutoff。topK 是 topK 每个节点要查找的邻居数 ​​，最小值为 1。相似性截止值会过滤掉相似性低于此阈值的关系。在这里，我们将使用默认值 topK10 和相对较高的相似性截止值 0.95。使用高相似性截止值（例如 0.95）可确保只有高度相似的对才被视为匹配，从而最大限度地减少误报并提高准确性。

![构建 k-最近图并将新关系存储在项目图中](https://cdn.jsdelivr.net/gh/filess/img4@main/2024/07/11/1720691805504-c7d9e06f-b5fa-49cc-a79c-84162d5f1057.png)

由于我们希望将结果存储回投影的内存图而不是知识图谱，因此我们将使用 mutate 算法的模式：

```python
similarity_threshold = 0.95

gds.knn.mutate(
  G,
  nodeProperties=['embedding'],
  mutateRelationshipType= 'SIMILAR',
  mutateProperty= 'score',
  similarityCutoff=similarity_threshold
)
```

下一步是识别与新推断的相似性关系相关的实体组。识别连接节点组是网络分析中常见的过程，通常称为社区检测或聚类，其中涉及查找密集连接节点的子组。在此示例中，我们将使用[弱连通分量算法](https://neo4j.com/docs/graph-data-science/current/algorithms/wcc/)，该算法可帮助我们找到图中所有节点都连接的部分，即使我们忽略连接的方向。

![将 WCC 的结果写回数据库](https://cdn.jsdelivr.net/gh/filess/img5@main/2024/07/11/1720691921263-c853ee3e-973e-408a-83f1-3b553ee8f8ef.png)

我们利用算法的 write 模式将结果存回数据库（存储图）：

```python
gds.wcc.write(
    G,
    writeProperty="wcc",
    relationshipTypes=["SIMILAR"]
)
```

文本嵌入比较有助于找到潜在的重复项，但它只是实体解析过程的一部分。例如，谷歌和苹果在嵌入空间中非常接近（使用 ada-002 嵌入模型的余弦相似度为 0.96）。宝马和奔驰也是如此（余弦相似度为 0.97）。高文本嵌入相似度是一个好的开始，但我们可以改进它。因此，我们将添加一个额外的过滤器，仅允许文本距离为三个或更少的单词对（意味着只能更改字符）：

```python
word_edit_distance = 3
potential_duplicate_candidates = graph.query(
    """MATCH (e:`__Entity__`)
    WHERE size(e.id) > 3 // longer than 3 characters
    WITH e.wcc AS community, collect(e) AS nodes, count(*) AS count
    WHERE count > 1
    UNWIND nodes AS node
    // Add text distance
    WITH distinct
      [n IN nodes WHERE apoc.text.distance(toLower(node.id), toLower(n.id)) < $distance
                  OR node.id CONTAINS n.id | n.id] AS intermediate_results
    WHERE size(intermediate_results) > 1
    WITH collect(intermediate_results) AS results
    // combine groups together if they share elements
    UNWIND range(0, size(results)-1, 1) as index
    WITH results, index, results[index] as result
    WITH apoc.coll.sort(reduce(acc = result, index2 IN range(0, size(results)-1, 1) |
            CASE WHEN index <> index2 AND
                size(apoc.coll.intersection(acc, results[index2])) > 0
                THEN apoc.coll.union(acc, results[index2])
                ELSE acc
            END
    )) as combinedResult
    WITH distinct(combinedResult) as combinedResult
    // extra filtering
    WITH collect(combinedResult) as allCombinedResults
    UNWIND range(0, size(allCombinedResults)-1, 1) as combinedResultIndex
    WITH allCombinedResults[combinedResultIndex] as combinedResult, combinedResultIndex, allCombinedResults
    WHERE NOT any(x IN range(0,size(allCombinedResults)-1,1)
        WHERE x <> combinedResultIndex
        AND apoc.coll.containsAll(allCombinedResults[x], combinedResult)
    )
    RETURN combinedResult
    """, params={'distance': word_edit_distance})
```

这个 Cypher 语句稍微复杂一些，其解释超出了本博文的范围。您可以随时请 LLM 对其进行解释。

![Anthropic Claude Sonnet 3.5 — 解释重复实体检测语句](https://cdn.jsdelivr.net/gh/filess/img8@main/2024/07/11/1720692052289-4c242a26-20c3-408d-8cf5-fc2ebde4513b.png)

此外，单词距离截止可以是单词长度的函数而不是单个数字，并且实现可以更具可扩展性。

重要的是，它输出我们可能想要合并的潜在实体组。以下是要合并的潜在节点列表：

```json
 {'combinedResult': ['Sinn Fein', 'Sinn Féin']},
 {'combinedResult': ['Government', 'Governments']},
 {'combinedResult': ['Unreal Engine', 'Unreal_Engine']},
 {'combinedResult': ['March 2016', 'March 2020', 'March 2022', 'March_2023']},
 {'combinedResult': ['Humana Inc', 'Humana Inc.']},
 {'combinedResult': ['New York Jets', 'New York Mets']},
 {'combinedResult': ['Asia Pacific', 'Asia-Pacific', 'Asia_Pacific']},
 {'combinedResult': ['Bengaluru', 'Mangaluru']},
 {'combinedResult': ['U.S. Securities And Exchange Commission',
   'Us Securities And Exchange Commission']},
 {'combinedResult': ['Jp Morgan', 'Jpmorgan']},
 {'combinedResult': ['Brighton', 'Brixton']},
```

如您所见，我们的解析方法对某些节点类型比其他节点类型效果更好。根据快速检查，它似乎对人和组织效果更好，而对日期效果不佳。如果我们使用预定义的节点类型，我们可以为各种节点类型准备不同的启发式方法。在此示例中，我们没有预定义的节点标签，因此我们将求助于 LLM 来做出是否应合并实体的最终决定。

首先，我们需要制定 LLM 提示，以有效地指导和告知有关节点合并的最终决定：

```python
system_prompt = """You are a data processing assistant. Your task is to identify duplicate entities in a list and decide which of them should be merged.
The entities might be slightly different in format or content, but essentially refer to the same thing. Use your analytical skills to determine duplicates.

Here are the rules for identifying duplicates:
1. Entities with minor typographical differences should be considered duplicates.
2. Entities with different formats but the same content should be considered duplicates.
3. Entities that refer to the same real-world object or concept, even if described differently, should be considered duplicates.
4. If it refers to different numbers, dates, or products, do not merge results
"""
user_template = """
Here is the list of entities to process:
{entities}

Please identify duplicates, merge them, and provide the merged list.
"""
```

当需要结构化数据输出时，我总是喜欢使用 with_structured_outputLangChain 中的方法，以避免必须手动解析输出。

在这里，我们将输出定义为 list of lists，其中每个内部列表包含应合并的实体。此结构用于处理输入可能是的情况[Sony, Sony Inc, Google, Google Inc]。在这种情况下，您可能希望将“Sony”和“Sony Inc”与“Google”和“Google Inc”分开合并。

```python
class DuplicateEntities(BaseModel):
    entities: List[str] = Field(
        description="Entities that represent the same object or real-world entity and should be merged"
    )


class Disambiguate(BaseModel):
    merge_entities: Optional[List[DuplicateEntities]] = Field(
        description="Lists of entities that represent the same object or real-world entity and should be merged"
    )


extraction_llm = ChatOpenAI(model_name="gpt-4o").with_structured_output(
    Disambiguate
)
```

接下来，我们将 LLM 提示与结构化输出相结合，使用 LangChain 表达语言 (LCEL) 语法创建链并将其封装在消歧函数中。

```python
extraction_chain = extraction_prompt | extraction_llm


def entity_resolution(entities: List[str]) -> Optional[List[List[str]]]:
    return [
        el.entities
        for el in extraction_chain.invoke({"entities": entities}).merge_entities
    ]
```

我们需要通过该实体消歧函数运行所有潜在的候选节点，以决定是否应该合并它们。为了加快这一进程，我们将再次并行化 LLM 调用：

```python
merged_entities = []
with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
    # Submitting all tasks and creating a list of future objects
    futures = [
        executor.submit(entity_resolution, el['combinedResult'])
        for el in potential_duplicate_candidates
    ]

    for future in tqdm(
        as_completed(futures), total=len(futures), desc="Processing documents"
    ):
        to_merge = future.result()
        if to_merge:
            merged_entities.extend(to_merge)
```

实体解析的最后一步涉及从 LLM 获取结果 entity_resolution 并通过合并指定的节点将其写回数据库：

```python
graph.query("""
UNWIND $data AS candidates
CALL {
  WITH candidates
  MATCH (e:__Entity__) WHERE e.id IN candidates
  RETURN collect(e) AS nodes
}
CALL apoc.refactor.mergeNodes(nodes, {properties: {
    description:'combine',
    `.*`: 'discard'
}})
YIELD node
RETURN count(*)
""", params={"data": merged_entities})
```

这种实体解析并不完美，但它为我们提供了一个可以改进的起点。此外，我们可以改进确定哪些实体应该保留的逻辑。

## 元素总结

下一步，作者执行元素汇总步骤。本质上，每个节点和关系都会经过[实体汇总提示](https://github.com/microsoft/graphrag/blob/main/graphrag/prompt_tune/template/entity_summarization.py)。作者指出了他们的方法的新颖性和趣味性：

> “总体而言，我们在可能存在噪声的图形结构中使用丰富的描述性文本来描述同质节点，这既符合 LLM 的功能，也符合全局、以查询为中心的摘要的需求。这些特性也将我们的图形索引与典型的知识图谱区分开来，后者依靠简洁一致的知识三元组（主语、谓语、宾语）来完成下游推理任务。”

这个想法很令人兴奋。我们仍然从文本中提取主题和对象 ID 或名称，这样即使实体出现在多个文本块中，我们也能将关系链接到正确的实体。然而，关系并没有简化为单一类型。相反，关系类型实际上是一种自由格式的文本，它允许我们保留更丰富、更细致入微的信息。

此外，使用 LLM 总结实体信息，使我们能够更有效地嵌入和索引这些信息和实体，以便进行更准确的检索。

有人可能会说，这种更丰富、更细致的信息也可以通过添加额外的、可能是任意的节点和关系属性来保留。任意节点和关系属性的一个问题是，很难一致地提取信息，因为 LLM 可能使用不同的属性名称或关注每次执行的各种细节。

其中一些问题可以通过使用预定义属性名称以及附加类型和描述信息来解决。在这种情况下，您需要一位主题专家来帮助定义这些属性，而 LLM 几乎没有空间提取预定义描述之外的任何重要信息。

这是一种在知识图谱中呈现更丰富信息的令人兴奋的方法。

元素汇总步骤的一个潜在问题是，它不能很好地扩展，因为它需要对图中的每个实体和关系进行 LLM 调用。我们的图相对较小，只有 13,000 个节点和 16,000 个关系。即使对于这么小的图，我们也需要 29,000 次 LLM 调用，每次调用都会使用几百个标记，这使得它非常昂贵且耗时。因此，我们将在这里避免此步骤。我们仍然可以使用在初始文本处理期间提取的描述属性。

## 构建和总结社区

图谱构建和索引过程的最后一步是识别图中的社区。在这种情况下，社区是一组节点，这些节点之间的连接比与图谱其余部分的连接更紧密，表示更高程度的交互或相似性。以下可视化显示了社区检测结果的示例。

![各个国家的颜色取决于其所属的社区](https://cdn.jsdelivr.net/gh/filess/img14@main/2024/07/11/1720692970078-0f43865a-87b0-460e-8519-06c205a409e3.png)

一旦使用聚类算法识别出这些实体社区，LLM 就会为每个社区生成摘要，提供有关其各自特征和关系的见解。

再次，我们使用 Graph Data Science 库。我们首先投影内存中的图形。为了准确遵循原始文章，我们将实体图投影为无向加权网络，其中网络表示两个实体之间的连接数：

```python
G, result = gds.graph.project(
    "communities",  #  Graph name
    "__Entity__",  #  Node projection
    {
        "_ALL_": {
            "type": "*",
            "orientation": "UNDIRECTED",
            "properties": {"weight": {"property": "*", "aggregation": "COUNT"}},
        }
    },
)
```

作者采用了[莱顿算法(Leiden)](https://neo4j.com/docs/graph-data-science/current/algorithms/leiden/)（一种分层聚类方法）来识别图中的社区。使用分层社区检测算法的一个优点是能够在多个粒度级别检查社区。作者建议总结每个级别的所有社区，以全面了解图的结构。

首先，我们将使用弱连通分量 (WCC) 算法来评估图的连通性。该算法识别图中的孤立部分，这意味着它检测彼此连接但不与图的其余部分连接的节点或组件的子集。这些组件帮助我们了解网络内的碎片化，并识别独立于其他节点的节点组。WCC 对于分析图的整体结构和连通性至关重要。

```python
wcc = gds.wcc.stats(G)
print(f"Component count: {wcc['componentCount']}")
print(f"Component distribution: {wcc['componentDistribution']}")
# Component count: 1119
# Component distribution: {
#   "min":1,
#   "p5":1,
#   "max":9109,
#   "p999":43,
#   "p99":19,
#   "p1":1,
#   "p10":1,
#   "p90":7,
#   "p50":2,
#   "p25":1,
#   "p75":4,
#   "p95":10,
#   "mean":11.3 }
```

WCC 算法结果识别出 1,119 个不同的组件。值得注意的是，最大的组件包含 9,109 个节点，这在现实世界的网络中很常见，因为单个超级组件与许多较小的孤立组件共存。最小的组件有一个节点，平均组件大小约为 11.3 个节点。

接下来，我们将运行 Leiden 算法（该算法在 GDS 库中也可用），并启用参数 includeIntermediateCommunities 以返回和存储各级社区。我们还包含了一个 relationshipWeightProperty 参数来运行 Leiden 算法的加权变体。使用 write 算法的模式将结果存储为节点属性。

```python
gds.leiden.write(
    G,
    writeProperty="communities",
    includeIntermediateCommunities=True,
    relationshipWeightProperty="weight",
)
```

该算法确定了五个级别的社区，最高级别（最小粒度级别，社区最多）有 1,188 个社区（而组件有 1,119 个）。以下是使用 Gephi 对最后一级社区的可视化。

![Gephi 中的社区结构可视化](https://cdn.jsdelivr.net/gh/filess/img13@main/2024/07/11/1720693391356-02981ae7-4fc6-46b3-a21e-3a4eee9de219.png)

可视化 1,000 多个社区非常困难；甚至为每个社区挑选颜色也几乎是不可能的。不过，它们可以带来漂亮的艺术效果。

在此基础上，我们将为每个社区创建一个不同的节点，并将其层次结构表示为一个相互关联的图表。稍后，我们还将把社区摘要和其他属性存储为节点属性。

```python
graph.query("""
MATCH (e:`__Entity__`)
UNWIND range(0, size(e.communities) - 1 , 1) AS index
CALL {
  WITH e, index
  WITH e, index
  WHERE index = 0
  MERGE (c:`__Community__` {id: toString(index) + '-' + toString(e.communities[index])})
  ON CREATE SET c.level = index
  MERGE (e)-[:IN_COMMUNITY]->(c)
  RETURN count(*) AS count_0
}
CALL {
  WITH e, index
  WITH e, index
  WHERE index > 0
  MERGE (current:`__Community__` {id: toString(index) + '-' + toString(e.communities[index])})
  ON CREATE SET current.level = index
  MERGE (previous:`__Community__` {id: toString(index - 1) + '-' + toString(e.communities[index - 1])})
  ON CREATE SET previous.level = index - 1
  MERGE (previous)-[:IN_COMMUNITY]->(current)
  RETURN count(*) AS count_1
}
RETURN count(*)
""")
```

作者还引入了 community rank，表示社区内实体出现的不同文本块的数量：

```python
graph.query("""
MATCH (c:__Community__)<-[:IN_COMMUNITY*]-(:__Entity__)<-[:MENTIONS]-(d:Document)
WITH c, count(distinct d) AS rank
SET c.community_rank = rank;
""")
```

现在让我们研究一个示例层次结构，其中许多中间社区在更高级别合并。社区不重叠，这意味着每个实体在每个级别都属于一个社区。

![分层社区结构；社区为橙色，实体为紫色](https://cdn.jsdelivr.net/gh/filess/img14@main/2024/07/11/1720693599532-3b429e14-6fa2-4290-b95b-f91790895708.png)

该图表示莱顿社区检测算法产生的层次结构。紫色节点表示单个实体，而橙色节点表示层次化社区。

层次结构显示了这些实体组织成各种社区的情况，较小的社区在较高级别上合并到较大的社区。

现在让我们来看看较小的社区是如何在较高层次上合并的。

![分层社区结构](https://cdn.jsdelivr.net/gh/filess/img9@main/2024/07/11/1720693658602-edf9b14e-90ce-46df-a063-a6d4b21f6f5f.png)

该图表明，联系较少的实体以及因此较小的社区在各个层级上经历的变化很小。例如，这里的社区结构仅在前两个层级发生变化，但在最后三个层级保持不变。因此，对于这些实体而言，层级通常显得多余，因为整体组织在不同层级上没有发生显著变化。

让我们更详细地研究社区的数量及其规模和不同级别：

```python
community_size = graph.query(
    """
MATCH (c:__Community__)<-[:IN_COMMUNITY*]-(e:__Entity__)
WITH c, count(distinct e) AS entities
RETURN split(c.id, '-')[0] AS level, entities
"""
)
community_size_df = pd.DataFrame.from_records(community_size)
percentiles_data = []
for level in community_size_df["level"].unique():
    subset = community_size_df[community_size_df["level"] == level]["entities"]
    num_communities = len(subset)
    percentiles = np.percentile(subset, [25, 50, 75, 90, 99])
    percentiles_data.append(
        [
            level,
            num_communities,
            percentiles[0],
            percentiles[1],
            percentiles[2],
            percentiles[3],
            percentiles[4],
            max(subset)
        ]
    )

# Create a DataFrame with the percentiles
percentiles_df = pd.DataFrame(
    percentiles_data,
    columns=[
        "Level",
        "Number of communities",
        "25th Percentile",
        "50th Percentile",
        "75th Percentile",
        "90th Percentile",
        "99th Percentile",
        "Max"
    ],
)
percentiles_df
```

![社区规模按级别分布](https://cdn.jsdelivr.net/gh/filess/img0@main/2024/07/11/1720693729081-50096ed9-22d9-4e20-ba3a-7b2e66f25ec1.png)

在最初的实施中，每个级别的社区都进行了汇总。在我们的案例中，这将是 8,590 个社区，因此有 8,590 个 LLM 调用。我认为，根据分层社区结构，并非每个级别都需要进行汇总。例如，最后一级和倒数第二级之间的差异只有 4 个社区（1,192 对 1,188）。因此，我们将创建大量冗余摘要。一种解决方案是创建一个可以对不同级别上不变的社区进行单一汇总的实施；另一种解决方案是折叠不变的社区层次结构。

另外，我不确定我们是否要总结只有一名成员的社区，因为它们可能不会提供太多价值或信息。在这里，我们将总结 0、1 和 4 级的社区。首先，我们需要从数据库中检索它们的信息：

```python
community_info = graph.query("""
MATCH (c:`__Community__`)<-[:IN_COMMUNITY*]-(e:__Entity__)
WHERE c.level IN [0,1,4]
WITH c, collect(e ) AS nodes
WHERE size(nodes) > 1
CALL apoc.path.subgraphAll(nodes[0], {
 whitelistNodes:nodes
})
YIELD relationships
RETURN c.id AS communityId,
       [n in nodes | {id: n.id, description: n.description, type: [el in labels(n) WHERE el <> '__Entity__'][0]}] AS nodes,
       [r in relationships | {start: startNode(r).id, type: type(r), end: endNode(r).id, description: r.description}] AS rels
""")
```

目前社区信息结构如下：

```python
{'communityId': '0-6014',
 'nodes': [{'id': 'Darrell Hughes', 'description': None, type:"Person"},
  {'id': 'Chief Pilot', 'description': None, type: "Person"},
   ...
  }],
 'rels': [{'start': 'Ryanair Dac',
   'description': 'Informed of the change in chief pilot',
   'type': 'INFORMED',
   'end': 'Irish Aviation Authority'},
  {'start': 'Ryanair Dac',
   'description': 'Dismissed after internal investigation found unacceptable behaviour',
   'type': 'DISMISSED',
   'end': 'Aidan Murray'},
   ...
]}
```

现在，我们需要准备一个 LLM 提示，根据社区元素提供的信息生成自然语言摘要。我们可以从[研究人员使用的提示](https://github.com/microsoft/graphrag/blob/main/graphrag/prompt_tune/template/community_report_summarization.py)中获得一些启发。

作者不仅总结了社区，还为每个社区提出了发现。发现可以定义为有关特定事件或信息的简明信息。例如：

```text
"summary": "Abila City Park as the central location",
"explanation": "Abila City Park is the central entity in this community, serving as the location for the POK rally. This park is the common link between all other
entities, suggesting its significance in the community. The park's association with the rally could potentially lead to issues such as public disorder or conflict, depending on the
nature of the rally and the reactions it provokes. [records: Entities (5), Relationships (37, 38, 39, 40)]"
```

我的直觉表明，仅通过一次通过提取发现可能不如我们需要的那么全面，就像提取实体和关系一样。

此外，我还没有在本地或全局搜索检索器中找到任何关于它们代码使用的参考或示例。因此，在这种情况下，我们将避免提取发现。或者，正如学者们经常说的那样：这个练习留给读者。此外，我们还跳过了声明或[协变量信息提取](https://github.com/microsoft/graphrag/blob/main/graphrag/index/graph/extractors/claims/prompts.py)，乍一看，它们与发现类似。

我们用来制作社区摘要的提示非常简单：

```python
community_template = """Based on the provided nodes and relationships that belong to the same graph community,
generate a natural language summary of the provided information:
{community_info}

Summary:"""  # noqa: E501

community_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "Given an input triples, generate the information summary. No pre-amble.",
        ),
        ("human", community_template),
    ]
)

community_chain = community_prompt | llm | StrOutputParser()
```

剩下唯一要做的事情就是将社区表示转换为字符串，以避免 JSON 令牌开销来减少令牌的数量，并将链包装为一个函数：

```python
def prepare_string(data):
    nodes_str = "Nodes are:\n"
    for node in data['nodes']:
        node_id = node['id']
        node_type = node['type']
        if 'description' in node and node['description']:
            node_description = f", description: {node['description']}"
        else:
            node_description = ""
        nodes_str += f"id: {node_id}, type: {node_type}{node_description}\n"

    rels_str = "Relationships are:\n"
    for rel in data['rels']:
        start = rel['start']
        end = rel['end']
        rel_type = rel['type']
        if 'description' in rel and rel['description']:
            description = f", description: {rel['description']}"
        else:
            description = ""
        rels_str += f"({start})-[:{rel_type}]->({end}){description}\n"

    return nodes_str + "\n" + rels_str

def process_community(community):
    stringify_info = prepare_string(community)
    summary = community_chain.invoke({'community_info': stringify_info})
    return {"community": community['communityId'], "summary": summary}
```

现在我们可以为选定的级别生成社区摘要。同样，我们并行化调用以加快执行速度：

```python
summaries = []
with ThreadPoolExecutor() as executor:
    futures = {executor.submit(process_community, community): community for community in community_info}

    for future in tqdm(as_completed(futures), total=len(futures), desc="Processing communities"):
        summaries.append(future.result())
```

我没有提到的一个方面是，作者还解决了输入社区信息时超出上下文大小的潜在问题。随着图的扩大，社区也会显著增长。在我们的案例中，最大的社区由 545 名成员组成。考虑到 GPT-4o 的上下文大小超过 100,000 个标记，我们决定跳过这一步。

作为最后一步，我们将社区摘要存储回数据库：

```python
graph.query("""
UNWIND $data AS row
MERGE (c:__Community__ {id:row.community})
SET c.summary = row.summary
""", params={"data": summaries})
```

最终的图形结构：

![](https://cdn.jsdelivr.net/gh/filess/img17@main/2024/07/11/1720694269995-cf85303d-630b-47cc-9108-a65c0e3853fe.png)

该图现在包含原始文档、提取的实体和关系以及分层社区结构和摘要。

## 总结

“从局部到全局”论文的作者在展示 GraphRAG 的新方法方面做得非常出色。他们展示了如何将来自各种文档的信息组合并汇总到分层知识图谱结构中。

没有明确提到的一件事是，我们还可以在图形中集成结构化数据源；输入不必仅限于非结构化文本。

我特别欣赏他们的提取方法，因为他们可以捕获节点和关系的描述。描述可以让 LLM 保留更多信息，而不是将所有内容简化为节点 ID 和关系类型。

此外，他们还表明，对文本进行一次提取可能无法捕获所有相关信息，并引入了在必要时执行多次提取的逻辑。作者还提出了一个有趣的想法，即对图社区进行摘要，使我们能够在多个数据源中嵌入和索引精简的主题信息。

在下一篇博文中，我们将讨论本地和全局搜索检索器的实现，并讨论我们可以基于给定的图形结构实现的其他方法。

与往常一样，案例代码可在GitHub上获取。

这次，我还上传了[Neo4j数据库转储文件](https://drive.google.com/file/d/13_2rxUZAvuf7h9hxrMYw_HQhYVSqA6os/view?pli=1)，以便您可以探索结果并尝试不同的检索器选项。

您还可以将此转储导入永久免费的 [Neo4j AuraDB 实例](https://console.neo4j.io/)，我们可以将其用于检索探索，因为我们不需要图形数据科学算法 - 只需要图形模式匹配、向量和全文索引。

在我的书“[数据科学的图算法](https://www.manning.com/books/graph-algorithms-for-data-science)”中[了解有关Neo4j 与所有 GenAI 框架和实用图形算法的集成的更多信息](https://neo4j.com/labs/genai-ecosystem/)。
