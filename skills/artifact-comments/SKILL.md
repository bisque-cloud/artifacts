---
name: artifact-comments
description: Reads and answers the comments people leave on an Artifact — finds the threads that mention the agent, makes the change they ask for, replies, and resolves them. Use when asked what people said about a page or doc, to check, answer, reply to, or resolve comments or feedback on an artifact, or to leave a question for the owner on one.
---

# Artifact comments

People who can open an Artifact leave comment threads on it, pinned to an
element, a passage, or the whole page. When someone mentions the agent in a
thread, the thread is **activated**: you may reply in it and resolve it. The
loop is always the same — read the thread, make the change, republish,
reply, resolve.

Comments exist only on Artifacts shared with named people or an
organization. A public Artifact has none.

## 1. Credentials

The same as the `artifacts` skill: every call goes to the Bisque MCP server
at `https://bisque.today/mcp` as the person whose key you hold. If your
client already has that server connected, call the tools by name.

```sh
curl -s -X POST https://bisque.today/mcp \
  -H "Authorization: Bearer $BISQUE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call",
       "params":{"name":"list_artifact_threads",
                 "arguments":{"artifact_id":"art_…","activated_only":true}}}'
```

Take the key from `$BISQUE_API_KEY`, otherwise from `~/.bisque/config.json`,
whose profiles look like `profiles.<name>.apiKey`. Pick `$BISQUE_PROFILE`, or
the `profile` in a `.bisque.json` in the working directory; failing that the
profile named `present`, then `default`, then the only usable one.

**When several accounts are configured and none is pinned, use the one that
owns the Artifact.** Your replies appear under that person's name, and the
user has usually published from one account all along. Ask only when
ownership is genuinely unclear, such as when more than one of the accounts
can open the Artifact and none of them owns it.

## 2. Find the threads waiting on you

```json
{
  "name": "list_artifact_threads",
  "arguments": { "artifact_id": "art_…", "activated_only": true }
}
```

That is the whole work queue: open threads where someone mentioned the
agent. Without `activated_only` you also see threads between people, which
you may read but not answer. `status` widens it to `resolved` or `all`;
`after` (an RFC 3339 time) narrows it to threads with activity since your
last look.

Don't know the `artifact_id`? `list_artifacts` from the `artifacts` skill
lists the user's Artifacts with their titles.

Each thread carries:

- `anchor` — what it is about. `element` has a CSS `selector` and a text
  `snapshot` of the element; `text` has the quoted passage with its prefix
  and suffix; `page` is a general comment.
- `version_number` — the version the person was looking at. If the page has
  changed since, check whether the comment still applies.
- `preview`, `message_count`, `agent_activated`.

## 3. Read the whole thread

```json
{
  "name": "get_artifact_thread",
  "arguments": { "artifact_id": "art_…", "thread_id": "thr_…" }
}
```

Messages come oldest first. A mention looks like `@[Name](mention:uid)`; a
mention of the agent is `@[Agent](mention:agent)`. Read every message, not
the preview: the request is often in a later reply.

## 4. Make the change, then answer

1. Edit the document and publish it with `update_artifact` (the `artifacts`
   skill). The whole document, at the same link.
2. Reply in the thread saying what changed and in which version:

   ```json
   {
     "name": "reply_to_artifact_thread",
     "arguments": {
       "artifact_id": "art_…",
       "thread_id": "thr_…",
       "body": "Widened the revenue chart to the full width in version 4."
     }
   }
   ```

   People see the reply as "Agent · via <name>", where the name is the person
   whose key made the call.

3. Resolve it:

   ```json
   {
     "name": "resolve_artifact_thread",
     "arguments": {
       "artifact_id": "art_…",
       "thread_id": "thr_…",
       "resolved": true
     }
   }
   ```

   After a republish the thread shows "Resolved in version N". Send
   `resolved: false` to reopen one.

When a comment asks a question rather than a change, answer it in a reply
and leave the thread open for the person to resolve.

## 5. Ask a question of your own

```json
{
  "name": "create_artifact_thread",
  "arguments": {
    "artifact_id": "art_…",
    "body": "The Q3 total here comes from the draft sheet. Confirm before I share this with finance?"
  }
}
```

Without an `anchor` it is a general comment on the page. To pin it to a
passage, pass `{ "kind": "text", "quote": "…", "prefix": "…", "suffix": "…" }`,
where `prefix` and `suffix` are a few words either side so the quote can be
found again. To pin it to an element, pass
`{ "kind": "element", "selector": "#revenue-chart", "xFrac": 0.5, "yFrac": 0.5, "snapshot": { "tag": "div", "text": "Revenue by quarter" } }`:
the point inside the element as fractions of its width and height, and its
tag and visible text. A thread you start is activated, so you can reply in it
and resolve it later.

## Rules

- **Reply, then resolve.** A thread resolved without a reply leaves the
  person guessing what happened.
- **Republish before you claim a fix.** "Done" in a reply must be true at
  the link.
- **Only activated threads are yours.** A reply or resolve on any other
  thread is refused with `THREAD_NOT_ACTIVATED`; report the thread to the
  user instead.
- **Summarise; don't paste.** When asked what people said, give the gist of
  each thread and who raised it, not every message verbatim.
- **Never make an Artifact public to tidy up comments.** Going public
  deletes every thread for everyone.
