# artifacts

**Agent Skills** that turn your agent's answer into a page at a link that
never changes. Ask for the migration plan, the Q3 report, or a sign-up sheet
for Friday, and you get a link to send. When the plan changes, the agent
updates the page at the same link. When people comment on it, the agent
answers them.

## What you can ask for

> Write up the migration plan as an Artifact and share it with ana@acme.com as
> an editor.

Your agent writes the document, publishes it, shares it with Ana, and hands
you the link. Ana can open it, comment on it, and publish changes of her own.

> Make a dashboard from these numbers.

A page with the charts on it, private to you until you say who else can open
it.

> Make a sign-up sheet for Friday that remembers who signed up.

The page keeps the names, and everyone who opens it sees them update live.

> Update the plan with the new cutover date.

A new version at the same link. The link you already sent keeps working, and
the earlier versions stay in the history.

> What did people say about the plan? Fix what you can.

Your agent reads the comments that mention it, makes the changes, republishes,
replies in each thread with what changed, and marks it resolved.

## The skills

| Skill               | What it does                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------------- |
| `artifacts`         | Publishes a Markdown document or an HTML page, updates it at the same link, shares it, deletes it |
| `artifact-comments` | Finds the comment threads that mention the agent, answers them, and resolves them                 |

## Install

```sh
npx skills add bisque-cloud/artifacts
```

That copies both skills into your agent's skills directory —
`~/.claude/skills/` for Claude Code, `~/.agents/skills/` for Codex and Gemini.

### Manual install

```sh
git clone https://github.com/bisque-cloud/artifacts
cp -r artifacts/skills/artifacts ~/.claude/skills/artifacts                  # Claude Code
cp -r artifacts/skills/artifact-comments ~/.claude/skills/artifact-comments
cp -r artifacts/skills/artifacts ~/.agents/skills/artifacts                  # Codex + Gemini
cp -r artifacts/skills/artifact-comments ~/.agents/skills/artifact-comments
```

Invoke with `/artifacts` (Claude Code), `$artifacts` (Codex), or let it
activate on description match.

## Requirements

- **A Bisque account.** The skills read `$BISQUE_API_KEY`, then the profiles
  in `~/.bisque/config.json` — the same credentials the
  [`present`](https://github.com/bisque-cloud/presenter) skill uses, so if you
  have published a presentation from this machine you are already signed in.
  A client that connects to the server by URL can sign in through the browser
  instead.
- **Network access to `bisque.today`.** Everything runs through the Bisque MCP
  server at `https://bisque.today/mcp`; nothing is stored locally.

## Who can open it

Every Artifact starts private to you. Name the people, your bisque.team
company, or "anyone with the link" when you ask, or later. Making a page open
to anyone with the link deletes its comment threads, so the agent asks before
it does that.

## Presentations

For slides that play with a voice, install
[`presenter`](https://github.com/bisque-cloud/presenter).

## License

[MIT](./LICENSE)
