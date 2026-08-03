# Junction

Build an AI pipeline as a graph, then leave with the code.

**Use it:** https://rahulatrkm.github.io/junction/

Write the pipeline as readable YAML or draw it on the canvas — they are the same
thing, so they can never disagree. Run it against your own API key. Export plain
Python that has no dependency on Junction at all.

Free, no account, no server. Nothing you write leaves your browser.

## Why this exists

AI pipelines get ugly fast in Python. Prompt, call, parse, retry, branch, call
again, handle the failure — a few hundred lines later nobody can see the shape
of the thing any more.

Drawing it helps. Being locked into someone's runtime does not.

Every comparable tool wants to become a dependency: their SDK, their hosted
runner, their format. Junction's whole position is the opposite. It shows you
the shape, runs it once so you can see it work, and then hands you code you own
and gets out of the way. If this page disappears tomorrow your pipeline still
runs.

## How it works

The YAML is the single source of truth. Edges are not stored anywhere — they are
read out of the `{{references}}` in each step. Connect two boxes on the canvas
and it inserts a reference into the target's prompt. There is no second copy of
the graph to drift out of step.

```yaml
name: Support triage
inputs:
  - ticket

steps:
  category:
    type: prompt
    system: You sort support tickets. Answer with one word only.
    prompt: |
      Ticket: {{ticket}}
      Answer with exactly one of: billing, bug, other

  route:
    type: branch
    on: category
    when:
      billing: billing_reply
      bug: bug_reply
    else: general_reply

  billing_reply:
    type: prompt
    prompt: |
      Write a short, warm reply about this billing issue.
      Ticket: {{ticket}}

output: billing_reply
```

Four kinds of step, which is enough to build most real things:

| Type | What it does |
| --- | --- |
| `prompt` | Calls the model. Takes `system`, `model`, `temperature`, `json`. |
| `text` | Fills a template. No model call, no cost. |
| `branch` | Reads one value and picks which step runs next. |
| `each` | Fans out over a list and runs the prompt for every item. |

Canvas edits patch the exact lines they touch rather than re-serialising the
document, so your comments, blank lines and ordering survive.

## Your key

There is no server here. Your key is kept in this browser's local storage and
sent only to the endpoint you choose. It never reaches me, because there is
nowhere for it to reach.

That also means the call is made from the page, so only use a key you are happy
to have sitting on that machine. Anything speaking the OpenAI chat-completions
API works — OpenAI, OpenRouter, Groq, or Ollama on localhost, which costs
nothing at all.

## What it will not do

It will not host your pipeline, schedule it, or run it for you. It has no
accounts and stores nothing. If you want a managed runtime, several good ones
exist and this is not one of them.

It also only speaks the chat-completions API. No embeddings, no vector stores,
no tool calling yet.

## Tests

```bash
node junction.test.mjs      # 98 assertions
python3 test_export.py      # the exported Python is compiled by Python
```

The JavaScript suite runs the real page against a stubbed DOM and drives the
YAML subset, the graph derived from references, the text surgery canvas edits
depend on, and the code generator.

The Python suite exists because "looks like Python" is not "is Python" — it
exports every example and asks Python itself to compile it. A generator that
emits plausible-but-broken code would break the one promise this tool makes, and
no amount of JavaScript testing would catch it.

Two real bugs came out of writing these. The topological sort only followed data
edges, so a branch target that depended on an input could be ordered *before*
the branch meant to decide whether it ran at all — it executed unconditionally,
and the exported Python had no `if` around it. And the wire layer did not share
the node layer's transform origin, so every arrow drifted away from its box as
soon as you zoomed.

## Run locally

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

## License

MIT.
