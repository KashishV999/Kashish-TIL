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
if someone need to connect their LLM application to external system to interact with it , they needed to "write" code for every tool they need:
Eg: If i need to read messages from slack , I need to write a tool readMessage tool using the /GET API from slack  and so on for other tools 
- Now this means that everytime I create my application I need to include this code as well , and if the Provider(Slack) changes their API , then I as well need to chnage my code 
- Also everyone writes their own code in their app so pretty inconsitent since everyone has their own way of writing the tool : Eg the shape of output returned by A code is different than B code even though both use same API for same tool


## After MCP

MCP just standardizes this whole process, as now someone (mostly officially by Provider eg: Slack create their MCP server and put all the low level implementaion inside it and exposes the standard interface/schema to users 
Now everyone can easily connect to it and if they chnage their API later , you do not have to change your own code cause the whoever wrote it they will chnage their low level implemenation while the interface remains standard

Eg:
Every MCP server, no matter who wrote it or what system it wraps, speaks the same wire format(language):
tools/list — same request/response JSON-RPC shape
tools/call — same request/response JSON-RPC shape
inputSchema — declared the same way (JSON Schema format) for every tool, on every server

So one day you were using Aircanada mcp server in your code , but later you decide to move to Emirates mcp server , the swicth will be easy since the interfcae exposed to you is using same langauge , so you can still call tool/list or tool/call on any server


## Model Context Protocol

MCP is used to connect LLM application to external systems and "interact" with them
Eg: When connected to Airline mcp server , LLM aplication can not just retrieve information but also perform task like book a flight etc. 

**MCP host**: This is the part where the user interacts with the LLM application and user types in eg: your claude application etc

**MCP client**: It lives inside the MCP host , each mcp client connects to one mcp server , and see what tool/s that server offers and give schema to llm and then whatever llm want to call , get that info and then call that tool by giving to server over protocol

**MCP server**: has all the available tool , and internally it has code that does the implementation but exposes the tools in a standard format and then return the output and give to client over the protocol

**MCP protocol** : How client and server talk to each other. Its an "agreed" upon language/grammar. It should have id , jsonrpc field etc fields, this way of they know what they want , 
Ex: JSON-RPC call from an clien to call tool on a mcp server:
```
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


Flow of Client and server:

1. Host start up , connect to mcp server/s , one client per server
2. Client calls tools/list , get all the available tools schema from server and passes to LLM as context
3. User types a request , LLM decided whether it need to call a tool , if yes then give the client the tool it need to call with args
4. Client ask the server to call the tool by pass the tools/call 
5. Server receives the message from client over prooocl , then runs the function and thne return the output schema back to client
6. client then  passes the result back up.






## Tools, Agents, Subagents, MCP

**Tool**: The most basic unit : A tool does not "decide" anything , instead its given certain input and it's deterministic code that runs when called and returns a result.
Eg: read_file tool accepts a filename and retunr the content of the file , similarly web_search tool etc 

**Agents**: The decision-making "loop" that perceive a situation, decide what to do and take actions toward a certain "goal".
It has access to tools , make autonomous decisions, without a human manually driving each step. 

Agent uses **ReAct** pattern:
Intead of directly finally producing the answer instead:
1. Reason: The model thinks about what it knows and what it should do next
2. Action — Based on this reasoning, the model calls a tool or takes a step
3. Observe — The result of that action is fed back into the context
4. Repeat in a loop until it has enough information to produce a final answer



Example Query: What is the population density of the most populated city in a country where the most recent olympics were held ?
```
AI: call the web_search_tool("In which country recent olympics were held?")
received: France
AI : call web_search_tool("Most populated city of France")
received : Paris
AI: call web_search_tool("Population m2 of Paris")
recived: 21092846
AI : call calculate_density_tool(area: 21092846)
received: 144

Final Answer : the population density  of Paris, France is 144
```

_Note: How with each step AI is perceiving the info and deciding what it need to do in a loop until it acheives the goal_


**Subagent** : Not architecturally different from an agent at all ,  it's just an agent that's been wrapped and exposed as a tool to a parent/main agent, and per the "agent as tool" pattern, and it has its own tools it needs. 


**MCP**: A container/exposer of tools. The server itself isn't intelligent — it's just infrastructure standardizing how a set of tools gets discovered and called. One server can hold many tools. 


**RAG(Retreival Augmented Generation)** is used to give/connect LLM to new information that it does not see during its pre training phase and providing that as context to generate output grounded on that infornation. For example connecting to your database etc. But MCP "interacts"



## Blogs I found useful :

- "I think “agent” may finally have a widely enough agreed upon definition to be useful jargon now" :  https://simonwillison.net/2025/Sep/18/agents/

- Google cloud MCP SERVER: https://cloud.google.com/discover/what-is-model-context-protocol

- JSON-RPC: https://apxml.com/courses/getting-started-model-context-protocol/chapter-1-architecture-and-fundamentals/json-rpc-message-structure







