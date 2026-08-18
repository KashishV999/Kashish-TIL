+++
title = 'TIL #1 : Model Context Protocol (MCP) and Agents'
date = 2026-08-03T07:07:07+01:00
draft = false
show_reading_time = true
omit_header_text = true
+++

This is my first post in my "Today I Learned" series. Words like MCP host, MCP server, protocol, agent, subagent, tool, and RAG get thrown around so easily as buzzwords. I tried to put together a simple explanation of them so that anyone can understand it.

<!--more-->

## Before MCP

If someone needed to connect their LLM application to an external system to interact with it, they had to "write" code for every tool they needed.
Eg: If I need to read messages from Slack, I need to write a `readMessage` tool using the GET API from Slack, and so on for every other tool.

- This means every time I build my application, I need to include this code myself, and if the provider (Slack) changes their API, I need to change my code too.
- Also, everyone writes their own code for this, so it's pretty inconsistent, since everyone has their own way of writing the tool. Eg: the schema of the output returned by A's code is different from B's code, even though both use the same API for the same tool.

## After MCP

MCP just standardizes this whole process. Now the provider (eg: Slack) creates their own MCP server and puts all the low-level implementation inside it, exposing a standard interface/schema to users.

Now everyone can easily connect to it, and if the provider changes their API later, you don't have to change your own code, since whoever wrote the server will change the low-level implementation while the interface stays the same.

Eg: Every MCP server, no matter who wrote it or what system it wraps, speaks the same wire format (language):

```
tools/list : same request/response JSON-RPC shape
tools/call : same request/response JSON-RPC shape
inputSchema : declared the same way (JSON Schema format) for every tool, on every server
```

So one day you were using the Air Canada MCP server in your code, but later you decide to move to the Emirates MCP server. The switch will be easy since the interface exposed to you uses the same language, so you can still call `tools/list` or `tools/call` on any server.

## Model Context Protocol

MCP is used to connect an LLM application to external systems and "interact" with them.
Eg: When connected to an airline MCP server, the LLM application can not only retrieve information but also perform tasks like booking a flight, etc.

**MCP host**: The part where the user interacts with the LLM application, where the user types things in. Eg: your Claude application, etc.

**MCP client**: Lives inside the MCP host. Each MCP client connects to one MCP server, sees what tool(s) that server offers, and gives the schema to the LLM. Whatever tool the LLM wants to call, the client takes that request and calls the tool on the server.

**MCP server**: Holds all the available tools. Internally it has the code that implements them, but it exposes the tools in a standard format, then returns the output to the client over the protocol.

**MCP protocol**: How the client and server talk to each other. It's an "agreed upon" language/grammar, it should have `id`, `jsonrpc`, etc. fields, so both sides know what's being asked.

Ex: JSON-RPC call from a client to call a tool on an MCP server:

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "name_of_the_tool_to_run",
    "arguments": {
      "parameter_1": "value_1",
      "parameter_2": "value_2"
    }
  },
  "id": "unique-request-id-123"
}
```

Flow of client and server:

1. Host starts up, connects to MCP server(s), one client per server.
2. Client calls `tools/list`, gets all the available tool schemas from the server, and passes them to the LLM as context.
3. User types a request. The LLM decides whether it needs to call a tool, and if yes, it tells the client which tool to call with which args.
4. Client asks the server to call the tool by passing `tools/call`.
5. Server receives the message from the client over the protocol, runs the function, and returns the output schema back to the client.
6. Client passes the result back up.

## Tools, Agents, Subagents, MCP

**Tool**: The most basic unit. A tool doesn't "decide" anything, it's given certain input, and it's deterministic code that runs when called and returns a result.
Eg: a `read_file` tool accepts a filename and returns the content of the file. Similarly, `web_search`, etc.

**Agents**: The decision-making "loop" that perceives a situation, decides what to do, and takes actions toward a certain goal. It has access to tools and makes autonomous decisions, without a human manually driving each step.

Agents use the **ReAct** pattern to get there. Instead of just jumping straight to a final answer, it loops through:

1. **Reason**: The model thinks about what it knows so far and what it should do next.
2. **Action**: Based on that reasoning, it calls a tool or takes a step.
3. **Observe**: The result of that action gets fed back into the context.
4. **Repeat** steps 1 to 3 until it has enough information to give a final answer.

Example query: *What is the population density of the most populated city in the country where the most recent Olympics were held?*

```
AI: call web_search_tool("In which country was the most recent Olympics held?")
received: France
AI: call web_search_tool("Most populated city in France")
received: Paris
AI: call web_search_tool("Population of Paris")
received: 2,102,650
AI: call calculate_density_tool(population: 2102650, area_km2: 105)
received: ~20,000/km²

Final Answer: The population density of Paris, France is ~20,000/km².
```

*Note: how at each step the AI perceives the info and decides what it needs to do next, in a loop, until it achieves the goal.*

**Subagent**: Not architecturally different from an agent at all. It's just an agent that's been wrapped and exposed as a tool to a parent/main agent, following the "agent as tool" pattern. It has its own tools it needs, just like any other agent.

**MCP**: A container/exposer of tools. The server itself isn't intelligent, it's just infrastructure standardizing how a set of tools gets discovered and called. One server can hold many tools.

**RAG (Retrieval Augmented Generation)**: Used to connect an LLM to new information it didn't see during pretraining, and provide that as context so the output is grounded in that information. Eg: connecting to your database, etc. MCP, on the other hand, "interacts."

## Blogs I found useful

- ["I think 'agent' may finally have a widely enough agreed upon definition to be useful jargon now"](https://simonwillison.net/2025/Sep/18/agents/)
- [Google Cloud: What is Model Context Protocol](https://cloud.google.com/discover/what-is-model-context-protocol)
- [JSON-RPC message structure](https://apxml.com/courses/getting-started-model-context-protocol/chapter-1-architecture-and-fundamentals/json-rpc-message-structure)
- [Context Engineering, Part 2: keep the model's toolset small](https://www.philschmid.de/context-engineering-part-2#3-keep-the-models-toolset-small)