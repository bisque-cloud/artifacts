---
name: artifacts
description: Publishes a page or a document at a link that never changes — a plan, a report, a dashboard, a checklist, a poll — and updates it at the same link, shares it with named people, or makes it public. Use when asked to make an artifact, publish a page, write something up as a doc people can open or comment on, build a dashboard or a form, share a link to a report, update or republish a page, or change who can open it.
---

# Artifacts

An Artifact is a page your agent publishes at a link that stays the same. Every
update is a new version at that link, earlier versions stay in the history,
and the people it is shared with can comment on it. Reach for one when the
answer is a page rather than a message: something a person opens, reads,
returns to, or sends on.

Two kinds come from this skill:

- **`markdown`** — a document read top to bottom: a plan, a report, meeting
  notes. Readers comment on a passage, and the owner can edit it in the
  browser.
- **`html`** — a page: a dashboard, a chart, a form, a checklist, a poll. It
  can keep data everyone who opens it shares.

## 1. Credentials

Every call runs through the Bisque MCP server at `https://bisque.today/mcp`
as the person whose key you hold. If your client already has that server
connected, call the tools by name and skip the `curl`.

```sh
curl -s -X POST https://bisque.today/mcp \
  -H "Authorization: Bearer $BISQUE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call",
       "params":{"name":"list_artifacts","arguments":{}}}'
```

Take the key from `$BISQUE_API_KEY`, otherwise from `~/.bisque/config.json`,
whose profiles look like `profiles.<name>.apiKey`. Pick `$BISQUE_PROFILE`, or
the `profile` in a `.bisque.json` in the working directory; failing that the
profile named `present`, then `default`, then the only usable one. There is
no anonymous path: an Artifact always belongs to an account.

**When several accounts are configured and none is pinned, use the one the
user's Artifacts already live in.** The Artifact is created in that account,
and only that account can update or share it later, so the question is whose
work this is, not how many keys are saved. When the user has consistently
published from one account — the earlier Artifacts in this conversation or
project, the links they share, what `list_artifacts` returns for each key —
publish routine work there without asking. Ask only when ownership is
genuinely unclear: the accounts both hold recent Artifacts and this one could
belong to either, or there is no history to go on. Updating an existing
Artifact always uses the account that owns it.

**No key at all.** Connect the server to the client by URL instead and let
it sign in; the server's first answer starts the sign-in, and the client
holds the token from then on. In Claude Code:

```sh
claude mcp add --transport http bisque https://bisque.today/mcp
```

If the `present` skill is installed, `node present.mjs login` saves a key to
`~/.bisque/config.json` under the profile `present`, and this skill finds it
there.

## 2. Write the whole document

Write the complete page before publishing. Both create and update take the
whole document, never a diff.

**For `markdown`**, send the Markdown source.

**For `html`**, send a complete page. It runs in a sandboxed frame:

- Styles, scripts and images are inline. The only things it may load are
  libraries from `cdnjs.cloudflare.com`, `cdn.jsdelivr.net/npm/`,
  `cdn.tailwindcss.com`, `code.jquery.com` or `unpkg.com`, and typefaces from
  Google Fonts.
- Pin every library to an exact version in its URL (`chart.js@4.4.1`, never
  `@latest`), so the page renders the same for as long as it is published.
- `fetch` reaches nowhere. Put the data in the page.
- Lay it out for a phone as well as a desktop; many links are opened on one.

**A page that remembers what people do on it** — the ticks on a checklist,
the names on a sign-up sheet, the votes in a poll — declares
`capabilities: ["state.read", "state.write"]` and reads and writes JSON
documents through `window.bisque.state` (`get`, `patch`, `list`,
`subscribe`). Changes show up live for everyone. Add `viewer.identity` to
know who is looking (`window.bisque.viewer`).

When some paths need different access, pass `state.rules`. A rule covers its
path and everything below it, and a deeper rule overrides it. A path ending
in `{self}` names each person's own document under that prefix, which only
that person can write. A poll where each person casts one vote and everyone
who can open the page sees the tally:

```json
{
  "rules": [
    { "path": "votes", "read": "viewer", "write": "owner" },
    { "path": "votes/{self}", "write": "viewer" }
  ]
}
```

`read` takes `anyone`, `viewer` (can open it), `contributor` or `owner`;
`write` takes the last three. Without rules, everyone who can open the page
reads, and contributors and the owner write.

## 3. Publish

```json
{
  "name": "create_artifact",
  "arguments": {
    "title": "Q3 revenue by region",
    "type": "html",
    "content": "<!doctype html>…",
    "description": "Revenue by region for Q3, with the EMEA dip flagged.",
    "icon": "📊"
  }
}
```

`type` defaults to `html`. The call waits until the page is live and returns
its `artifact_id` and `url`. Give the user the URL, and keep the
`artifact_id`: every later change needs it.

If `publication_status` comes back `queued` or `processing`, the page is still
being prepared. Call `get_artifact` until `published_version_id` is set
before you call the link live. `failed` carries `publication_error`; report
it, fix the content, and publish again.

An Artifact is private to its owner unless the request says otherwise. When
the user names an audience in the same breath ("…and share it with the
platform team"), pass `audience` and `share_with` on `create_artifact`
rather than making a second call.

## 4. Update it at the same link

```json
{
  "name": "update_artifact",
  "arguments": { "artifact_id": "art_…", "content": "<!doctype html>…" }
}
```

New `content` publishes a new version at the same URL and moves readers to
it. `title`, `description` and `icon` change in place, in the same call or on
their own. A new version keeps the previous one's `capabilities` and state
rules unless you pass new ones.

Find an Artifact you made earlier with `list_artifacts` (`scope`: `owned`,
`shared`, or `all`), and read one with `get_artifact`. Never create a second
Artifact to "update" the first: the link the user already sent would go
stale.

## 5. Share it

```json
{
  "name": "share_artifact",
  "arguments": {
    "artifact_id": "art_…",
    "audience": "users",
    "share_with": [
      { "email": "ana@acme.com", "role": "editor" },
      { "email": "dan@acme.com", "role": "viewer" }
    ]
  }
}
```

| `audience` | Who can open it                                                       |
| ---------- | --------------------------------------------------------------------- |
| `owner`    | Only the owner. The default.                                          |
| `users`    | The people in `share_with`. A viewer opens it; an editor can publish. |
| `org`      | Every member of the bisque.team organization that owns it.            |
| `public`   | Anyone with the link, signed in or not.                               |

`org` is offered only to members of a bisque.team organization.
`share_with` is the complete list and replaces the previous one; omit it to
keep the current list. People are matched to their accounts by email. When an
address has no account, the call is refused naming it; tell the user who, and
that they can share again once that person signs up.

`listed: true` shows the Artifact in listings rather than keeping it
link-only.

**Going public deletes every comment thread.** Comments do not exist on a
public Artifact, so `share_artifact` refuses `public` on one that has threads
unless `delete_threads` is true. Pass it only after the user has said, in
words, that the comments can go.

## 6. Delete

`delete_artifact` removes it, and its URL stops working for everyone. Confirm
first, naming the Artifact by its title.

## Rules

- **Say what the link reaches.** When you hand over a URL, say who can open
  it: "Only you can open this until you share it" or "Anyone with the link
  can open this."
- **One Artifact per thing.** Update the one that exists rather than
  publishing a copy.
- **Confirm before anything that removes something:** deleting an Artifact,
  taking a person off it, or making it public when it has comments.
- **Never guess an email address.** Sharing with a guessed address shares
  with whoever owns it.
- **A refusal is an answer.** An unknown email, a public change on an
  Artifact with threads, or an organization that disallows external sharing
  each mean the change was wrong. Report it; do not retry around it.
- **Comments are handled by the `artifact-comments` skill.** When someone
  asks what people said on an Artifact, or to answer its comments, use that.
- **A presentation is another skill.** For slides that play with a voice,
  use the `present` skill from
  [bisque-cloud/presenter](https://github.com/bisque-cloud/presenter).
