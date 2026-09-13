---
name: integration-standards
description: Team standards for touching third-party platforms (Slack, Gmail, and any other connected service). Use whenever a task involves an external service, posting to Slack, drafting email, reading CRM or payment data. Enforces the One CLI path (search, knowledge, execute), env-var connection keys, and the team's outbound rules.
---

# Integration standards

Every agent on this team touches outside platforms the same way. These are not style preferences. Each rule exists because the alternative already failed for somebody.

## The one path

All third-party calls go through the One CLI with the `--agent` flag, which returns structured JSON:

```bash
one --agent <command>
```

Never call a platform API directly with curl. Never install a platform SDK for a one-off call. Never paste a bearer token into a command. If the CLI has no action for what you need, stop and say so instead of improvising around it.

## Read before you execute

Three steps, in order, no skipping:

```bash
# 1. Find the action
one --agent actions search <platform> "<what you want to do>"

# 2. Read its documentation. REQUIRED before any execute.
one --agent actions knowledge <platform> <actionId>

# 3. Execute, with the connection key from an env var
one --agent actions execute <platform> <actionId> "$PLATFORM_KEY" [flags]
```

Do not execute an action whose knowledge you have not read in this session. The knowledge tells you the required parameters and which flag each one belongs to: path variables go in `--path-vars`, query parameters in `--query-params`, the request body in `-d`. Mixing those up is the most common cause of 403s.

Search ranks by relevance, not by safety. Asking Gmail for "draft" surfaces the send action right next to the create action. Read the title and path before you pick an actionId.

## Connection keys

- Keys come from environment variables, one alias per connection (`$SLACK_DEFAULT_KEY`, `$GMAIL_OPS_KEY`). Set them once in the shell profile; find your keys with `one --agent list`.
- The key is the positional third argument of `actions execute`. The shell expands the alias at run time.
- Never print a raw key, never commit one, and never write one into a skill or doc. If you need to confirm an alias is set, check its length, not its value: `echo ${#SLACK_DEFAULT_KEY}`.

## Slack

- Agents post to the team's ops channel only, never to customer-facing channels.
- Always include `text`, even when sending Block Kit `blocks`. It is the fallback for notifications and screen readers.
- Slack mrkdwn is not Markdown: bold is `*bold*` not `**bold**`, links are `<url|label>` not `[label](url)`. Headings and tables do not exist; use Block Kit `header` and `section` blocks.
- Slack returns HTTP 200 with `"ok": false` on failure. Check `ok` in the response, and record `ts` if a follow-up might need to thread.

## Email (Gmail)

- Agents draft, humans send. Use the create-draft action. Do not use `drafts/send` or `messages/send` unless a human has approved that specific message.
- Every Gmail action takes `--path-vars '{"userId":"me"}'`. Do not put `userId` in the body.
- Verify the draft landed: the response carries an `id` and `DRAFT` in `message.labelIds`.

## After any call

Read the JSON response and confirm the operation actually happened: `ok: true`, a returned `id`, the expected label. A zero exit code is not success. Put the returned id or timestamp in your task notes so the next agent can find what you created.
