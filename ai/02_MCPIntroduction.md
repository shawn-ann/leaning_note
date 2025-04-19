# What is Model Context Protocol (MCP)


MCP is an open standard introduced by Anthropic.
MCP standardize how AI applications (chatbots, IDE assistants, or custom agents) connect with external tools, data sources, and systems. Just like USB-C offers a standardized way to connect devices to various accessories.

<img src="../img/mcp_typec_for_llm.png"  width="600" height="auto">


# General Architecture

MCP follows a client-server architecture and consist of below key component:

<img src="../img/mcp_architecture.png"  width="600" height="auto">

### Host

Host represents any AI app (Claude desktop, Cursor) that provides an environment for AI interactions, accesses tools and data, and runs the MCP Client.

### Client

They maintain dedicated, one-to-one connections with MCP servers.

### Server
  
Lightweight servers exposing specific capabilities and provides access to data like:

  - Tools: Enable LLMs to perform actions through your server.

  - Resources: Expose data and content from your servers to LLMs.

  - Prompts: Create reusable prompt templates and workflows.



## Client-Server Communication

<img src="../img/mcp_server_client_communication.webp"  width="600" height="auto">


 - The client sends an initial request to learn server capabilities.

 - The server responds with details about its available tools, resources, prompts, and parameters
   - For instance, a Weather API server, when invoked, can reply back with available “tools” get_weather_by_location

# Sequencial Diagram

<img src="../img/mcp_sequencial_diagram.svg"  width="1200" height="auto">

# Ecosystem

## Hosts
- Claude Desktop
- Cusor
- Cline

## Servers
There are a lot of servers already and we can use them through the following approach.

- [Github](https://github.com/modelcontextprotocol/servers)
  - Fetch - A server that flexibly fetches HTML, JSON, Markdown, or plaintext.
  - Google Maps - Location services, directions, and place details
  - Slack - Channel management and messaging capabilities
  - PostgreSQL - Read-only database access with schema inspection
  - Playwright - This MCP Server will help you run browser automation and webscraping using Playwright
  - ...

- Marketplace
  
  - [MCP.so](https://mcp.so/)
  - [MCP Marketplace - Cline](https://cline.bot/mcp-marketplace)
  - [MCP Market | Discover Top MCP Servers](https://mcpmarket.com/)
- Develope by yourself

# Demo
## Search iPhone Prices

In this case, I will use the Browser Automation MCP server for  and then search for the iPhone's price on the official Apple website.

 <video controls>
  <source src="../video/mcp_demo1_webautomation.mov" type="video/mp4">
</video>

# Reference

https://blog.dailydoseofds.com/p/visual-guide-to-model-context-protocol
https://norahsakal.com/blog/mcp-vs-api-model-context-protocol-explained/

```mermaid
sequenceDiagram
    participant User
    participant MCP Host
    participant LLM
    participant MCP Client
    participant MCP Server
    participant Datasource Service

    User->>MCP Host: 1. Raise Question
    MCP Host->>LLM: 2. Forward Question to LLM
    LLM->>LLM: 3. Analyze Need for External Tools
    LLM->>MCP Client: 4. Request to Use Tool
    
    MCP Client->>MCP Server: 5. Invoke Tool via Server
    MCP Server->>Datasource Service: 6. Access Datasource/Service
    Datasource Service-->>MCP Server: 7. Return Data
    MCP Server-->>MCP Client: 8. Return Processed Result
    MCP Client-->>LLM: 9. Pass Tool Execution Output
    LLM-->>MCP Host: 10. Generate Answer (Based on Tool Execution Output)
    MCP Host-->>User: 11. Return Final Response
```