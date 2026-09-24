# dFlow skills

Agent skills for [dFlow](https://dflow.sh). This is dflow.sh, not the unrelated Solana project of a similar name.

```bash
npx skills add dflow-sh/skills
```

Install one skill:

```bash
npx skills add dflow-sh/skills --skill dflow-deploy
npx skills add dflow-sh/skills --skill dflow-mcp
```

| Skill | Use when |
| --- | --- |
| `dflow-deploy` | Ship a git app, Docker image, or database onto a dFlow worker node |
| `dflow-mcp` | Connect an OAuth MCP client and operate a dFlow workspace |

The MCP server is `https://app.dflow.sh/api/mcp`. Install the Cursor or Grok plugin from [dflow-sh/plugin](https://github.com/dflow-sh/plugin), then approve the OAuth prompt. These skills tell the agent how to use that server. They do not connect it by themselves.

Docs: [dFlow MCP](https://docs.dflow.sh/articles/3167609-dflow-mcp)
