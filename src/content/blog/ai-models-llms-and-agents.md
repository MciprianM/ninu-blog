---
title: 'AI Models, LLMs, and AI Agents'
description: "What's the difference, and what actually happens inside an AI agent?"
pubDate: 'Apr 18 2026'
---

As AI tools become part of everyday engineering work, knowing what's actually happening under the hood helps you use them better — and build with them. Let's start with the vocabulary.

**AI model**, **LLM**, and **AI agent** get used interchangeably, but they describe different things at different levels of abstraction. Think of them as nested concepts: every LLM is an AI model, but most AI models aren't LLMs. Every AI agent uses a model (often an LLM), but a model on its own isn't an agent.

## AI Model

An AI model is the broadest category. It's a mathematical system trained on data to recognize patterns and make predictions or generate outputs. This includes everything from a simple spam classifier to image recognition systems to speech-to-text engines. An AI model takes an input, runs it through learned parameters (weights), and produces an output. It's passive — it responds when called, and doesn't do anything on its own.

Examples: a model that detects tumors in X-rays, a recommendation model on Netflix, a chess-playing model, a language model.

## LLM (Large Language Model)

An LLM is a specific *type* of AI model — one trained on massive amounts of text to predict and generate language. "Large" refers to the scale: billions or trillions of parameters, trained on huge text datasets. LLMs can write, summarize, translate, answer questions, and reason through problems in natural language.

Examples: Claude, GPT-4, Gemini, Llama.

## AI Agent

An AI agent is a *system* built around a model (usually an LLM) that can take actions in the world to accomplish goals. Where an LLM just responds to a prompt, an agent can plan, use tools (search the web, run code, send emails, query databases), observe the results, and decide what to do next — looping until the task is done. The LLM is the "brain"; the agent is the whole body around it, including memory, tools, and control flow.

Examples: a coding assistant that writes, tests, and debugs code across multiple files; a research agent that searches sources, reads them, and compiles a report; a customer-service agent that looks up orders and issues refunds.

An agent isn't necessarily smarter than the LLM inside it — it's just given more autonomy and more tools. The quality of an agent depends heavily on the quality of the underlying model.

---

## Simulated Agent Trace: Writing `add.cpp`

To make this concrete, here's a step-by-step walkthrough of what happens inside an AI agent handling the prompt:

> *"Write a C++ function that adds two ints and save it as add.cpp."*

The agent works in a loop. Each iteration: the agent **calls the LLM** (which returns either a tool call or a final answer), then **runs the tool** if one was requested. The loop exits when the LLM responds with a plain message and no tool call.

### Setup

Three actors are involved:

- **You** — send the prompt, receive the final answer.
- **Agent** — the orchestrator: maintains the conversation, executes tools, loops.
- **LLM** — stateless: given a conversation, produces the next message.

Available tools:

- `write_file(path, content)` — creates or overwrites a file on disk
- `read_file(path)` — returns the contents of a file
- `list_dir(path)` — lists the entries in a directory

Messages in the conversation carry a role label:

- `system` — setup instructions
- `user` — your input
- `assistant` — the LLM's output
- `tool` — a tool's result

Because the LLM has no memory, the agent resends the full conversation on every call.

---

You send the prompt:
```
Write a C++ function... save as add.cpp
```

### Iteration 1 — check the folder

**Agent → LLM (request)**
```
[system]  You are a coding agent.
  Tools: write_file, read_file, list_dir.
[user]  Write a C++ function... save as add.cpp
```

**LLM → Agent (response)**
```
[assistant]  I'll check the folder first.
  <tool_call> list_dir({ "path": "." }) </tool_call>
```

**Agent runs the tool**

The agent executes the `list_dir` call. Output:
```
["README.md", "notes.txt"]
```

### Iteration 2 — write the file

Notice the conversation is growing — the agent appends the tool result and resends *everything* to the LLM.

**Agent → LLM (request)**
```
[system]  You are a coding agent. Tools: ...
[user]  Write a C++ function... save as add.cpp
[assistant]  I'll check the folder first.
  <tool_call> list_dir({ "path": "." }) </tool_call>
[tool]  ["README.md", "notes.txt"]
```

**LLM → Agent (response)**
```
[assistant]  <tool_call>
  write_file({
    "path": "./add.cpp",
    "content": "int add(int a, int b) {\n..."
  })
</tool_call>
```

**Agent runs the tool**

The agent executes the `write_file` call. Output:
```
{ "status": "success", "bytes_written": 42 }
```

### Iteration 3 — verify the result

A careful LLM doesn't trust a `"success"` status blindly — the write could have failed silently or produced unexpected content, so it reads the file back to verify.

**Agent → LLM (request)**
```
[system]  You are a coding agent. Tools: ...
[user]  Write a C++ function... save as add.cpp
[assistant]  I'll check the folder first.
  <tool_call> list_dir({ "path": "." }) </tool_call>
[tool]  ["README.md", "notes.txt"]
[assistant]
  <tool_call> write_file("./add.cpp", ...) </tool_call>
[tool]  { "status": "success", "bytes_written": 42 }
```

**LLM → Agent (response)**
```
[assistant]
  <tool_call> read_file("./add.cpp") </tool_call>
```

**Agent runs the tool**

The agent executes the `read_file` call. Output:
```
int add(int a, int b) {
    return a + b;
}
```

### Iteration 4 — nothing left to do

**Agent → LLM (request)**
```
[system]  You are a coding agent. Tools: ...
[user]  Write a C++ function... save as add.cpp
[assistant]  I'll check the folder first.
  <tool_call> list_dir({ "path": "." }) </tool_call>
[tool]  ["README.md", "notes.txt"]
[assistant]
  <tool_call> write_file("./add.cpp", ...) </tool_call>
[tool]  { "status": "success", "bytes_written": 42 }
[assistant]
  <tool_call> read_file("./add.cpp") </tool_call>
[tool]
  int add(int a, int b) {
      return a + b;
  }
```

**LLM → Agent → You (final response)**
```
[assistant]  Done. I created ./add.cpp containing:

  int add(int a, int b) {
      return a + b;
  }
```
No tool call → the agent exits the loop and delivers this message to you.

---

### The pattern

This is the **ReAct** loop — **Re**ason, then **Act**. Harder tasks run more iterations of the same shape. And the loop really shines when things go wrong: if the agent reads the file back and the code has a mistake, it can feed the problem back to the LLM, get a corrected version, and rewrite the file — same loop, more iterations.

Because every tool output is appended and resent, long runs can bump against the model's **context window** (the maximum amount of text it can process in a single call); real agents manage that by summarizing or dropping old messages.
