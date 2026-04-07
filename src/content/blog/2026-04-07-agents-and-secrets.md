---
title: Agents and secrets
description: Ways to give agents access to authenticated services, from least safe to most secure.
pubDate: 2026-04-07
---

This is a short rundown of ways to give an agent access to external services that require auth, especially headless agents that run **autonomously**. These are ordered from least safe to most secure.

Since there is an overall shift from MCP to CLI, this focuses mostly on CLI.

### Just give it the key

> Use Bloomberg API to build the best portfolio. Here is the API key you can use: xxxx-xxxx

You just give the agent your API key. This is the easiest option.

Problems:

- the agent may inadvertently leak the key somewhere
- the key sits in the transcript, which may be stored and read later
- the LLM provider, such as Anthropic or OpenAI, gets the key too

### Put it in an env variable

> Use Bloomberg API to build the best portfolio. API key is in the `$BLOOMBERG_KEY`

The key is not intentionally exposed to the LLM provider, but the agent can still inspect the variable to verify it exists.

It can also leak indirectly. If it uses something like `curl -v`, it may expose the key in verbose output or headers.

### Make a wrapper script

> Use Bloomberg API to build the best portfolio. Access the API via `bloomberg-curl` script, it is a curl wrapper that adds the right auth headers. Otherwise you can use it like curl.

In practice, this is often the sweet spot for local agents. The wrapper can redact the key from stdout and stderr, so verbose curl output is less of a problem.

The issue is what happens when the wrapper breaks. The agent may try to fix it, and eventually may try to inspect how the script gets the key.

The fundamental problem is that the wrapper script is fully readable by the agent, and the key still lives in the same environment.

### Make a wrapper binary

> Use Bloomberg API to build the best portfolio. Access the API via `bloomberg-curl` binary, it is a curl wrapper that adds the right auth headers. Otherwise you can use it like curl.

Assuming Linux, the wrapper should be an ELF binary with permissions set so the agent can execute it but not read it.

This can be a good middle ground without much maintenance. The binary still needs some way to access the secret, so the secret is only obscured, not removed from reach entirely.

Generally this is fairly agent-proof. It is a reasonable option for internal tools where users are unlikely to intentionally break isolation or prompt-inject the system.

### Move the secret off-site

> Use Bloomberg API to build the best portfolio. Access the API via `bloomberg` binary that acts like curl.

Here the binary routes requests to a proxy, for example `https://bloomberg.internal.your-company.com`, and adds tracing headers like `Agent-Trace-Id: xxx`.

A good idea is to also require an intent header, for example:

> `Agent-Intent: "buying stocks to make you rich"`

The proxy runs separately from the agent and injects the API key.

If the proxy is integrated with the agent control plane, it can also ask for confirmations through a side channel. For example, you might get a Slack message asking whether to allow `/purchase` with a given intent, while the agent just waits.

### MCP

Of course, remote MCP servers are basically the embodiment of this proxy idea, but they come with tradeoffs.

- the agent cannot compose MCP invocations as naturally as shell commands like `bloomberg '/get-top-tickers' | head -n 5`
- the model cannot lean as much on its existing knowledge of bash, Python, and other languages for processing, batching, looping, or waiting
- MCP tool exposure is often token-wasteful
- if you choose MCP, it is often because the agent does not have shell access in the first place
