# Contributing

This doc covers how to change the wave-mcp skill. It says what the skill covers, how to check and try a change, and how a change reaches users.

## What the skill covers

The skill tells an AI assistant how to use the MCP tools of a WAVE server. It names the tools, the exact values they take and the mistakes to avoid. For some Blocks, such as SQL, OpenSearch and SFTP, it also lists their operations, settings and profile fields.

The skill tells the assistant to read the recipe format and the settings of each Block from the WAVE server. Those match the version of that server.

Change the skill when a change to WAVE makes something in it wrong: an MCP tool, a value, a Block setting or a behavior that it names. A new Block needs no skill change, because the assistant reads its settings from the server.

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
