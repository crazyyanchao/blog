---
title: 释放大模型潜力：Model Context Protocol 引领 API 开发新纪元
tags: [MCP,AGENT,LLM]
author: Yc-Ma
show_author_profile: true
key: 2024-11-26-释放大模型潜力：Model Context Protocol 引领 API 开发新纪元
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

# 释放大模型潜力：Model Context Protocol 引领 API 开发新纪元

## 概述

&emsp; Model Context Protocol（简称MCP）是由人工智能公司Anthropic提出的一种API开发标准，旨在实现AI助手与数据源的无缝连接。MCP的核心价值在于统一数据访问、多功能支持和提升大型语言模型（LLM）的实用性。它允许AI模型从各种数据源（如本地内容存储库、业务工具、开发环境）中提取信息以完成任务，从而生成更相关、更准确的响应。

&emsp; **MCP协议的核心是将一般的`AI AGENT`函数调用的工程架构抽象为了客户端、服务器结构，可以理解为是对`函数调用AGENT`在工程层面实现了一个更好的抽象。抽象之后客户端可以是传统的服务请求、大模型请求、或者是一般的Tool请求等。服务器端的实现则是更适合大模型体质的方式，同时兼容非大模型的请求，服务器端的实现最核心的地方在于引入了Resources、Tools、Prompts等概念。**

&emsp;**Resources主要作用是描述当前服务端，一般包含服务的唯一标识、服务的名称、服务的主要功能、以及其它相关信息（当通过大模型的客户端提问时，需要知道当前问题应该需要由哪个服务解决此时会依赖这个资源描述信息）**。

&emsp;**Tools主要用于定义一组可用API，表示该服务可提供的功能，同时每个Tool都有类似`Resources`的描述，只不过Tool不需要唯一标识。**

&emsp;**Prompts主要作用是为大模型实现的客户端提供一些有价值的上下文信息（这是可选的），可以是模板方式生成、也可以是动态的，取决于当前服务具体的功能是什么。**

&emsp;MCP的核心特点包括：

**1. 数据访问标准化**：提供一个通用的接口，简化与不同数据源的连接。

**2. 双向安全连接**：确保AI应用和数据源之间的通信是安全的。

**3. 上下文感知能力**：AI模型能够根据上下文提取和使用信息。

**4. 模块化与可扩展性**：支持开发者通过“连接器”扩展功能。

**5. 开源与社区支持**：完全开源，鼓励社区贡献和迭代。

**6. 多场景应用支持**：适用于多种业务和开发环境。

&emsp;MCP的架构基于客户端-服务器模型，包括MCP主机、客户端、服务器以及本地和远程资源。它的工作原理涉及自动发现服务器、协议握手和执行操作，如运行SQL查询。Anthropic提供了MCP的开源工具和SDK，以促进快速集成和开发者生态的形成。

## MCP工作原理

```mermaid
%%{init: {'flowchart': {'curve': 'linear', 'nodeSpacing': 50, 'rankSpacing': 50}}}%%
graph LR;
    A[MCP Host <br> Claude, IDEs, Tools...]:::host
    B[MCP Server A]:::server
    C[MCP Server B]:::server
    D[MCP Server C]:::server
    E[Local Resource A]:::resource
    F[Local Resource B]:::resource

    subgraph internet
        G[Remote Resource C]:::resource
    end

    A -->|MCP Protocol| B;
    A -->|MCP Protocol| C;
    A -->|MCP Protocol| D;
    B --> E;
    C --> F;
    D -->|Web APIs| G;

    classDef default fill:#f5f5f5,stroke:#bbbbbb,stroke-width:2px,font-size:14px;
    classDef host fill:#ffecd2,stroke:#ff6347,stroke-width:2px,border-radius:10px;
    classDef server fill:#d3eafd,stroke:#547fb3,stroke-width:2px,border-radius:10px;
    classDef resource fill:#fff9cc,stroke:#c2a900,stroke-width:2px,border-radius:10px;
    classDef internet fill:#ffe0e0,stroke:#ff6b6b,stroke-width:2px,border-radius:10px;
```

- MCP Host：希望通过 MCP 访问资源的程序，例如 Claude Desktop、IDE 或 AI 工具等
- MCP Clients：与服务器保持 1:1 连接的协议客户端
- MCP Servers：轻量级程序，每个程序都通过标准化MCP协议公开特定功能
- Local Resource：MCP 服务器可以安全访问的您的计算机资源（数据库、文件、服务）
- Remote Resource：MCP 服务器可以连接到的互联网资源（例如通过 API）

## 案例解析

### 案例概述
&emsp; 在这个示例中Claude Desktop作为MCP客户端，SQLite MCP 服务器提供安全的数据库访问，资源是本地 SQLite 数据库存储实际数据（数据是否会暴露到互联网取决于MCP Server的具体实现逻辑）。

```mermaid
%%{init: {'flowchart': {'curve': 'linear', 'nodeSpacing': 50, 'rankSpacing': 50}}}%%
graph LR;
    A[Claude Desktop]:::host
    B[SQLite MCP Server]:::server
    C[SQLite Database ../test.db]:::resource

    A <-- MCP Protocol<br/> Queries & Results --> B
    B <-- Local Access<br/> SQL Operations --> C

    classDef default fill:#f5f5f5,stroke:#bbbbbb,stroke-width:2px,font-size:14px;
    classDef host fill:#ffecd2,stroke:#ff6347,stroke-width:2px,border-radius:10px;
    classDef server fill:#d3eafd,stroke:#547fb3,stroke-width:2px,border-radius:10px;
    classDef resource fill:#fff9cc,stroke:#c2a900,stroke-width:2px,border-radius:10px;
```

&emsp; 当实现一个服务器端时，相当于打包了一组Tools，作为一个服务端提供服务，例如`mcp-server-sqlite(查询数据,删除数据,创建数据表...)`。

### 截图演示
[截图演示](https://modelcontextprotocol.io/quickstart)
![](vx_images/188985621207407.png =512x)

### 执行过程
&emsp; 当向Claude Desktop询问数据时，它会首先确定应该使用哪个MCP服务器，例如在本例中为sqlite服务器。随后，Claude Desktop利用其MCP协议能力，向该MCP服务器发起请求，以获取所需数据或执行相关操作。完整交互流程如下：

```mermaid
sequenceDiagram
    participant CD as Claude Desktop
    participant MS as MCP Server
    participant DB as SQLite DB

    CD ->> MS: Initialize connection
    MS -->> CD: Available capabilities
    CD ->> MS: Query request
    MS ->> DB: SQL query
    DB -->> MS: Results
    MS -->> CD: Formatted results
```

### MPC Server实现
```python
import sqlite3
import logging
from logging.handlers import RotatingFileHandler
from contextlib import closing
from pathlib import Path
from mcp.server.models import InitializationOptions
import mcp.types as types
from mcp.server import NotificationOptions, Server
import mcp.server.stdio
from pydantic import AnyUrl
from typing import Any

logger = logging.getLogger('mcp_sqlite_server')
logger.info("Starting MCP SQLite Server")

PROMPT_TEMPLATE = """
The assistants goal is to walkthrough an informative demo of MCP. To demonstrate the Model Context Protocol (MCP) we will leverage this example server to interact with an SQLite database.
It is important that you first explain to the user what is going on. The user has downloaded and installed the SQLite MCP Server and is now ready to use it.
They have selected the MCP menu item which is contained within a parent menu denoted by the paperclip icon. Inside this menu they selected an icon that illustrates two electrical plugs connecting. This is the MCP menu.
Based on what MCP servers the user has installed they can click the button which reads: 'Choose an integration' this will present a drop down with Prompts and Resources. The user has selected the prompt titled: 'mcp-demo'.
This text file is that prompt. The goal of the following instructions is to walk the user through the process of using the 3 core aspects of an MCP server. These are: Prompts, Tools, and Resources.
They have already used a prompt and provided a topic. The topic is: {topic}. The user is now ready to begin the demo.
Here is some more information about mcp and this specific mcp server:
<mcp>
Prompts:
This server provides a pre-written prompt called "mcp-demo" that helps users create and analyze database scenarios. The prompt accepts a "topic" argument and guides users through creating tables, analyzing data, and generating insights. For example, if a user provides "retail sales" as the topic, the prompt will help create relevant database tables and guide the analysis process. Prompts basically serve as interactive templates that help structure the conversation with the LLM in a useful way.
Resources:
This server exposes one key resource: "memo://insights", which is a business insights memo that gets automatically updated throughout the analysis process. As users analyze the database and discover insights, the memo resource gets updated in real-time to reflect new findings. The memo can even be enhanced with Claude's help if an Anthropic API key is provided, turning raw insights into a well-structured business document. Resources act as living documents that provide context to the conversation.
Tools:
This server provides several SQL-related tools:
"read-query": Executes SELECT queries to read data from the database
"write-query": Executes INSERT, UPDATE, or DELETE queries to modify data
"create-table": Creates new tables in the database
"list-tables": Shows all existing tables
"describe-table": Shows the schema for a specific table
"append-insight": Adds a new business insight to the memo resource
</mcp>
<demo-instructions>
You are an AI assistant tasked with generating a comprehensive business scenario based on a given topic.
Your goal is to create a narrative that involves a data-driven business problem, develop a database structure to support it, generate relevant queries, create a dashboard, and provide a final solution.

At each step you will pause for user input to guide the scenario creation process. Overall ensure the scenario is engaging, informative, and demonstrates the capabilities of the SQLite MCP Server.
You should guide the scenario to completion. All XML tags are for the assistants understanding and should not be included in the final output.

1. The user has chosen the topic: {topic}.

2. Create a business problem narrative:
a. Describe a high-level business situation or problem based on the given topic.
b. Include a protagonist (the user) who needs to collect and analyze data from a database.
c. Add an external, potentially comedic reason why the data hasn't been prepared yet.
d. Mention an approaching deadline and the need to use Claude (you) as a business tool to help.

3. Setup the data:
a. Instead of asking about the data that is required for the scenario, just go ahead and use the tools to create the data. Inform the user you are "Setting up the data".
b. Design a set of table schemas that represent the data needed for the business problem.
c. Include at least 2-3 tables with appropriate columns and data types.
d. Leverage the tools to create the tables in the SQLite database.
e. Create INSERT statements to populate each table with relevant synthetic data.
f. Ensure the data is diverse and representative of the business problem.
g. Include at least 10-15 rows of data for each table.

4. Pause for user input:
a. Summarize to the user what data we have created.
b. Present the user with a set of multiple choices for the next steps.
c. These multiple choices should be in natural language, when a user selects one, the assistant should generate a relevant query and leverage the appropriate tool to get the data.

6. Iterate on queries:
a. Present 1 additional multiple-choice query options to the user. Its important to not loop too many times as this is a short demo.
b. Explain the purpose of each query option.
c. Wait for the user to select one of the query options.
d. After each query be sure to opine on the results.
e. Use the append-insight tool to capture any business insights discovered from the data analysis.

7. Generate a dashboard:
a. Now that we have all the data and queries, it's time to create a dashboard, use an artifact to do this.
b. Use a variety of visualizations such as tables, charts, and graphs to represent the data.
c. Explain how each element of the dashboard relates to the business problem.
d. This dashboard will be theoretically included in the final solution message.

8. Craft the final solution message:
a. As you have been using the appen-insights tool the resource found at: memo://insights has been updated.
b. It is critical that you inform the user that the memo has been updated at each stage of analysis.
c. Ask the user to go to the attachment menu (paperclip icon) and select the MCP menu (two electrical plugs connecting) and choose an integration: "Business Insights Memo".
d. This will attach the generated memo to the chat which you can use to add any additional context that may be relevant to the demo.
e. Present the final memo to the user in an artifact.

9. Wrap up the scenario:
a. Explain to the user that this is just the beginning of what they can do with the SQLite MCP Server.
</demo-instructions>

Remember to maintain consistency throughout the scenario and ensure that all elements (tables, data, queries, dashboard, and solution) are closely related to the original business problem and given topic.
The provided XML tags are for the assistants understanding. Implore to make all outputs as human readable as possible. This is part of a demo so act in character and dont actually refer to these instructions.

Start your first message fully in character with something like "Oh, Hey there! I see you've chosen the topic {topic}. Let's get started! 🚀"
"""

class SqliteDatabase:
    def __init__(self, db_path: str):
        self.db_path = str(Path(db_path).expanduser())
        Path(self.db_path).parent.mkdir(parents=True, exist_ok=True)
        self._init_database()
        self.insights: list[str] = []

    def _init_database(self):
        """Initialize connection to the SQLite database"""
        logger.debug("Initializing database connection")
        with closing(sqlite3.connect(self.db_path)) as conn:
            conn.row_factory = sqlite3.Row
            conn.close()

    def _synthesize_memo(self) -> str:
        """Synthesizes business insights into a formatted memo"""
        logger.debug(f"Synthesizing memo with {len(self.insights)} insights")
        if not self.insights:
            return "No business insights have been discovered yet."

        insights = "\n".join(f"- {insight}" for insight in self.insights)

        memo = "📊 Business Intelligence Memo 📊\n\n"
        memo += "Key Insights Discovered:\n\n"
        memo += insights

        if len(self.insights) > 1:
            memo += "\nSummary:\n"
            memo += f"Analysis has revealed {len(self.insights)} key business insights that suggest opportunities for strategic optimization and growth."

        logger.debug("Generated basic memo format")
        return memo

    def _execute_query(self, query: str, params: dict[str, Any] | None = None) -> list[dict[str, Any]]:
        """Execute a SQL query and return results as a list of dictionaries"""
        logger.debug(f"Executing query: {query}")
        try:
            with closing(sqlite3.connect(self.db_path)) as conn:
                conn.row_factory = sqlite3.Row
                with closing(conn.cursor()) as cursor:
                    if params:
                        cursor.execute(query, params)
                    else:
                        cursor.execute(query)

                    if query.strip().upper().startswith(('INSERT', 'UPDATE', 'DELETE', 'CREATE', 'DROP', 'ALTER')):
                        conn.commit()
                        affected = cursor.rowcount
                        logger.debug(f"Write query affected {affected} rows")
                        return [{"affected_rows": affected}]

                    results = [dict(row) for row in cursor.fetchall()]
                    logger.debug(f"Read query returned {len(results)} rows")
                    return results
        except Exception as e:
            logger.error(f"Database error executing query: {e}")
            raise

async def main(db_path: str):
    logger.info(f"Starting SQLite MCP Server with DB path: {db_path}")

    db = SqliteDatabase(db_path)
    server = Server("sqlite-manager")

    # Register handlers
    logger.debug("Registering handlers")

    @server.list_resources()
    async def handle_list_resources() -> list[types.Resource]:
        logger.debug("Handling list_resources request")
        return [
            types.Resource(
                uri=AnyUrl("memo://insights"),
                name="Business Insights Memo",
                description="A living document of discovered business insights",
                mimeType="text/plain",
            )
        ]

    @server.read_resource()
    async def handle_read_resource(uri: AnyUrl) -> str:
        logger.debug(f"Handling read_resource request for URI: {uri}")
        if uri.scheme != "memo":
            logger.error(f"Unsupported URI scheme: {uri.scheme}")
            raise ValueError(f"Unsupported URI scheme: {uri.scheme}")

        path = str(uri).replace("memo://", "")
        if not path or path != "insights":
            logger.error(f"Unknown resource path: {path}")
            raise ValueError(f"Unknown resource path: {path}")

        return db._synthesize_memo()

    @server.list_prompts()
    async def handle_list_prompts() -> list[types.Prompt]:
        logger.debug("Handling list_prompts request")
        return [
            types.Prompt(
                name="mcp-demo",
                description="A prompt to seed the database with initial data and demonstrate what you can do with an SQLite MCP Server + Claude",
                arguments=[
                    types.PromptArgument(
                        name="topic",
                        description="Topic to seed the database with initial data",
                        required=True,
                    )
                ],
            )
        ]

    @server.get_prompt()
    async def handle_get_prompt(name: str, arguments: dict[str, str] | None) -> types.GetPromptResult:
        logger.debug(f"Handling get_prompt request for {name} with args {arguments}")
        if name != "mcp-demo":
            logger.error(f"Unknown prompt: {name}")
            raise ValueError(f"Unknown prompt: {name}")

        if not arguments or "topic" not in arguments:
            logger.error("Missing required argument: topic")
            raise ValueError("Missing required argument: topic")

        topic = arguments["topic"]
        prompt = PROMPT_TEMPLATE.format(topic=topic)

        logger.debug(f"Generated prompt template for topic: {topic}")
        return types.GetPromptResult(
            description=f"Demo template for {topic}",
            messages=[
                types.PromptMessage(
                    role="user",
                    content=types.TextContent(type="text", text=prompt.strip()),
                )
            ],
        )

    @server.list_tools()
    async def handle_list_tools() -> list[types.Tool]:
        """List available tools"""
        return [
            types.Tool(
                name="read-query",
                description="Execute a SELECT query on the SQLite database",
                inputSchema={
                    "type": "object",
                    "properties": {
                        "query": {"type": "string", "description": "SELECT SQL query to execute"},
                    },
                    "required": ["query"],
                },
            ),
            types.Tool(
                name="write-query",
                description="Execute an INSERT, UPDATE, or DELETE query on the SQLite database",
                inputSchema={
                    "type": "object",
                    "properties": {
                        "query": {"type": "string", "description": "SQL query to execute"},
                    },
                    "required": ["query"],
                },
            ),
            types.Tool(
                name="create-table",
                description="Create a new table in the SQLite database",
                inputSchema={
                    "type": "object",
                    "properties": {
                        "query": {"type": "string", "description": "CREATE TABLE SQL statement"},
                    },
                    "required": ["query"],
                },
            ),
            types.Tool(
                name="list-tables",
                description="List all tables in the SQLite database",
                inputSchema={
                    "type": "object",
                    "properties": {},
                },
            ),
            types.Tool(
                name="describe-table",
                description="Get the schema information for a specific table",
                inputSchema={
                    "type": "object",
                    "properties": {
                        "table_name": {"type": "string", "description": "Name of the table to describe"},
                    },
                    "required": ["table_name"],
                },
            ),
            types.Tool(
                name="append-insight",
                description="Add a business insight to the memo",
                inputSchema={
                    "type": "object",
                    "properties": {
                        "insight": {"type": "string", "description": "Business insight discovered from data analysis"},
                    },
                    "required": ["insight"],
                },
            ),
        ]

    @server.call_tool()
    async def handle_call_tool(
        name: str, arguments: dict[str, Any] | None
    ) -> list[types.TextContent | types.ImageContent | types.EmbeddedResource]:
        """Handle tool execution requests"""
        try:
            if name == "list-tables":
                results = db._execute_query(
                    "SELECT name FROM sqlite_master WHERE type='table'"
                )
                return [types.TextContent(type="text", text=str(results))]

            elif name == "describe-table":
                if not arguments or "table_name" not in arguments:
                    raise ValueError("Missing table_name argument")
                results = db._execute_query(
                    f"PRAGMA table_info({arguments['table_name']})"
                )
                return [types.TextContent(type="text", text=str(results))]

            elif name == "append-insight":
                if not arguments or "insight" not in arguments:
                    raise ValueError("Missing insight argument")

                db.insights.append(arguments["insight"])
                _ = db._synthesize_memo()

                # Notify clients that the memo resource has changed
                await server.request_context.session.send_resource_updated(AnyUrl("memo://insights"))

                return [types.TextContent(type="text", text="Insight added to memo")]

            if not arguments:
                raise ValueError("Missing arguments")

            if name == "read-query":
                if not arguments["query"].strip().upper().startswith("SELECT"):
                    raise ValueError("Only SELECT queries are allowed for read-query")
                results = db._execute_query(arguments["query"])
                return [types.TextContent(type="text", text=str(results))]

            elif name == "write-query":
                if arguments["query"].strip().upper().startswith("SELECT"):
                    raise ValueError("SELECT queries are not allowed for write-query")
                results = db._execute_query(arguments["query"])
                return [types.TextContent(type="text", text=str(results))]

            elif name == "create-table":
                if not arguments["query"].strip().upper().startswith("CREATE TABLE"):
                    raise ValueError("Only CREATE TABLE statements are allowed")
                db._execute_query(arguments["query"])
                return [types.TextContent(type="text", text="Table created successfully")]

            else:
                raise ValueError(f"Unknown tool: {name}")

        except sqlite3.Error as e:
            return [types.TextContent(type="text", text=f"Database error: {str(e)}")]
        except Exception as e:
            return [types.TextContent(type="text", text=f"Error: {str(e)}")]

    async with mcp.server.stdio.stdio_server() as (read_stream, write_stream):
        logger.info("Server running with stdio transport")
        await server.run(
            read_stream,
            write_stream,
            InitializationOptions(
                server_name="sqlite",
                server_version="0.1.0",
                capabilities=server.get_capabilities(
                    notification_options=NotificationOptions(),
                    experimental_capabilities={},
                ),
            ),
        )
```

- [mcp_sqlite_server code](https://github.com/modelcontextprotocol/servers/blob/main/src/sqlite/src/mcp_server_sqlite/server.py)
- [Model Context Protocol](https://github.com/modelcontextprotocol)
- [Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol)
- [Model Context Protocol Quickstart](https://modelcontextprotocol.io/quickstart)



