<p align="center">
  <img src="public/logo.png" alt="Depfender" width="200" />
</p>

<h1 align="center">Depfender</h1>

<p align="center">
  Detect data exfiltration threats in open-source dependencies before they compromise your systems.
</p>

<p align="center">
  <a href="https://depfender.dev">Website</a> ·
  <a href="#installation">Installation</a> ·
  <a href="https://github.com/depfenderdev/depfender/discussions">Discussions</a>
</p>

<p align="center">
  <a href="https://depfender.dev"><img src="https://img.shields.io/badge/Website-depfender.dev-blue" alt="Website" /></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-green" alt="License" /></a>
</p>

---

## The Problem

**Without Depfender:**

- Supply chain attacks hide in trusted packages
- Manual code review doesn't scale to thousands of dependencies
- Traditional scanners miss sophisticated exfiltration patterns
- You only find out after the breach

## The Solution

**With Depfender:**

- AI-powered analysis detects subtle exfiltration attempts
- Multi-agent consensus reduces false positives
- Scan packages before they enter your codebase
- Get detailed evidence reports for every finding

---

## Installation

Install the MCP server to scan packages directly from your IDE. See the [MCP server repo](https://github.com/depfenderdev/mcp) for full setup instructions.

### Cursor

Add to your Cursor MCP settings:

```json
{
  "mcpServers": {
    "depfender": {
      "command": "npx",
      "args": ["@depfenderdev/mcp"]
    }
  }
}
```

### Claude Code

```bash
claude mcp add depfender -- npx @depfenderdev/mcp
```

### VS Code

Add to your VS Code MCP settings (`.vscode/mcp.json`):

```json
{
  "mcpServers": {
    "depfender": {
      "command": "npx",
      "args": ["@depfenderdev/mcp"]
    }
  }
}
```

---

## Features

- **Multi-Agent Analysis** - Six specialized security agents examine code from different angles
- **IDE Integration** - Works directly in your editor via MCP
- **API Access** - Integrate into CI/CD pipelines
- **Evidence Reports** - Detailed findings with file/line citations

---

## Documentation

Visit [depfender.dev/docs](https://depfender.dev/docs) for full documentation.

---

## Community

- [GitHub Discussions](https://github.com/depfenderdev/depfender/discussions) - Questions and ideas
- [Twitter/X](https://x.com/depfenderdev) - Updates and announcements

---

## License

[MIT](LICENSE)
