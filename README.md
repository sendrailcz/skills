# Sendrail skills

Agent skills for [Sendrail](https://sendrail.cz), an SMS API with its own network of GSM gateways: transactional SMS, verification codes (OTP), number check, opt-outs, webhooks.

| Skill | What it teaches an agent |
|---|---|
| [`sendrail`](sendrail/SKILL.md) | Sending SMS and verification codes through the Sendrail API, and writing an integration correctly: idempotency, signed webhooks, test keys, bulk rules, error handling. |

## Install

Claude Code, for yourself:

```
git clone https://github.com/sendrailcz/skills.git /tmp/sendrail-skills
mkdir -p ~/.claude/skills && cp -r /tmp/sendrail-skills/sendrail ~/.claude/skills/
```

For one project, copy the `sendrail` directory into `.claude/skills/` of the repository instead. Other agents that read `SKILL.md` files work the same way; for anything else, point the model at <https://sendrail.cz/llms.txt>.

## What you need

- A Sendrail account and an API key: <https://sendrail.cz/app>. Start with a **test key** (`sr_test_…`): it behaves like a live one, sends nothing and costs nothing.
- The key in the environment as `SENDRAIL_API_KEY`.

For a key you hand to an agent, set limits on it in the dashboard (messages and spend per 24 hours, allowed recipients).

## Related

- MCP server: `https://api.sendrail.cz/mcp` — <https://sendrail.cz/ai>
- Documentation: <https://sendrail.cz/docs>, as one Markdown file <https://sendrail.cz/docs.md>
- n8n node: <https://github.com/sendrailcz/n8n-nodes-sendrail>

## License

MIT
