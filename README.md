# Nightwatch Cursor Plugin

Official Laravel Nightwatch integration plugin for Cursor. Monitor your Laravel applications, investigate issues, view traces, and configure data collection rules directly from your editor.

## Installation

### From Cursor Marketplace

Search for "Nightwatch" in the Cursor plugin marketplace and install.

Restart Cursor to activate the plugin, then verify:

```
/nightwatch      # Should show the help message
/mcp             # Should show nightwatch MCP server
```

## Slash Commands

### `/nightwatch <query>`

Ask questions about your Nightwatch environment in natural language.

```bash
/nightwatch Fix issue 123
/nightwatch What are the top issues in the last 24 hours?
/nightwatch Show me all unresolved issues in my production environment
/nightwatch Which exceptions are affecting the most users?
/nightwatch Show me the full trace for issue 1234
/nightwatch Show issues related to our Stripe integration
/nightwatch Show me routes that are exceeding thresholds
```

## Skills

### nightwatch-configure

Configure Nightwatch data collection, sampling rates, filtering rules, and redaction policies.

Use this skill when:

- Setting up Nightwatch for the first time
- Reducing data volume or event quota usage
- Protecting sensitive data (PII, credentials, tokens)
- Optimizing monitoring for production workloads

The skill covers:

- **Sampling**: Control which requests/commands are captured
- **Filtering**: Exclude specific events (queries, cache, mail, etc.)
- **Redaction**: Obfuscate sensitive data before storage

## Configuration

The plugin automatically configures the Nightwatch MCP server on install. No additional setup required beyond restarting Cursor.

Authentication is handled via OAuth - complete the browser flow when first using the plugin.

## Plugin Structure

```
nightwatch-for-cursor/
├── .cursor-plugin/
│   └── plugin.json              # Plugin metadata
├── mcp.json                     # MCP server configuration
├── commands/
│   └── nightwatch.md            # /nightwatch command
├── skills/
│   └── nightwatch-configure/    # Configuration skill
│       └── SKILL.md
├── assets/
│   └── logo.svg                 # Plugin logo
├── README.md
└── LICENSE
```

## Features

- **Issue Investigation**: Query exceptions, view stack traces, see affected users, track issue status
- **Issue Resolution**: Fix issues directly through natural language commands
- **Threshold Monitoring**: Identify routes exceeding performance thresholds
- **Data Collection Control**: Configure sampling, filtering, and redaction rules
- **Privacy Protection**: Built-in guidance for PII protection and data minimization

## Documentation

- [Nightwatch Documentation](https://nightwatch.laravel.com/docs)
- [MCP Server Setup](https://nightwatch.laravel.com/docs/mcp-server)
- [Filtering & Configuration](https://nightwatch.laravel.com/docs/filtering)

## License

MIT
