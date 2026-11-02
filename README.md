# WAVE skills

Skills for AI assistants that use WAVE, the Workflow Automation and Validation Engine. The code is in [waveapphq/wave](https://github.com/waveapphq/wave) and the docs are in [waveapphq/wave-docs](https://github.com/waveapphq/wave-docs).

This repo is a Claude Code plugin marketplace. It has one plugin:

- **wave-mcp:** teaches the assistant to work with a WAVE server through its MCP tools. With it, the assistant writes and changes tests as recipes, runs them, finds out why a run failed, and handles secrets and test data.

## Install

In Claude Code, run:

```
/plugin marketplace add waveapphq/wave-skills
/plugin install wave-mcp@wave-skills
```

To use it, you need a WAVE server with its MCP server turned on, and your assistant connected to it. The skill explains how to connect.

## Updates

To get the latest skill, run these commands in a terminal, then restart Claude Code:

```sh
claude plugin marketplace update wave-skills
claude plugin update wave-mcp@wave-skills
```

To get the latest skill each time Claude Code starts, run `/plugin` in Claude Code. In the list of marketplaces, open wave-skills and choose 'Enable auto-update'.

## Versions

The skill tells the assistant to get the details that change between WAVE versions from your WAVE server, such as the recipe format and the settings of each Block.

## Contributing

To change the skill, see [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT. See [LICENSE](LICENSE).
