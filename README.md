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
\.venv\Scripts\Activate.ps1
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