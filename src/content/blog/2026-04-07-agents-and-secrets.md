---
title: Agents and secrets
description: Ways you can give an agent access to external services that require auth.
pubDate: 2026-04-07
---

# Agents and secrets

This is a short run down on ways you can give an agent access to external services that require auth, especially headless agents that run **autonomously**. From least safe to the most secure.

Since there is overall shift from MCP to CLI, I am focusing mostly on CLI here.

### Just give it the key

> Use Bloomberg API to build the best portfolio. Here is the API key you can use: xxxx-xxxx

You just give the agent your API key, this is the easiest of course. The agent will figure out how to use it.
This has the following possibilities:
- agent may inadvertedly leak the key somewhere
- you are leaving the key in the transcript which, depending on implementation, may be stored for a while and read later (with intention to improve the agent, but nevertheless the key will be there)
- LLM provider (e.g. Anthropic, OpenAI) gets your API keys

### Put it in an env variable

> Use Bloomberg API to build the best portfolio. API key is in the $BLOOMBERG_KEY

The key is not exposed to the LLM provider intentionally, but the agent can still decide to inspect the variable to make sure the key is there. Similarly if it uses something like `curl -v …` (verbose mode) it will unintentionally see the key in the output of cURL in the headers.

### Make a wrapper script

> Use Bloomberg API to build the best portfolio. Access the API via `bloomberg-curl` script, it is a curl wrapper that adds the right auth headers, otherwise you can use it like curl.

In practice this tends to be the sweet spot for local agents. The wrapper can take care of redacting the key from stdout and stderr as well, so the verbose cURL mode is not an issue.

However what tends to happen if the wrapper is for some reason not working, is that the agent will start trying to fix it, and eventually might try to access the API key itself, based on how that script does it.

The fundamental problem here is that the wrapper script is fully accessible and readable to the agent, and that the API key lives on the same environment where agent operates.

### Make a wrapper binary

> Use Bloomberg API to build the best portfolio. Access the API via `bloomberg-curl` binary, it is a curl wrapper that adds the right auth headers, otherwise you can use it like curl.

(assuming Linux)
The wrapper must be a ELF binary, and the file permissions must be set such that the agent cannot read that binary but can execute (111 mask).

This can be great middle ground, that doesn’t require a lot of maintenance. The binary still needs some way of accessing the secret, it is well-obscured from the agent, but if agent finds a way to discover it (eg maybe by simply enumerating all possible options), it will still be able to access the secret.

Generally this is quite agent-proof, it can be a good option for e.g. internal tools where you can assume with some certainty that the users will not attempt to hack anything, and won’t get prompt-injected.

### Move the secret off-site

> Use Bloomberg API to build the best portfolio. Access the API via `bloomberg` binary that acts like curl.

Here the idea is that the binary routes requests to a proxy, eg: `https://bloomberg.internal.your-company.com` and adds a tracing header eg `Agent-Trace-Id: xxx`. A good idea is also to instruct the agent to include a header that explains the intent, eg `Agent-Intent: "buying stocks to make you rich”`.

The proxy, which runs separately from the agent, injects the API key. If the proxy is integrated with your agent control plane, it can even ask you for confirmations on API calls via a side-channel (eg as a user you would receive a Slack message asking if you “give the agent authority to invoke endpoint `/purchase`, with the intent 'buying stocks to make you rich’”; while the agent simply waits).

### MCP

Of course, remote MCP servers are basically the embodiment of this proxy idea, but they come with limitations.

The agent can’t compose MCP invocations the same way it can do `bloomberg '/get-top-tickers' | head -n 5` for example. (though this could change, if MCP spec is improved 😉)
The LLM can’t build up on its already baked-in knowledge of bash/Python/other languages for processing data, batching API calls, looping, waiting, etc.
The way MCP clients expose available tools to the agent tends to be quite token-wasteful.
If you are going for MCP it is more likely to do with the fact that your agent doesn’t have the shell access in the first place.
