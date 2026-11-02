# Contributing

This doc covers how to change the wave-mcp skill. It says what the skill covers, how to check and try a change, and how a change reaches users.

## What the skill covers

The skill tells an AI assistant how to use the MCP tools of a WAVE server. It names the tools, the exact values they take and the mistakes to avoid.

The skill does not describe the recipe format, the Blocks or their settings. The assistant reads those from the WAVE server, so they match the version of that server.

Change the skill when WAVE adds, removes, renames or changes an MCP tool. Also change it when a value or a mistake that it lists is no longer right. A new Block, a new Block setting or a change to the recipe format needs no change here.

## Where each part is

| File | What it holds |
|---|---|
| `plugins/wave-mcp/skills/wave-mcp/SKILL.md` | The skill. |
| `plugins/wave-mcp/.claude-plugin/plugin.json` | The name, description and links of the plugin. |
| `.claude-plugin/marketplace.json` | The plugins that this repo offers to Claude Code. |

## Check a change

You need Claude Code. Run this command at the root of your clone:

```sh
claude plugin validate .
```

The output ends with `Validation passed with warnings`. It shows one warning, `No version specified`. That warning is expected, as [How a change reaches users](#how-a-change-reaches-users) says.

## Try a change

You need a WAVE server with its MCP server turned on. The skill says how to connect Claude Code to it.

Run this command at the root of your clone:

```sh
claude --plugin-dir plugins/wave-mcp
```

Claude Code starts with the skill from your clone. For this session only, the skill from your clone takes the place of the wave-mcp plugin that you installed. Ask the assistant for a task that your change covers, such as writing a test and running it. Check that the assistant follows the changed skill.

## How a change reaches users

This repo has no releases, tags or version numbers. A change is released as soon as it is on `master`.

The plugin has no version number on purpose. Without one, Claude Code takes each commit on `master` as a new version. Do not add a version to `plugin.json`. With one, users get a change only when the number goes up.

Users get the change when Claude Code updates the plugin, as [Updates](README.md#updates) in the README says.

## License

This repo is under the MIT License. [LICENSE](LICENSE) holds its text.

Contributions are licensed under the same MIT License. No sign-off line or contributor agreement is needed.
