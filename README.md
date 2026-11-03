# Foundry Agent MCP Wrapper

A small Python MCP server that exposes an existing Microsoft Foundry agent as an MCP tool. An MCP client sends a code excerpt and a review question to the server; the server forwards them to the configured Foundry agent and returns the agent's review.

This project is a transport wrapper, not an agent host or vulnerability scanner. The Foundry project and agent must already exist, and the agent's instructions determine the review format and behavior.

## How It Works

1. The server reads the Foundry project endpoint, agent name, and optional code-size limit from environment variables (including a local `.env` file).
2. `DefaultAzureCredential` authenticates the server to Azure.
3. The server gets an OpenAI client for the named Foundry agent.
4. The MCP tool `review_code_with_agent1` validates and packages the caller's question, filename, and code excerpt, then calls the agent.
5. The response is parsed as a JSON object when possible. If the agent returns non-JSON text, the wrapper returns it in `review.raw_review`.

## Requirements

- Python installed on the machine running the server.
- Access to a Microsoft Foundry project and an existing agent in that project.
- An identity supported by `DefaultAzureCredential`, with permission to use the project and agent. For local development, one option is to sign in with Azure CLI using `az login`.
- The project endpoint and agent name.

## Configure

From the project directory, create a `.env` file:

```dotenv
FOUNDRY_PROJECT_ENDPOINT=https://<your-project-resource>.services.ai.azure.com/api/projects/<your-project>
FOUNDRY_AGENT_NAME=<your-agent-name>
# Optional. Defaults to 20000 characters.
FOUNDRY_MAX_CODE_CHARS=20000
```

Use the project endpoint shown for your Foundry project. Do not put credentials, keys, or tokens in this file. Keep `.env` out of source control.

## Install and Run

In PowerShell, from the project directory:

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
az login
python server.py
```

The server uses MCP stdio transport. When launched directly, it waits for an MCP client on standard input and writes the MCP protocol on standard output; it is not an HTTP server. To use it from an MCP client, configure that client to launch `server.py` as a stdio server rather than leaving the command running in a separate terminal.

## Configure an MCP Client

Add a server entry in the MCP client's configuration. This example uses the common `mcpServers` shape; use the equivalent location and schema for your client. Replace the paths and Foundry values with your own:

```json
{
  "mcpServers": {
    "foundry-code-review": {
      "command": "C:\\dev-tools\\foundry-agent-mcp\\.venv\\Scripts\\python.exe",
      "args": [
        "C:\\dev-tools\\foundry-agent-mcp\\server.py"
      ],
      "env": {
        "FOUNDRY_PROJECT_ENDPOINT": "https://<your-project-resource>.services.ai.azure.com/api/projects/<your-project>",
        "FOUNDRY_AGENT_NAME": "<your-agent-name>",
        "FOUNDRY_MAX_CODE_CHARS": "20000"
      }
    }
  }
}
```

The client process also needs a working `DefaultAzureCredential` identity. For example, sign in with Azure CLI in the same user context before starting the client. Restart or reload the MCP client after changing its configuration.

## Use in VS Code Copilot

Registering this MCP server makes its tools available to Copilot; it does not automatically add the remote Foundry agent to the Chat agent dropdown. You can use the tool from ordinary **Agent** mode without a custom agent file. To give this integration its own named dropdown entry, create a VS Code custom agent.

### Register the MCP Server

In the workspace where you want to review code, add the following to `.vscode/mcp.json`. VS Code uses `servers`, not the generic `mcpServers` key shown above. Replace the paths and Foundry values:

```json
{
  "servers": {
   "foundry-code-review": {
    "type": "stdio",
    "command": "C:\\dev-tools\\foundry-agent-mcp\\.venv\\Scripts\\python.exe",
    "args": ["C:\\dev-tools\\foundry-agent-mcp\\server.py"],
    "env": {
      "FOUNDRY_PROJECT_ENDPOINT": "https://<your-project-resource>.services.ai.azure.com/api/projects/<your-project>",
      "FOUNDRY_AGENT_NAME": "<your-agent-name>",
      "FOUNDRY_MAX_CODE_CHARS": "20000"
    }
   }
  }
}
```

Start the server using the **Start** action in the MCP configuration editor, approve any trust prompt after reviewing the configuration, and confirm that `review_code_with_agent1` is available in Chat's tool picker. Authentication must already be configured as described above.

### Add a Dropdown Agent

Create `.github/agents/foundry-reviewer.agent.md` relative to the workspace root. The standard workspace location is `.github/agents`, not `.agents`. Put the file in the project you are reviewing; it does not have to live beside this wrapper's Python source.

```markdown
---
name: Foundry Code Reviewer
description: Review code quality and architecture using the configured Microsoft Foundry agent.
tools: ['read', 'search', 'foundry-code-review/*']
---

Review code using the MCP tool review_code_with_agent1.

1. Identify the user's review question and the selected code or requested file.
  Ask for missing context when necessary.
2. Read only the relevant code. Do not send secrets or unrelated files.
  Code sent to the tool leaves this machine for the configured Foundry project.
3. Call review_code_with_agent1 with developer_question, code_context, and filename.
  Keep each excerpt within the configured FOUNDRY_MAX_CODE_CHARS limit
  (20,000 characters by default). Ask the user to narrow oversized requests.
4. Present the returned findings and prioritized actions, with file references
  where supported by the result. Do not modify files or claim vulnerability scanning.
5. If the tool is unavailable or fails, report that failure explicitly.
  Do not present a local review as a successful Foundry review.
```

The `foundry-code-review/*` tool entry must match the server key in `.vscode/mcp.json`. It enables tools from that MCP server; `read` and `search` let Copilot gather the requested code. The displayed `name` can differ from `FOUNDRY_AGENT_NAME`: the former names the Copilot persona, while the latter selects the remote Foundry agent.

Open Copilot Chat, select **Foundry Code Reviewer** from the agent dropdown, attach the relevant file or selection, and ask for a review. If the entry does not appear, check the file location and YAML frontmatter, then reload the VS Code window. This custom agent still uses a Copilot model to coordinate the request; the review itself is delegated to Foundry through the MCP tool.

See the VS Code documentation for [custom agents](https://code.visualstudio.com/docs/copilot/customization/custom-agents) and [MCP servers](https://code.visualstudio.com/docs/copilot/customization/mcp-servers).

## Use the Tool

Call `review_code_with_agent1` with:

| Argument | Required | Description |
| --- | --- | --- |
| `developer_question` | Yes | What you want the agent to review. Whitespace-only input is rejected; the wrapper forwards at most 2,000 characters. |
| `code_context` | Yes | The code or file excerpt to review. Whitespace-only input is rejected. Its size must not exceed `FOUNDRY_MAX_CODE_CHARS` (20,000 by default). |
| `filename` | No | A label for the supplied code. Defaults to `not-provided`; the wrapper forwards at most 500 characters. |

Example request:

```json
{
  "developer_question": "Review this function for maintainability and suggest focused improvements.",
  "filename": "src/parser.py",
  "code_context": "def parse_record(value):\n    ..."
}
```

The result includes the configured `agent_name`, the `filename`, `characters_reviewed`, and a `review` object. When the agent returns valid JSON, that JSON object is used as `review`. Otherwise, `review` contains a summary and the original text in `raw_review`.

Only send code and context that you are authorized to share with the configured Foundry project. The wrapper forwards the supplied code to the agent; it does not redact secrets or scan for vulnerabilities. Do not include credentials or other sensitive values in a review request.

## Legal Disclaimer

This repository is provided for educational and demonstration purposes only. The code, documentation, and any agent-generated output are provided "AS IS" and "AS AVAILABLE", without warranty of any kind, express or implied, including warranties of merchantability, fitness for a particular purpose, and noninfringement.

To the maximum extent permitted by applicable law, the author and contributors shall not be liable for any claims, damages, losses, costs, or other liability arising from or related to the use, misuse, modification, distribution, or inability to use this repository or its outputs, whether in contract, tort, or otherwise. This includes data loss, security incidents, service interruptions, cloud charges, and reliance on inaccurate or incomplete AI-generated output.

You are responsible for evaluating, testing, securing, and operating the code; reviewing all AI-generated output before relying on it; obtaining permission to share submitted data; and complying with applicable laws, organizational policies, and third-party service terms. Use this repository at your own risk. Nothing in this disclaimer excludes or limits liability that cannot lawfully be excluded or limited.

This disclaimer does not itself grant a license to use, modify, or redistribute the code.

## Troubleshooting

- **Missing environment variable:** Confirm `FOUNDRY_PROJECT_ENDPOINT` and `FOUNDRY_AGENT_NAME` are present in the process environment or in the `.env` file loaded from the project directory.
- **Authentication or authorization failure:** Confirm `DefaultAzureCredential` can obtain a token and that the signed-in identity can access the Foundry project and agent.
- **Agent not found:** Check that the configured agent name matches an existing agent available in the configured project.
- **Input too large:** Send a smaller excerpt or increase `FOUNDRY_MAX_CODE_CHARS` to an appropriate limit.
- **Client cannot start the server:** Check the Python executable and `server.py` paths in the MCP client configuration, then reload the client and inspect its MCP server logs.