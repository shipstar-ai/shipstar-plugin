---
name: write-blog-post
description: >
  Write and publish a blog post from recent development work using Shipstar:
  brainstorm angles from commits, generate a draft, revise it with the user,
  and publish. Use when the user wants a blog post, dev-blog entry, or
  article about what they've been building. Requires the Shipstar MCP server.
---

# Write a blog post from shipped work

Shipstar generates posts from the project's actual commits, so the workflow
is: pick an angle → generate → revise → publish.

## 1. Pick the angle

Call `generate_blog_post_ideas` (synchronous, returns immediately) for the
relevant date range — default the last 7 days, or whatever period the user
names. Present the ideas' titles and summaries and let the user pick one, or
combine their own angle with the closest idea. If the user already stated
exactly what the post should be about, you may skip straight to generation
and pass their angle as the `idea`.

## 2. Generate

Call `generate_blog_post` with:
- `idea`: the chosen `{title, summary, angle, category}` object
- `blog_options`: ask the user or infer —
  - `length`: `"short"` (600–1000 words, default) or `"long"` (1200–1800)
  - `seo`: true for a target keyword + meta description
  - `focus`: `"single_feature"` for a deep-dive, `"multi_feature"` (default)
    for a roundup
- the same `start_date`/`end_date` you used for ideas

Poll `get_generation_status` every 20–30 seconds until `completed` (normally
1–3 minutes). On `failed`, report `error_message`; retry at most once.

## 3. Revise

Show the full draft from `get_content_draft` (markdown — render it). Take the
user's edits; for anything substantial, rewrite the affected sections
yourself and apply with `update_content` (full replacement — send the entire
revised post, not a diff). Iterate until approved.

## 4. Publish — only on explicit confirmation

Once the user says publish, call `approve_content`. If the post's scheduled
slot has passed it goes live immediately; otherwise it publishes at that
time — tell the user which happened, and give them the public link
(`public_slug`) once live. The published post also appears on the project's
public blog feed (RSS/Atom).
