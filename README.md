# Junction

Build an AI pipeline — or an agent with tools — as a graph, then leave with the code.

**Use it:** https://rahulatrkm.github.io/junction/

Write the pipeline as readable YAML or draw it on the canvas — they are the same
thing, so they can never disagree. Run it against your own API key. Export plain
Python or TypeScript that has no dependency on Junction at all.

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

## Why this rather than the tools you already know

A feature table would be marketing. The honest version:

**LangChain, LlamaIndex, LangGraph** are libraries — you write the code and they
own the abstractions, which have a habit of moving underneath you. They can do
far more than this can. Junction is not competing on capability; it writes the
code and then leaves, so there is nothing to move.

**Flowise and Langflow** are also visual, also open source, and genuinely good.
But the graph stays theirs: you export their JSON and run their server. Here the
export is plain Python or TypeScript that imports nothing of mine, and the test
suite runs the generated code on every commit to prove it.

**Dify** is a platform — accounts, hosting, datasets, observability. If that is
what you want, use it; it is a much bigger thing than this. Junction is one HTML
file with no server to run and no account to make.

**Braintrust, Langfuse and Humanloop** do evals and tracing well, as a separate
system you integrate and your prompts live on. Here the cases sit in the same
file as the pipeline and export alongside it.

So the reason to reach for this one:

- **You leave with the code.** Not a config file, not an API endpoint. Python or
  TypeScript, idiomatic and dependency-free.
- **Pipelines compose.** One can use another, and it exports as a function call —
  see [Pipelines that use pipelines](#pipelines-that-use-pipelines).
- **You can add your own step types, tool kinds, checks and languages** without
  forking it — see [Extending it](#extending-it).
- **Nothing to install, nothing to sign up for.** Open the page. Your key stays
  in your browser and the calls go straight to your endpoint.
- **The diagram and the text are one object.** Edges are read out of the
  `{{references}}`, so there is no second copy to drift.
- **Changing a prompt costs one call, not a whole run** — see
  [One step at a time](#one-step-at-a-time).
- **It tells you why it broke, and fixes what it can** — see
  [When it is slow, or broken](#when-it-is-slow-or-broken).
- **What must stay true is written down and travels with it** — see
  [Checks](#checks).

And the reasons not to: it will not host anything, schedule anything, or run
anything for you. There is no retrieval, no vector store, and no team features.
It is where you design a pipeline and prove it, not where you operate it.

Everything else is a registry you can add to — step types, tool kinds, checks and
languages. See [Extending it](#extending-it).

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
| `each` | Fans out over a list and runs the prompt for every item. Takes `concurrent:` to run several at once. |
| `agent` | Given a goal and some tools, works out its own steps. |
| `use` | Runs another pipeline you saved, and hands back what it returned. |

A `tests:` block pins what must stay true when you change a prompt — see
[Checks](#checks).

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

## One step at a time

Prompt work is a loop: change the wording, look at what comes back, change it
again. Running the whole pipeline to see the fourth step means paying for the
first three every time, in seconds and in tokens, for answers you already have.

Click a step, edit its prompt, press **Run this step**. It reuses whatever the
last run produced upstream and calls the model once. The answer appears under
the prompt you just edited — not in a panel covering it — along with what it
reused, and the previous answer so you can tell whether the change actually
helped.

If nothing upstream has run yet it says so, and names the step it needs, rather
than quietly running with empty values. After a `Test` run the values come from
the last case, so a failing case can be picked apart one step at a time.

## The language you actually work in

An export you have to hand-port is not an export. **Python** and **TypeScript**
both come out idiomatic and dependency-free — same pipeline, same checks, same
behaviour, because both are generated from one shared plan rather than two
separate ideas of what the pipeline does.

```bash
pip install openai  &&  python pipeline.py          # Python
npm install openai  &&  node pipeline.ts            # TypeScript, no build step
```

Both carry the whole thing: branches become real `if` statements, `each` becomes
a real loop, agents get the tool-calling loop, and the calculator is a parser in
each language rather than an `eval`. The checks come too, and
`python pipeline.py test` / `node pipeline.ts test` exit non-zero on failure.

Adding a language is a dialect, not a rewrite — roughly one screen of code
against the shared plan. What is **not** here yet is Go, Rust, Java and C#. I
will not ship a target I cannot execute in the test suite, and there is no
toolchain for those on this machine yet; an emitter nobody has run is a promise,
not a feature.

## Pipelines that use pipelines

A pipeline you cannot use inside another is an island, and a shelf of islands is
not a library. Save one, then `use` it anywhere:

```yaml
steps:
  sorted:
    type: use
    pipeline: Support triage     # one you saved
    with:
      ticket: "{{ticket}}"

  reply:
    type: prompt
    prompt: "Write a {{sorted}} reply to {{ticket}}"
```

It runs, and hands back whatever it returned. Its steps appear in the run panel
under the step that called it — `sorted › category` — so a nested run is still
something you can watch rather than a box that returns an answer.

**In the export it is a function call**, which is what it always was:

```python
def triage(ticket):
    category = ask(f"""sort {ticket}""")
    return category


def run(ticket):
    sorted = triage(f"""{ticket}""")
    reply = ask(f"""Write a {sorted} reply to {ticket}""")
    return reply
```

Nested pipelines are defined before they are called, deduplicated if used twice,
and any helper only the inner one needs — the agent loop, a tool, the calculator
— is still emitted. Both languages, both verified by running the result.

Two things it refuses rather than doing quietly: a pipeline that uses itself, or
two that use each other, is a loop and is reported as one; and a pipeline with
its own errors cannot be used, with its problem quoted so you know which one to
open. Nesting stops at five deep.

One honest limit: a **share link carries the document, not the shelf it was
standing on**. If a shared pipeline uses saved ones, the link says so when you
copy it, and the recipient is told exactly which pipeline is missing.

## When it is slow, or broken

A run that only tells you *what* happened leaves you to work out *why*. After
every run and every test, Junction reads what actually happened and says what it
found — with the button that fixes it where a fix exists.

**"Failed to fetch"** is the worst error in this business, because it is what the
browser says for a dozen different problems. Junction reads it as what it almost
always is:

> The browser blocked the call before it reached the endpoint. Almost always
> CORS: the endpoint did not say a web page may call it. Your key is probably
> fine — this is a browser-only limit, and the exported code does not have it
> because it does not run in a browser.

The provider's original message is always kept underneath. This adds the reading;
it never replaces the evidence.

| What happened | What it says, and offers |
| --- | --- |
| Blocked by the browser | CORS, or a local model that is not running — opens the connection settings |
| `401` / `403` | The key was refused, or is not allowed to do this |
| `404` | Wrong address — usually a missing `/v1`, or a model this provider does not serve |
| `429` / `5xx` | Rate limited or a provider wobble — **adds `retry: 2`** |
| Context length | The prompt outgrew the model — suggests splitting with an `each` |
| Agent gave up | Out of steps — **raises `max_steps`** |

For speed it uses the timings it already took, so it can say *"8400ms, about
1200ms each"* rather than guessing:

- **A fan-out running one at a time.** Ten sections summarised in sequence is ten
  round trips waiting on each other. One click writes `concurrent: 4`, and the
  page and *both exports* then run four at a time — a bounded pool, because rate
  limits are real and a hundred requests in flight is how you find that out.
- **The slow step**, named, with its share of the run.
- **Steps that wait on each other for no reason** — nothing reads the other, so
  the second is only slow because it is second.

Nothing here is a guess about your bill. There are still no invented prices.

## Extending it

The step types, tool kinds, checks and languages are registries, and all four are
open. An extension can add to any of them without you forking anything.

```js
export default function (junction) {
  junction.defineCheck("one_word", {
    label: "is a single word",
    failed: "is not a single word",
    test: (got) => got.length > 0 && !/\s/.test(got),
    emit: {
      python: 'len(got) > 0 and " " not in got',
      typescript: "got.length > 0 && !/\\s/.test(got)",
    },
  });

  junction.defineStep("agree", {
    label: "Agree",
    needs: ["prompt"],
    async run(step, ctx) {
      const prompt = ctx.fill(step.prompt);
      const first = await ctx.ask(prompt, { temperature: 0 });
      const second = await ctx.ask(prompt, { temperature: 0 });
      return first.trim() === second.trim() ? first : `${first} (it changed its mind)`;
    },
    emit: { python: (step, api) => [...], typescript: (step, api) => [...] },
  });
}
```

Install it by address from the **Extensions** panel. There are two worked
examples — [extensions/lengths.js](extensions/lengths.js) for checks and
[extensions/agree.js](extensions/agree.js) for a step type and a tool kind. They
are the documentation, and the test suites register them through the real API and
run what they emit, so they cannot quietly go stale.

| Call | Adds |
| --- | --- |
| `defineCheck(name, def)` | a check for `tests:` — `test`, and `emit` per language |
| `defineStep(type, def)` | a `type:` — `needs`, `run(step, ctx)`, and `emit` per language |
| `defineTool(kind, def)` | a tool `kind:` — `params`, `run(tool, args)`, and `emit` |
| `defineLanguage(id, def)` | a language — `emit(plan, cfg)` |

A step's `run` gets `ctx.ask(prompt, options)`, `ctx.fill(template)` and
`ctx.values`. Nothing else: no page, no key, no registries.

### emit is not optional in spirit

It is the part people skip, and the reason this took a while to build. Anything
you add has to be writable into the exported code, or your pipeline runs here and
can never leave — which breaks the one promise everything else rests on.

So a missing emitter **refuses that language**, in the export panel, before you
copy anything:

> This pipeline cannot be written in TypeScript: the "py_only" step type cannot
> be written in TypeScript. The extension that added it has to teach it this
> language, or choose another one above.

The other languages still work. A check with no emitter is louder still — the
generated file raises rather than letting a check silently pass in CI.

`emit` is handed the escaping helpers rather than left to reinvent them:
`api.string(text)` turns a template with `{{references}}` into a correctly
escaped literal for that language, which is the easiest thing to get wrong by
hand.

### The line that matters

**A pipeline can never install an extension.** Not a document, not a shared link,
not an example. Extensions come only from that panel, from somebody typing an
address and pressing a button. The test suite opens a pipeline that claims
`extensions:` and asserts nothing is installed. That keeps the promise this has
always made — nothing evaluates code you were *sent* — while letting you run code
you *chose*.

What it is not is a sandbox. An installed extension is code running in this page,
with the same reach as anything you install from npm. Read it first. Nothing here
pretends otherwise.

An extension cannot replace a built-in: a name collision is refused with a
message rather than silently winning. One that throws while registering is
reported and skipped, and one whose `test` throws during a run fails that check
instead of taking the run down.

## Checks

Changing a prompt is the easiest thing in the world to do and the hardest to be
sure about. A `tests:` block says what has to stay true, so you can change a
prompt and find out in seconds instead of finding out from a customer.

```yaml
tests:
  a_double_charge_is_billing:
    inputs:
      ticket: I have been charged twice for the same month
    expect:
      category:
        is_one_of: [billing, bug, other]
      billing_reply:
        not_empty: yes
        not_contains: refund

  a_crash_is_a_bug:
    inputs:
      ticket: the export button does nothing
    expect:
      category: bug          # the short form means equals
```

Press **Test** and every case runs the whole pipeline from clean values, so one
case cannot quietly change what the next one sees. A failure tells you the step,
what was wanted, and what actually came back — every failing check at once, not
one at a time.

That second `not_contains: refund` is the interesting one. The prompt above says
*"Do not promise a refund"*, which until now was a wish. Now it is checked.

| Check | Passes when |
| --- | --- |
| `equals` | the whole answer matches, ignoring case and surrounding space |
| `contains` | the phrase appears somewhere |
| `not_contains` | the phrase appears nowhere |
| `matches` | a regular expression finds something |
| `is_one_of` | the answer is one of a list |
| `json` | it parses as JSON (`json: no` insists it does not) |
| `not_empty` | there is anything there at all |

Checks are declarative for the same reason the tools are: a pipeline you can
share carries its cases with it, and nothing here evaluates a string you were
sent.

Expecting something of a step the branch skipped is a failure, not a pass —
otherwise a routing bug would quietly satisfy every case it never reached.

The cases export with the pipeline. `python your_pipeline.py test` runs them and
exits non-zero if any failed, which is all CI needs:

```
pass a_crash_is_a_bug
FAIL a_double_charge_is_billing
    billing_reply should not contain 'refund'
1 of 2 passed
```

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
stores, no streaming yet. Agents run one at a time, and steps that do not read
each other still run in order — a fan-out can go several at once, but two
unrelated steps cannot.

## Tests

```bash
node junction.test.mjs      # 391 assertions against the real page
python3 test_export.py      # the exported Python is compiled and run by Python
python3 test_export_ts.py   # the exported TypeScript is run by Node
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
