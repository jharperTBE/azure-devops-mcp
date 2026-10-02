# Azure DevOps Server (On-Premises): Read-Only Pull Request Review

The local MCP server targets Azure DevOps Services by default. Passing `--server-url` switches it to a separate, read-only mode for **Azure DevOps Server** (2019 and later). This mode is for reviewing pull requests. It does not expose the cloud toolset.

> [!NOTE]
> In this mode the server never calls cloud-only endpoints such as tenant discovery or Microsoft Entra sign-in. It loads only the tools listed below. Write operations are not available. Cloud behavior does not change when you omit `--server-url`.

## Install from this fork on Windows

These instructions use [jharperTBE/azure-devops-mcp](https://github.com/jharperTBE/azure-devops-mcp), **not** the `@azure-devops/mcp` npm package. That package does not include this fork's on-premises changes. Run commands in PowerShell. You need network access to your Azure DevOps Server (including VPN, if required), a Windows account with access to the desired project collection, and a GitHub account entitled to use Copilot CLI.

1. Install [Git for Windows](https://git-scm.com/download/win), [Node.js 22 or later](https://nodejs.org/en/download), and Windows PowerShell (`powershell.exe`, included with Windows). Open a **new** PowerShell window and check:

   ```powershell
   git --version
   node --version
   npm --version
   Get-Command powershell.exe
   ```

   If a command is not found, reopen the terminal after installation and confirm the installer's PATH setting. Node.js 22+ also meets the [Copilot CLI installation requirement](https://docs.github.com/en/copilot/get-started/cli-quickstart).

2. Clone and build this fork. Choose a folder you can keep; the Copilot configuration will point to its built files:

   ```powershell
   New-Item -ItemType Directory -Force "$env:USERPROFILE\source\repos" | Out-Null
   Set-Location "$env:USERPROFILE\source\repos"
   git clone https://github.com/jharperTBE/azure-devops-mcp.git
   Set-Location .\azure-devops-mcp
   npm ci
   npm run build
   (Resolve-Path .\dist\index.js).Path
   ```

   The final command prints the absolute path to use in the configuration. Keep the clone and its `node_modules` directory; the built server needs its installed dependencies at runtime. Do not use `npx @azure-devops/mcp` to launch this fork.

3. Install [GitHub Copilot CLI](https://docs.github.com/en/copilot/get-started/cli-quickstart):

   ```powershell
   npm install -g @github/copilot
   copilot
   ```

   In Copilot CLI, run `/login` and follow the GitHub sign-in prompts. This signs you in to **GitHub**, not Azure DevOps. Exit the CLI before editing its MCP configuration.

4. Edit `%USERPROFILE%\.copilot\mcp-config.json` (in PowerShell, `$env:USERPROFILE\.copilot\mcp-config.json`). Create the `.copilot` folder and file if needed. Add this entry under `mcpServers`, replacing `YOUR_USERNAME`, `COLLECTION_NAME`, and `https://YOUR_SERVER_ROOT`:

   ```json
   {
     "mcpServers": {
       "ado-onprem": {
         "type": "local",
         "command": "node",
         "args": ["C:\\Users\\YOUR_USERNAME\\source\\repos\\azure-devops-mcp\\dist\\index.js", "COLLECTION_NAME", "--server-url", "https://YOUR_SERVER_ROOT", "-a", "windows"],
         "tools": ["*"]
       }
     }
   }
   ```

   Use the exact path printed in step 2 and escape each `\` as `\\` in JSON. The server root **excludes** the collection (for example, `https://ado.example.com` with `DefaultCollection`, or `https://ado.example.com/tfs` with `DefaultCollection`). The resulting collection URL is `<server-root>/<collection>`. If `mcp-config.json` already exists, **add only the `ado-onprem` entry to its existing `mcpServers` object**; do not replace other servers. Use a different entry name for each collection you want to connect to. Do not put passwords or tokens in this file.

5. Start Copilot CLI again (`copilot`), run `/mcp`, and confirm `ado-onprem` is connected and exposes five `onprem_*` tools. Try: `Use ado-onprem to list projects in this collection.` Then, with a project and repository name, try: `Use ado-onprem to list active pull requests in project PROJECT and repository REPOSITORY.` For a known PR: `Review pull request 123 using ado-onprem. Read its metadata, changed files and diffs, linked work items, and discussion threads; summarize outstanding review concerns. Do not modify anything.`

The Windows transport runs as **the account that launches Copilot CLI** and uses Windows integrated authentication (Negotiate/NTLM); no Azure DevOps password is requested or saved. If the collection name is unknown, open `<server-root>/_apis/projectCollections?api-version=5.0` in a browser while connected to the same network and signed in to Windows. Ask your Azure DevOps administrator if you cannot access that endpoint or the collection.

### Update this installation

When this fork changes, stop Copilot CLI, then run these commands from the clone and restart the CLI:

```powershell
Set-Location "$env:USERPROFILE\source\repos\azure-devops-mcp"
git pull --ff-only
npm ci
npm run build
```

If you cloned to a different location, use that path instead. `npm ci` uses the lockfile to install the versioned dependencies.

### Troubleshooting

| Symptom                            | Check                                                                                                                                                                   |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ado-onprem` is absent from `/mcp` | Confirm the JSON syntax, the `mcpServers` key, and the absolute path to `dist\index.js`. Restart Copilot CLI after editing the file.                                    |
| `node` or `git` is not recognized  | Install the prerequisite and open a new PowerShell window so PATH is refreshed.                                                                                         |
| Connection fails or returns 404    | Ensure VPN/network access, the exact server root (which may include `/tfs`), and the collection name. Do not include the collection twice.                              |
| 401/403 or no projects             | Run Copilot CLI under a Windows account authorized for that collection. Verify access in a browser; check with the server administrator if integrated auth is disabled. |
| PowerShell worker fails            | Confirm `powershell.exe` is available. PowerShell Constrained Language Mode and application-control policies can block the worker; contact your administrator.          |
| Only cloud tools appear            | Launch this fork's built `dist\index.js`, not the published npm package, and pass `--server-url`.                                                                       |

## Usage

```bash
node dist/index.js <collection> --server-url <server-root> [-a windows|pat] [--api-version 5.0] [-d repositories]
```

| Argument               | Description                                                                                                                                                                 |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `<collection>`         | Project collection name (for example `DefaultCollection`). In on-prem mode this positional argument replaces the organization name.                                         |
| `--server-url`         | Server root without the collection, for example `https://ado.contoso.com` or `https://ado.contoso.com/tfs`. Do not put credentials, query strings, or fragments in the URL. |
| `-a, --authentication` | `windows` (default in this mode) or `pat`. No other authentication types are accepted with `--server-url`.                                                                  |
| `--api-version`        | REST API version sent with every request. Defaults to `5.0` (Azure DevOps Server 2019). Newer servers accept higher versions such as `7.0`.                                 |
| `-d, --domains`        | Must include `repositories` (the default `all` does). Other domains are ignored and a warning is logged.                                                                    |

The collection URL is `<server-root>/<collection>`.

## Authentication

### Windows integrated authentication (`windows`, default)

Requests are sent from a persistent, hidden Windows PowerShell worker. The worker uses .NET `HttpClient` with `UseDefaultCredentials`. NTLM or Negotiate (Kerberos) runs through Windows SSPI with the identity of the signed-in user. Neither Node.js nor the MCP configuration ever holds a password or token.

- Windows only. `powershell.exe` must be on `PATH`.
- PowerShell Constrained Language Mode or application control policies can block the worker. If that happens, the worker's error message is returned in the tool result.
- Redirects are not followed. Requests are restricted to the configured collection URL, so integrated credentials are never sent to another host.

### Personal access token (`pat`)

Set `PERSONAL_ACCESS_TOKEN` in the environment that starts the MCP client to the base64 encoding of `<username>:<pat>`, the same format used in cloud `pat` mode. The token is sent with HTTP Basic authentication. The server must have PAT authentication enabled. Do not put the token in the MCP configuration file.

## Tools

All tools are read-only. Their output is wrapped as untrusted content, as in cloud mode.

| Tool                         | Actions                                              | Purpose                                                                                                                                                                             |
| ---------------------------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onprem_project_list`        | —                                                    | List projects in the collection.                                                                                                                                                    |
| `onprem_repository`          | `list`, `get`                                        | List or get Git repositories in a project.                                                                                                                                          |
| `onprem_pull_request`        | `list`, `get`, `commits`, `iterations`, `work_items` | Search pull requests, read metadata (reviewers, votes, branches, merge status), commits, iterations (pushes), and linked work items.                                                |
| `onprem_pull_request_change` | `list`, `diff`, `content`                            | List changed files for an iteration. Produce a unified diff of one file against the merge base. Read file content from the source, base, or target side, or from a specific commit. |
| `onprem_pull_request_thread` | —                                                    | Read discussion threads, including file and line anchors. System threads and deleted comments are excluded by default.                                                              |

Diffs are computed locally from the two file versions, because Azure DevOps Server does not provide a text diff REST API. Changes use the latest iteration by default. Pass `iterationId` and `compareTo` to review a single push.

## GitHub Copilot CLI

Follow [Install from this fork on Windows](#install-from-this-fork-on-windows) for the complete download, build, authentication, and Copilot CLI setup. The `mcp-config.json` example above uses Windows authentication without storing credentials.

## Limitations

- Pull request review only. Work item, pipeline, wiki, search, and test plan tools are not available in this mode.
- Windows integrated authentication requires Windows. Use `pat` on other platforms.
- Very large files can produce a non-minimal diff. When that happens, the changed region is shown as one replacement and the output includes a `note`.
