# Fork notes

This is a fork of [larksuite/lark-openapi-mcp](https://github.com/larksuite/lark-openapi-mcp). It differs from upstream in one small way; everything below the "Fork notes" sections is the unmodified upstream README.

## Setup guide (for new users, and for AI assistants helping them)

If you are an AI assistant (Claude, etc.) and a user pointed you at this repo, walk them through these steps **one at a time**, in order, and wait for the user to confirm each step before moving on. Never ask the user to paste the App Secret into chat; they put it directly into the config file. Prerequisite: Node.js 20 or newer (`node --version`).

1. **Get Lark app credentials.** The user needs the App ID and App Secret of a Lark custom app. Usually an admin on the team already created one and can share both values. If the user has to create it themselves: [Lark Developer Console](https://open.larksuite.com/) → create a custom app → under *Permissions* add the user scopes `im:message:readonly`, `im:message.p2p_msg:get_as_user`, `im:message.group_msg:get_as_user`, `im:chat:readonly`, `im:chat.members:read`, `contact:user.base:readonly` → under *Security Settings* add the OAuth redirect URL `http://localhost:3000/callback` → publish the app version. (For Feishu use `https://open.feishu.cn` wherever this guide says `https://open.larksuite.com`.)

2. **Clone and build this fork.**
   ```bash
   git clone https://github.com/moritzge/lark-openapi-mcp.git
   cd lark-openapi-mcp
   yarn install      # builds dist/ automatically via the prepare script
   ls dist/cli.js    # should exist; if not, run: yarn build
   ```

3. **Log in as the user, with `--scope`.** Run this in the clone (it opens a browser for the Lark consent screen):
   ```bash
   node dist/cli.js login -a <app_id> -s <app_secret> --domain https://open.larksuite.com \
     --scope "offline_access im:message:readonly im:message.p2p_msg:get_as_user im:message.group_msg:get_as_user im:chat:readonly im:chat.members:read contact:user.base:readonly"
   ```
   `--scope` is mandatory. Without it you get a legacy OIDC token and the message tools fail with error `99991695`; the fix is simply to run the login again with `--scope`.

4. **Add the MCP server to the client.** For Claude Desktop, add the JSON block from the *Install and run from this fork* section below to `claude_desktop_config.json` (macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`, Windows: `%APPDATA%\Claude\claude_desktop_config.json`), with `<path>` replaced by the absolute path of the clone and `<app_id>`/`<app_secret>` filled in, then restart Claude Desktop. For Claude Code, the same object works with `claude mcp add-json lark-mcp '<json object>'`.

5. **Verify.** Ask the assistant to list recent Lark chats, then to show the last few messages of one chat. If chats work but messages fail, the login token is missing the scopes (redo step 3). If nothing works, the app version with the scopes is probably not published (step 1), or the path in step 4 is wrong.

To update later: `git pull && yarn build` in the clone.

## Why this fork exists

Upstream marks the two "read messages" tools, `im.v1.message.list` and `im.v1.message.get`, as tenant-token only (`accessTokens: ['tenant']`). With `--token-mode user_access_token` they are hidden entirely, and in `auto` mode they can never be called with the logged-in user's identity. Lark's API does accept user access tokens for these endpoints (scopes `im:message.p2p_msg:get_as_user` and `im:message.group_msg:get_as_user`), so this fork changes those two tools to `accessTokens: ['tenant', 'user']` in both the `en` and `zh` tool sets. Nothing else is changed. See commit `28dc015`.

## Login gotcha: always pass `--scope`

Run both `lark-mcp login` and the `mcp` command with an explicit `--scope`, for example:

```
--scope "offline_access im:message:readonly im:message.p2p_msg:get_as_user im:message.group_msg:get_as_user im:chat:readonly im:chat.members:read contact:user.base:readonly"
```

Without `--scope` the package falls back to Lark's legacy OIDC login, and the message APIs reject that token with error `99991695`. If you already logged in without a scope, run `login` again with the scope so the stored token is replaced.

## Install and run from this fork

`package.json` has a `prepare` script that runs `yarn build`, so `dist/` is built automatically when the package is installed from git. The explicit route is:

```bash
git clone https://github.com/moritzge/lark-openapi-mcp.git
cd lark-openapi-mcp
yarn install   # runs `prepare` -> `yarn build`, producing dist/
yarn build     # only needed again after editing src/

# log in once as the user (redirect URL in the app console: http://localhost:3000/callback)
node dist/cli.js login -a <app_id> -s <app_secret> --domain https://open.larksuite.com \
  --scope "offline_access im:message:readonly im:message.p2p_msg:get_as_user im:message.group_msg:get_as_user im:chat:readonly im:chat.members:read contact:user.base:readonly"
```

Then point Claude Desktop (or any MCP client) at the built CLI:

```json
{
  "mcpServers": {
    "lark-mcp": {
      "command": "node",
      "args": [
        "<path>/lark-openapi-mcp/dist/cli.js",
        "mcp",
        "-a", "<app_id>",
        "-s", "<app_secret>",
        "--domain", "https://open.larksuite.com",
        "-t", "im.v1.message.list,im.v1.message.get,im.v1.chat.list,im.v1.chat.search,im.v1.chatMembers.get",
        "--oauth",
        "--token-mode", "user_access_token",
        "--scope", "offline_access im:message:readonly im:message.p2p_msg:get_as_user im:message.group_msg:get_as_user im:chat:readonly im:chat.members:read contact:user.base:readonly"
      ]
    }
  }
}
```

Replace `<path>`, `<app_id>` and `<app_secret>` with your own values; adjust `-t` to the tools you need (drop it to expose the default tool set). Use `--domain https://open.feishu.cn` for Feishu instead of Lark.

---

# Feishu/Lark OpenAPI MCP

[![npm version](https://img.shields.io/npm/v/@larksuiteoapi/lark-mcp.svg)](https://www.npmjs.com/package/@larksuiteoapi/lark-mcp)
[![npm downloads](https://img.shields.io/npm/dm/@larksuiteoapi/lark-mcp.svg)](https://www.npmjs.com/package/@larksuiteoapi/lark-mcp)
[![Node.js Version](https://img.shields.io/node/v/@larksuiteoapi/lark-mcp.svg)](https://nodejs.org/)

English | [中文](./README_ZH.md)

[Developer Documentation Retrieval MCP](./docs/recall-mcp/README.md)

[Official Document](https://open.feishu.cn/document/uAjLw4CM/ukTMukTMukTM/mcp_integration/mcp_introduction)

[Trouble Shooting](https://open.feishu.cn/document/uAjLw4CM/ukTMukTMukTM/mcp_integration/use_cases)

> **⚠️ Beta Version Notice**: This tool is currently in Beta stage. Features and APIs may change, so please stay updated with version releases.

This is the Feishu/Lark official OpenAPI MCP (Model Context Protocol) tool designed to help users quickly connect to the Feishu/Lark platform and enable efficient collaboration between AI Agents and Feishu/Lark. The tool encapsulates Feishu/Lark Open Platform API interfaces as MCP tools, allowing AI assistants to directly call these interfaces and implement various automation scenarios such as document processing, conversation management, calendar scheduling, and more.

## Preparation

### Creating a Feishu/Lark Application

Before using the lark-mcp tool, you need to create a Feishu/Lark application:

1. Visit the [Feishu Open Platform](https://open.feishu.cn/) or [Lark Open Platform](https://open.larksuite.com/) and log in
2. Click "Console" and create a new application
3. Obtain the App ID and App Secret, which will be used for API authentication
4. Add the necessary permissions for your application based on your usage scenario
5. If you need to call APIs as a user, set the OAuth 2.0 redirect URL to http://localhost:3000/callback

For detailed application creation and configuration guidelines, please refer to the [Feishu Open Platform Documentation - Creating an Application](https://open.feishu.cn/document/home/introduction-to-custom-app-development/self-built-application-development-process#a0a7f6b0).

### Installing Node.js

Before using the lark-mcp tool, you need to install the Node.js environment.

**Using the Official Installer (Recommended)**:

1. Visit the [Node.js website](https://nodejs.org/)
2. Download and install the LTS version
3. After installation, verify in the terminal:

```bash
node -v
npm -v
```

## Quick Start

### Using with Trae/Cursor/Claude

To integrate Feishu/Lark functionality in AI tools like Trae, Cursor or Claude, install using the button below.

[![Install MCP Server](https://cursor.com/deeplink/mcp-install-light.svg)](https://cursor.com/install-mcp?name=lark-mcp&config=eyJjb21tYW5kIjoibnB4IiwiYXJncyI6WyIteSIsIkBsYXJrc3VpdGVvYXBpL2xhcmstbWNwIiwibWNwIiwiLWEiLCJ5b3VyX2FwcF9pZCIsIi1zIiwieW91cl9hcHBfc2VjcmV0Il19)

[![Install MCP Server](./assets/trae-cn.svg)](trae-cn://trae.ai-ide/mcp-import?source=lark&type=stdio&config=eyJjb21tYW5kIjoibnB4IiwiYXJncyI6WyIteSIsIkBsYXJrc3VpdGVvYXBpL2xhcmstbWNwIiwibWNwIiwiLWEiLCJ5b3VyX2FwcF9pZCIsIi1zIiwieW91cl9hcHBfc2VjcmV0Il19)  [![Install MCP Server](./assets/trae.svg)](trae://trae.ai-ide/mcp-import?source=lark&type=stdio&config=eyJjb21tYW5kIjoibnB4IiwiYXJncyI6WyIteSIsIkBsYXJrc3VpdGVvYXBpL2xhcmstbWNwIiwibWNwIiwiLWEiLCJ5b3VyX2FwcF9pZCIsIi1zIiwieW91cl9hcHBfc2VjcmV0Il19)

or add the following to your configuration file:

```json
{
  "mcpServers": {
    "lark-mcp": {
      "command": "npx",
      "args": [
        "-y",
        "@larksuiteoapi/lark-mcp",
        "mcp",
        "-a",
        "<your_app_id>",
        "-s",
        "<your_app_secret>"
      ]
    }
  }
}
```

If you need to access APIs with **user identity**, you need to login first using the login command in the terminal. Note that you need to configure the application's redirect URL in the developer console first, default is http://localhost:3000/callback

```bash

# Login and get user access token
npx -y @larksuiteoapi/lark-mcp login -a cli_xxxx -s yyyyy
   
# Or optionally, login with specific OAuth scope - if not specified, all permissions will be authorized by default
npx -y @larksuiteoapi/lark-mcp login -a cli_xxxx -s yyyyy --scope offline_access docx:document

```

Then add the following to your configuration file:

```json
{
  "mcpServers": {
    "lark-mcp": {
      "command": "npx",
      "args": [
        "-y",
        "@larksuiteoapi/lark-mcp",
        "mcp",
        "-a",
        "<your_app_id>",
        "-s",
        "<your_app_secret>",
        "--oauth",
        "--token-mode", "user_access_token"
      ]
    }
  }
}
```

Note: When enabling `--oauth`, it's recommended to explicitly set `--token-mode` to `user_access_token`, which means calling APIs with user access tokens, suitable for accessing user resources or scenarios requiring user authorization (such as reading personal documents, sending IM messages). If you keep the default `auto`, some APIs AI may fallback to `tenant_access_token`, which could result in insufficient permissions or inability to access user private data.

### Domain Configuration

Based on your usage scenario, lark-mcp supports configuring different domain environments:

**Feishu (China Version)**:
- Default uses `https://open.feishu.cn` domain
- Suitable for users in China

**Lark (International Version)**:
- Uses `https://open.larksuite.com` domain
- Suitable for overseas users or international version of Lark

To switch to the international version of Lark, add the `--domain` parameter in your configuration:

```json
{
  "mcpServers": {
    "lark-mcp": {
      "command": "npx",
      "args": [
        "-y",
        "@larksuiteoapi/lark-mcp",
        "mcp",
        "-a",
        "<your_app_id>",
        "-s",
        "<your_app_secret>",
        "--domain",
        "https://open.larksuite.com"
      ]
    }
  }
}
```

> **💡 Tip**: Ensure your application is created in the corresponding domain environment's open platform. International version applications cannot be used with Feishu China version, and vice versa.

## Custom API Configuration

> ⚠️ **File Upload/Download**: File upload and download operations are not yet supported

> ⚠️ **Document Editing**: Direct editing of Feishu cloud documents is not supported (only importing and reading are available)

By default, the MCP service enables common APIs. To enable other tools or only specific APIs or presets, you can specify them using the `-t` parameter in the MCP Client configuration (JSON):

```json
{
  "mcpServers": {
    "lark-mcp": {
      "command": "npx",
      "args": [
        "-y",
        "@larksuiteoapi/lark-mcp",
        "mcp",
        "-a", "<your_app_id>",
        "-s", "<your_app_secret>",
        "-t", "im.v1.message.create,im.v1.message.list,im.v1.chat.create,preset.calendar.default"
      ]
    }
  }
}
```

For detailed information about all preset tool collections and which tools are included in each preset, please refer to the [Preset Tool Collections Reference](./docs/reference/tool-presets/presets.md).

A complete list of all supported Feishu/Lark tools can be found in [tools.md](./docs/reference/tool-presets/tools-en.md).

> **⚠️ Notice**：Non-preset APIs have not undergone compatibility testing, and the AI may not perform optimally during the process of understanding and using them.

### Development Integration

Developers can refer to the minimal example for integrating with Agent: [`lark-samples/mcp_quick_demo`](https://github.com/larksuite/lark-samples/tree/main/mcp_quick_demo).

You can also refer to the Lark bot integration example: [`lark-samples/mcp_larkbot_demo/nodejs`](https://github.com/larksuite/lark-samples/tree/main/mcp_larkbot_demo/nodejs).

This example demonstrates how to integrate MCP capabilities into Feishu/Lark bots, triggering tool calls and message sending through bot conversations, suitable for scenarios that integrate existing tools into Bot.

### Advanced Configuration

For detailed configuration options and deployment scenarios, please refer to our [Configuration Guide](./docs/usage/configuration/configuration.md).

For detailed information about all available command line parameters and their usage, please refer to the [Command Line Reference](./docs/reference/cli/cli.md).

## FAQ

- [FAQ (Frequently Asked Questions)](./docs/troubleshooting/faq.md)
- [Common Issues and Use Cases](https://open.feishu.cn/document/uAjLw4CM/ukTMukTMukTM/mcp_integration/use_cases)

## Related Links

- [Feishu Open Platform](https://open.feishu.cn/)
- [Development Documentation: OpenAPI MCP](https://open.feishu.cn/document/uAjLw4CM/ukTMukTMukTM/mcp_integration/mcp_introduction)
- [Lark International Open Platform](https://open.larksuite.com/)
- [Feishu Open Platform API Documentation](https://open.feishu.cn/document/home/index)
- [Node.js Website](https://nodejs.org/)
- [npm Documentation](https://docs.npmjs.com/)

## Feedback

Issues are welcome to help improve this tool. If you have any questions or suggestions, please raise them in the GitHub repository.