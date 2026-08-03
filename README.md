# Junction

Build an AI pipeline — or an agent with tools — as a graph, then leave with the code.

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

Four kinds of step, plus agents, which is enough to build most real things:

| Type | What it does |
| --- | --- |
| `prompt` | Calls the model. Takes `system`, `model`, `temperature`, `json`, `retry`. |
| `text` | Fills a template. No model call, no cost. |
| `branch` | Reads one value and picks which step runs next. |
| `each` | Fans out over a list and runs the prompt for every item. |
| `agent` | Given a goal and some tools, works out its own steps. |

Canvas edits patch the exact lines they touch rather than re-serialising the
document, so your comments, blank lines and ordering survive.

## Agents

An agent gets a goal and a set of tools, then decides for itself what to call
and in what order, until it has an answer or runs out of steps.

```yaml
tools:
  wikipedia:
    kind: http
    description: Search Wikipedia and get back matching article summaries.
    params:
      query: The phrase to search for
    url: https://en.wikipedia.org/w/api.php?action=query&list=search&format=json&srsearch={{query}}

  calculate:
    kind: calc
    description: Work out an arithmetic expression exactly, rather than guessing.

steps:
  researcher:
    type: agent
    tools: [wikipedia, calculate]
    max_steps: 8
    goal: |
      Answer this, and show which tools you used: {{question}}
```

Every tool call is written into the run panel as it happens — which tool, what
arguments the model chose, what came back. An agent you cannot watch is an agent
you cannot debug.

### Tools are declarative, deliberately

An agent hands arguments it invented to whatever you list. So nothing here will
ever run code a model wrote. There is no `eval`, no `Function`, no shelling out,
and the test suite asserts that.

| Kind | What it is |
| --- | --- |
| `http` | A URL template you can read. Arguments are URL-encoded on the way in. |
| `calc` | Arithmetic, parsed by hand. It can only ever produce a number. |
| `clock` | The current time in UTC. |

HTTP tools must use `https`, or `http` only on `localhost`. That is checked when
you write the tool and again after the model's arguments are filled in, because
the arguments are not trusted. The calls come from your browser, so they are
also subject to the target's CORS policy — some endpoints will simply refuse.

If three kinds is not enough: export the Python and write a real function. That
is the whole point of the export.

## Everything else

- **Both directions.** Write YAML or click a node and edit it in a form. Drag
  between two nodes and it inserts the reference for you. It is one document
  either way.
- **Undo and redo** across every kind of edit, including the ones the canvas
  makes for you, which is exactly where the browser's own undo gives up.
- **Save** pipelines in this browser, download them as `.yaml`, or open one.
- **Share** copies a link that carries the whole pipeline in the fragment —
  the part of a URL browsers never send to a server. Nothing is stored anywhere
  and no link expires. If a shared pipeline defines HTTP tools, you are told so
  before you run it.
- **Token counts** per run, taken from what the API reports. No cost estimate,
  because I would have to invent the prices and they go stale.
- **`retry:`** on any step that calls a model, for the transient 429 that should
  not lose a whole run.

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

It speaks the chat-completions and tool-calling APIs. No embeddings, no vector
stores, no streaming yet. Agents run one at a time, not in parallel.

## Tests

```bash
node junction.test.mjs      # 170 assertions
python3 test_export.py      # the exported Python is compiled and run by Python
```

The JavaScript suite runs the real page against a stubbed DOM and drives the
YAML subset, the graph derived from references, the text surgery canvas edits
depend on, the tool definitions, the arithmetic parser, and the code generator.

The Python suite exists because "looks like Python" is not "is Python" — it
exports every example and asks Python itself to compile it. It then goes further
and *imports* the generated agent code to attack the calculator it emits,
because that function is handed strings a model invented:

```python
"__import__('os').system('echo pwned')"
"().__class__.__bases__[0].__subclasses__()"
```

Both are refused, in the generated code, by Python, at test time.

Four real bugs came out of writing these. The topological sort only followed
data edges, so a branch target that depended on an input could be ordered
*before* the branch meant to decide whether it ran at all — it executed
unconditionally, and the exported Python had no `if` around it. The wire layer
did not share the node layer's transform origin, so every arrow drifted away
from its box as soon as you zoomed. The privacy check that asserted the page
"talks to nothing unexpected" was reading example data as though it were app
code. And the tests pinned examples by array index, so adding one silently moved
every assertion onto the wrong pipeline.

## Run locally

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

## License

MIT.
