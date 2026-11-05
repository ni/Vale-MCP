# Vale MCP Server

Vale MCP Server exposes [Vale](https://vale.sh/) prose linting through the Model Context Protocol (MCP). It supports two transports:

- **stdio** for desktop MCP clients that launch a local process.
- **Streamable HTTP** for clients that connect to a URL, including AI Positron in Oxygen XML Web Author.

The tool implementations are shared. `index.ts` creates and registers the Vale tools; `index.ts` starts stdio when run directly, while `http.ts` creates a server instance for each HTTP MCP session.

The NI fork of Vale MCP Server has added the Streamable HTTP capability as well as changes for it to run in a Windows environment.

## Requirements

- Node.js 22 or later and npm.
- Vale CLI 3.0 or later installed on the machine that runs the MCP server and available on `PATH`.
- A Vale configuration file (`.vale.ini`) and any required styles/packages. These must be available to the MCP host, not just to the desktop client or Web Author browser.
- Network access from the MCP host to package sources when running `vale sync`.

Install Vale using the instructions for your operating system in the [Vale installation documentation](https://vale.sh/docs/vale-cli/installation/).

## Install and Build

From the project directory:

```sh
npm install
npm run build
```

The TypeScript output is written to `build/`. Confirm that Vale is available:

```sh
vale --version
```

Configure Vale before starting the server. For example, `.vale.ini` can be placed in the process working directory:

```ini
StylesPath = styles
Packages = write-good, proselint

[*]
BasedOnStyles = write-good, proselint
```

Download configured packages:

```sh
vale sync
```

Alternatively, set `VALE_CONFIG_PATH` to an absolute path to the configuration file. The server checks configuration in this order:

1. `VALE_CONFIG_PATH`, when set and readable.
2. `.vale.ini` in the server process working directory.
3. Vale's own defaults and configuration discovery.

For `check_file`, Vale is also allowed to discover configuration from the target file's directory. `check_text` uses the server configuration unless a `config_path` is supplied to the tool.

## Desktop Deployment

### Local stdio server

Use stdio when the MCP client and Vale MCP run on the same desktop. Build the project, then configure the client to launch Node with the built entry point. Example MCP client configuration:

```json
{
  "mcpServers": {
    "vale": {
      "command": "node",
      "args": ["C:/path/to/Vale-MCP/build/index.js"],
      "env": {
        "VALE_CONFIG_PATH": "C:/path/to/project/.vale.ini"
      }
    }
  }
}
```

Use absolute paths appropriate to the computer. Omit `VALE_CONFIG_PATH` if the server should find `.vale.ini` from its working directory or use Vale's defaults. Some clients allow setting a working directory; otherwise, an explicit `VALE_CONFIG_PATH` is the most predictable option.

The stdio entry point also supports:

```sh
node build/index.js --help
node build/index.js --version
node build/index.js --debug
```

Keep standard output reserved for MCP messages. This server writes status and diagnostic messages to standard error.

### Local Streamable HTTP server

For a desktop MCP client that supports Streamable HTTP, start the HTTP entry point on loopback:

**PowerShell:**

```powershell
$env:MCP_HOST = "127.0.0.1"
$env:MCP_PORT = "3000"
$env:MCP_ENDPOINT = "/mcp"
$env:VALE_CONFIG_PATH = "C:\path\to\project\.vale.ini"
npm run start:http
```

**Linux or macOS shell:**

```sh
MCP_HOST=127.0.0.1 MCP_PORT=3000 \
  VALE_CONFIG_PATH=/path/to/project/.vale.ini npm run start:http
```

Configure the client with the HTTP endpoint, if it supports remote MCP servers:

```json
{
  "mcpServers": {
    "vale": {
      "type": "http",
      "url": "http://127.0.0.1:3000/mcp"
    }
  }
}
```

The defaults are host `127.0.0.1`, port `3000`, and endpoint `/mcp`. `MCP_ENDPOINT` may be set to another path. The HTTP server handles MCP `POST`, `GET`, and `DELETE` requests and maintains session transports in process memory.

## Remote Deployment for Oxygen Web Author

AI Positron in Oxygen XML Web Author connects to a **remote Streamable HTTP endpoint**. Web Author does not launch a local stdio process. The Oxygen Web Author server must be able to reach the MCP endpoint over the network.

### 1. Prepare the Vale host

Install Node.js, npm, and Vale on a server that can remain available to Web Author. Copy or clone the Vale MCP project to that host and build it:

```sh
npm ci
npm run build
vale --version
```

Provide a `.vale.ini` and install the configured styles on the host. For example:

```sh
export VALE_CONFIG_PATH=/opt/vale-mcp/.vale.ini
vale sync --config="$VALE_CONFIG_PATH"
```

The configuration and styles must remain accessible to the service account that runs the MCP server.

### 2. Run the HTTP process

For a local test on the server:

```sh
MCP_HOST=127.0.0.1 MCP_PORT=3000 \
  VALE_CONFIG_PATH=/opt/vale-mcp/.vale.ini npm run start:http
```

For production, keep the Node process bound to loopback and place a reverse proxy or gateway in front of it. Publish an HTTPS URL such as:

```text
https://vale-mcp.example.com/mcp
```

Configure the proxy to forward the MCP HTTP methods (`POST`, `GET`, and `DELETE`) to the Node process, preserve MCP headers such as `Mcp-Session-Id`, and support streaming responses. Disable response buffering for the MCP route and use suitable idle timeouts so server-sent event streams are not cut off. Restrict inbound access to the Web Author deployment where possible.

Run the process under a service manager (for example, `systemd`, a container orchestrator, or a Windows service manager) so it starts on boot, restarts on failure, and runs as a dedicated low-privilege account. Ensure that account can execute Vale and read only the configuration, styles, and content it needs.

Example `systemd` unit (adjust paths and account names):

```ini
[Unit]
Description=Vale MCP Streamable HTTP Server
After=network.target

[Service]
Type=simple
User=vale-mcp
Group=vale-mcp
WorkingDirectory=/opt/vale-mcp
Environment=MCP_HOST=127.0.0.1
Environment=MCP_PORT=3000
Environment=MCP_ENDPOINT=/mcp
Environment=VALE_CONFIG_PATH=/opt/vale-mcp/.vale.ini
ExecStart=/usr/bin/npm run start:http
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3. Secure the endpoint

The HTTP server in this project currently has **no built-in authentication, authorization, or file-path allowlist**. Do not expose it directly to the public internet or an untrusted network. A reverse proxy must provide HTTPS and access control, such as a bearer token accepted from an `Authorization` header, and the firewall should restrict access to trusted callers. Verify that the proxy forwards the authorization header and required MCP session headers.

Oxygen supports static request headers in its MCP configuration. Such a credential is shared by the deployment's users, not unique per user. Do not use a privileged Vale host account or place secrets in public documentation. For stronger identity-based access control, add a suitable authenticated gateway or implement and test authentication in the MCP service. The current server does not implement OAuth.

The `check_file` tool can read files accessible to the MCP process. Its `path` is interpreted on the **MCP server**, not on the user's workstation, and this implementation does not restrict paths to a workspace. Use a dedicated account and filesystem permissions to limit exposure. For content open in Web Author, prefer `check_text`, which sends the text to Vale without asking the server to open a client-side file path.

### 4. Configure AI Positron

In Oxygen Web Author, open the AI Positron plugin configuration and select the **MCP** tab. Add the configuration below, change the URL to the public HTTPS address of the MCP endpoint, and click **Apply**:

```json
{
  "mcpServers": {
    "vale": {
      "type": "http",
      "url": "https://vale-mcp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer REPLACE_WITH_YOUR_TOKEN"
      }
    }
  }
}
```

Remove the `headers` property if the trusted endpoint does not require a static header. The server name (`vale`) is used as a prefix for the tools shown by Oxygen. For example, tools may appear as `vale_check_text`, `vale_check_file`, `vale_vale_status`, and `vale_vale_sync`.

After applying the configuration, open the MCP servers and tools dialog in AI Positron. Confirm that the Vale server is connected and enable the tools needed for the current chat mode. Oxygen's MCP configuration is administered on the Web Author deployment; local stdio `command`, `args`, and `env` entries are not supported there.

## Available Tools and Example Prompts

AI assistants invoke MCP tools from natural-language requests. Use prompts that explicitly identify the desired operation and content.

### `vale_status`

Checks whether Vale is installed and reports the version and platform. If Vale is missing, the result includes installation guidance.

Example prompts:

- “Check whether Vale is installed on the MCP server.”
- “Run the Vale status check and report the installed version.”

### `vale_sync`

Runs `vale sync` to download configured style packages. It can accept an optional `config_path`.

Example prompts:

- “Run Vale sync using the server's configured `.vale.ini`.”
- “Run `vale_sync` with config path `/opt/vale-mcp/.vale.ini`.”

Only run this when the configuration is trusted and the server is allowed to access the configured package sources.

### `check_text`

Checks text sent in the tool call. Optional inputs are `text_file_ext` (for example, `.md` or `.txt`) and `config_path`.

Example prompts:

- “Use Vale to check this text as Markdown and summarize the errors, warnings, and suggestions: [paste text].”
- “Run `check_text` on the following content with `text_file_ext` set to `.md`: [paste text].”
- “Check this paragraph using `/opt/vale-mcp/.vale.ini`: [paste text].”

For long content, send only the relevant section or use `check_file` when the file is available on the MCP host.

### `check_file`

Checks a path readable by the MCP server. Required input: `path`. Optional input: `config_path`.

Example prompts:

- “Run Vale on `/srv/docs/guide.md` using the server's normal config.”
- “Check `/srv/docs/guide.md` with `/srv/docs/.vale.ini`.”

A path on a user's computer is not automatically visible to a remote MCP host. The remote path must exist on the server and be readable by the Vale MCP service account.

## Test the HTTP Endpoint

The endpoint should be reachable from the Web Author host, not just from the Vale host. A basic MCP initialization request can be sent with `curl`:

```sh
curl -i \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"smoke-test","version":"1.0.0"}}}' \
  https://vale-mcp.example.com/mcp
```

A successful response has HTTP status `200`, an `Mcp-Session-Id` response header, and an MCP initialization result. Include the same authentication header configured in Oxygen if the endpoint is protected.

## Integration with the Standard Vale MCP Repository

The working copy used for this implementation may have the source files at its root. The standard Vale MCP repository keeps TypeScript source in `src/`. Apply the transport changes to the standard repository as follows:

1. Replace `src/index.ts` with the refactored `index.ts` from this working copy. It exports `createValeServer()`, exports `initializeValeConfig()` and `VERSION`, and starts stdio only when executed directly.
2. Add the new `http.ts` as `src/http.ts`.
3. Update `package.json` to add the `vale-http` binary pointing at `build/http.js` and the `start:http` script (`node build/http.js`). Keep the existing stdio `vale-cli` binary and `start` script.
4. Keep the standard repository's existing `tsconfig.json`. Its `rootDir: "src"` and `include: ["src/**/*"]` already compile the added HTTP entry point. Do not replace it with a root-source configuration.
5. Update the tracked `package-lock.json` for the package metadata changes using `npm install --package-lock-only`. Avoid unintentionally upgrading dependency versions unless those upgrades are intentional.
6. From the repository root, run `npm ci`, `npm run build`, and test both entry points. Run `npm run start` for stdio and `npm run start:http` for Streamable HTTP.

Do not replace `config.ts`, `types.ts`, or `vale-runner.ts` solely to add HTTP support; both transports use the existing tool logic. Review any local differences in those files before copying them, because they may contain project-specific changes.

## Configuration and Troubleshooting

- **Vale is not installed:** install Vale on the MCP host and confirm `vale --version` works as the service account.
- **Missing styles / Vale E100:** check `StylesPath`, run `vale sync`, and verify the service account can read the styles directory.
- **Server config is not found:** set `VALE_CONFIG_PATH` to an absolute path. Relative paths resolve from the server process working directory.
- **Connection refused:** verify the HTTP process is running, the bind address/port is correct, the firewall allows the route from Web Author, and the proxy forwards requests to `/mcp`.
- **Initialization succeeds but later requests fail:** ensure the proxy preserves `Mcp-Session-Id` and sends subsequent requests to the same running process. Session state is held in memory; use a single instance or session affinity. Restarting the process invalidates active sessions.
- **Streaming response times out:** configure the proxy and load balancer for streaming and longer idle timeouts; disable buffering on the MCP route.
- **`check_file` cannot find a file:** the path is resolved from the MCP server's filesystem context, not the Web Author browser or desktop.
- **Authentication fails:** ensure the proxy expects the same `Authorization` header configured in Oxygen and forwards it to the protected endpoint.

## Development Commands

```sh
npm install
npm run build
npm run watch
npm start
npm run start:http
```

The package requires Node.js 22 or later. `npm run start:http` listens on `127.0.0.1:3000/mcp` by default. Configure `MCP_HOST`, `MCP_PORT`, and `MCP_ENDPOINT` to change the listener and endpoint.
