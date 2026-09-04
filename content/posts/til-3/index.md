+++
title = "Secure Agentic AI on Kubernetes : Sidecar Pattern, Gateway, LLM-as-a-Judge, GitOps"
date = 2026-09-03
draft = true
show_reading_time = true
omit_header_text = true
toc = true
description: "Securing agentic AI on Kubernetes with the sidecar pattern,and LLM-as-a-judge, Gitops and walk through using the real GitHub MCP prompt injection attack."
tags: ["kubernetes", "ai-agents", "security", "sidecar-pattern", "mcp", "prompt-injection", "gitops", "zero-trust"]
images = [""]
+++

We're in the middle of the biggest shift in how we build software. Traditionally, software is very deterministic: whatever pre-defined instructions you write, your software behaves exactly like that. With deterministic software, it's pretty intuitive to secure it, because you mostly know the points of failure and you secure them.

But this shift has been tremendous with the coming of agentic AI that uses large language models. The old rules still apply, but they're not enough anymore, and figuring out *why* they're not enough and *how* you can fix it is basically what this blog is about.
<!--more-->

## What are agents?

An agent is a system that has a decision-making "loop": it perceives a situation, decides what to do, and takes actions toward a certain goal. It has access to tools and makes autonomous decisions, without a human manually driving each step.

Agents use the ReAct pattern to get there. Instead of just jumping straight to a final answer, it loops through:

- **Reason**: the model thinks about what it knows so far and what it should do next
- **Act**: based on that reasoning, it calls a tool or takes a step
- **Observe**: the result of that action gets fed back into the context
- Repeat until it has enough information to give a final answer

**Example:** "Book me a flight to Toronto for September 10th"

```
Agent: reasons, calls list_flights(date="Sept 10")
Agent: observes the results, reasons about which flight fits
Agent: calls book_flight(flight_id=...)
Agent: observes the confirmation, responds to the user
```

In this example, see how, without any human intervention, the agent starts and based on the information it gets back, it reasons and takes actions by calling tools. 

Because of this autonomy, we get non-deterministic behavior and that leads to a whole new category of security concerns.

In a traditional system, we kept **data** and **instructions** separate. With agents, that line blurs completely: whatever "data" the LLM pulls in can end up being used as the *next set of instructions* it reasons and acts on. Imagine someone prompt-injecting your agent into leaking all your financial data publicly, without you ever agreeing to it. This isn't hypothetical: it's already happened to a very real, very big target.

## The Real Attack: GitHub MCP Cross-Repository Data Leak (May 2025, disclosed by Invariant Labs)

Here's what happened the user is using an MCP client (like Claude Desktop) with the GitHub MCP server connected to their account. The user has two repositories:

- `<user>/public-repo`: publicly accessible, anyone on GitHub can create issues on it
- `<user>/private-repo`: private, holding proprietary code / private company data

The attacker places a malicious issue in the public repo. That issue contains hidden instructions telling the agent to pull data from the private repo into context and leak it by autonomously creating a pull request on the public repo.

The user innocently asks their agent: *"Have a look at the issues in my public repo and address them."*

That's where the naive request goes sideways. The agent goes through the list of issues, hits the attacker's payload it willingly pulls private repo data into its context and leaks it into a public PR, freely accessible to the attacker.

With this much autonomy handed to agents, the *data itself* becomes the attack surface. If it can happen to a company the size of GitHub's user base, it can absolutely happen to your agent too.

## The Sidecar Pattern

So how do we actually build infrastructure that's dynamic and resilient enough for this new kind of workload?

We go back to fundamentals. The needs are changing, but the underlying concepts stay the same, just with a different flavor. One of those fundamentals: **the principle of least privilege**, giving users, programs and systems only the bare minimum permissions they need to do their job and nothing more.

That's exactly what we need here. So what's a sidecar?

**Sidecar containers** are secondary containers that run alongside the main application container, within the same Pod. They extend the primary app's functionality (logging, monitoring, security) while sharing the same lifecycle and resources as the main container.

**Why do we need one, and what goes inside it?**

The main idea is separation of concerns between your infrastructure layer and your business logic. Say you have a set of security rules: you don't want to hardcode those rules inside every single app across your org. If you ever need to change the policy, you'd have to go update the code inside every app you own. Instead, the better pattern: write that logic ONCE, into a sidecar, and attach it to every app that needs it. Change the policy once, in one place, and every app that uses that sidecar picks up the change. That's how you keep **STANDARDIZATION** across your whole org.

We use this exact pattern when deploying agents.

Here's the thing: 
1. Secret mamagement: your agent needs to interact with the real world by making tool calls, say, a `read_repo` tool call against a GitHub MCP server. You do NOT want to hand your agent's own code the raw secrets to do that. Imagine the agent gets manipulated (see: the GitHub attack above); now it has direct access to everything, and nothing is standing in the way.

2. Roles and permissions: And in a multi-agent system, you don't want to give every agent access to every tool. Your finance agent might only need `read_balance`; your GitHub agent has zero business touching that tool at all. Giving everyone access to everything just doesn't make sense.

That's exactly where the sidecar comes in. But before we get into how it works, let's define a few building blocks.

### Service Account: giving your agent an identity

Think of a ServiceAccount as a non-human identity. Since you'll normally run one agent per Pod, this is how you give that specific agent a real, verifiable identity.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: finance-agent-sa
  namespace: ai-agents
```

That's it: this creates the identity. 

### Roles & RoleBindings: attaching permissions to that identity

Just like a human account can have a certain level of permission, you attach permission levels to this ServiceAccount too. Example: this account is allowed to read a specific secret, like a GitHub token or a bank-account API token, nothing more.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: finance-agent-role
  namespace: ai-agents
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["finance-refresh-token"]
    verbs: ["get"]
```

One important nuance: this is Kubernetes-level permission. It controls what the *sidecar* can touch inside the Kubernetes API (like reading one specific Secret object). It says nothing about which business-level tools (like `read_balance` vs `make_payment`) the agent itself is allowed to call. That's a separate concern, which brings us to the next building block.

### Custom CRD: teaching Kubernetes about tool-level permissions

Kubernetes RBAC has zero concept of "tools" like `read_balance` or `make_payment`; those are business concepts we invented, meaningless to Kubernetes by default. So we define our own **Custom Resource Definition (CRD)** that teaches Kubernetes a new object type, call it `ToolPermission`, so we can declare, per agent, exactly which tools it's allowed to call.

And then, per agent, you create an instance of it:

```yaml
apiVersion: agents.mycompany.com/v1
kind: ToolPermission
metadata:
  name: finance-agent-permissions
  namespace: ai-agents
spec:
  serviceAccount: finance-agent-sa
  allowedTools:
    - read_balance
```

### Secret Management: short-lived tokens, not permanent passwords

Now to actually make those tool calls, you need a token. So never give your agent access to a token directly. Instead, the sidecar holds that responsibility.

Now, inside the sidecar itself: it fetches long-lived credentials from a secrets manager (like HashiCorp Vault) and instead of using that long-lived secret directly, it trades it for a **short-lived access token**. That short-lived token is what actually gets used to call the real tool and bring the result back.

Now the agent just talks to its sidecar over `localhost`, and the sidecar handles everything behind the scenes.

### Putting it together: the request flow



## What Happens on Prompt Injection? LLM-as-a-Judge

Here's the next problem: everything above protects against "is this agent ALLOWED to do this." It does nothing about the actual GitHub attack we walked through earlier, where the agent was fully authorized, used a completely legitimate tool call, and STILL got tricked, because the danger was hiding inside the *content* it read, not in whether the call itself was permitted.

This is where we can add one more job to the sidecar: before letting a tool's response reach the agent, run it through a small LLM whose only job is to judge: *"does this content look like it's trying to inject new instructions into an agent?"*


This is called LLM-as-a-judge pattern where you use one LLM to judge another LLM answer. If you want to learn more about it I put together a video of it as well , check it out [YOUTUBE]

## GitOps: Making This Organization-Wide

All of this (ServiceAccounts, Roles, ToolPermissions, the sidecar config) is a lot to get right, every single time, for every single agent. If it's left to individual developers to configure by hand, someone eventually forgets a step, or gets it slightly wrong.

Also, if someone's manually running `kubectl apply` every time, there's no observability into who changed what, and why. We already have that kind of visibility for our application code through version control, we want the same for our infrastructure code. That's where GitOps comes in.

**GitOps** is the fix: instead of developers manually running `kubectl apply` themselves, they just write a description of what they want (the YAML files above) and commit it to a Git repository. A separate automated controller (tools like ArgoCD or Flux) constantly watches that repo, compares it against what's actually running in the cluster, and automatically applies any differences, and just as importantly, automatically reverts anything that was changed by hand outside of Git.

The result: there's no path to deploy an insecure agent by accident, because there's no path to deploy ANYTHING except through this one reviewed, automated pipeline. Security stops being something developers have to remember, and becomes something the platform enforces by default.