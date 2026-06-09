# The Clean-Agent Evaluation

The feedback loop of the method (principle 7). You cannot judge your own repo's
agent-friendliness from the inside — you know too much. This harness has a
**fresh agent**, with no prior context, build something real using only your
**published** artifacts, and report exactly where it struggled.

## Why "clean" and "published" matter

- **Clean** — a new agent/session with none of your design discussions in
  context. It discovers the repo the way a real user's agent would.
- **Published** — it must fetch the *released* version (from the package
  registry / a fresh clone of the tag), **not** your working tree. This catches
  "the manual wasn't in the installed package" and "that fix isn't released yet"
  — common, invisible-from-the-inside failures.

## How to run

1. Create a **throwaway directory outside your project tree** and have the agent
   work only there (so it can't accidentally read your repo and contaminate the
   result).
2. Tell it to fetch the **published** artifact at a specific version.
3. Give it the prompt below, filled in.
4. Read its friction log. Triage each item: doc gap, design gap, or "agent error
   we can prevent." That list is your backlog.
5. Fix. Re-run on the next release. Track whether friction goes down.

> Tip: run it as a sub-agent / separate session. Withhold your design rationale —
> if the agent rediscovers a trap you "already documented," your docs aren't
> discoverable enough.

### Discovery-only mode (for read-only / sandboxed agents)

The full loop needs an agent that can **write files, compile, and execute** — not
always possible (e.g. a sandboxed sub-agent with no write/exec). When the build
can't run, fall back to **discovery-only**: have the fresh agent *read the
published artifact and report first-contact friction without building it*. Tell
it to: discover the docs as a real agent would, write the program it *would* ship
and verify each call against the source, and log every point it had to read
source, guess, or where a doc was missing/wrong. You lose the runtime
verification (does it actually work?) but keep most of the **discoverability and
selection signal**, which is what agent-readiness is really about. State in the
report that it was discovery-only. (An A/B — one agent with your scaffolded
manual, one without — makes the friction delta explicit.)

## The prompt (fill in the {braces})

```
You are evaluating a third-party {language} {library/service} by building a real
tool with it. Approach this as a fresh engineer who has never seen it before —
your honest first-contact experience is the deliverable.

## Task
Build a working {non-trivial but well-scoped tool — e.g. a CLI / parser / client}
that genuinely exercises {Project Name} ({how to fetch the PUBLISHED version,
e.g. `go get {module}@{version}` / `npm i {pkg}@{version}` / clone tag {tag}}).
The point is NOT just to ship the tool — it is to evaluate how easy or hard this
is for an LLM agent, and to write a candid report.

## Ground rules (for a fair evaluation)
1. Work ONLY in this fresh directory: {/abs/path/to/throwaway}. Initialize a new
   project there.
2. Do NOT read anything under {the project's own source tree} — pretend it does
   not exist; reading it would contaminate the evaluation.
3. Use the PUBLISHED artifact from the registry, not any local/linked copy.
4. {Any environment notes — e.g. network access for fetching deps.}
5. Discover the library as an agent naturally would: its README, in-language
   docs, and any shipped docs. {Point to an authoritative external spec if the
   task needs one.}

## What to build
{2–4 concrete requirements that force real use of the core API, including the
parts you suspect are error-prone — variable-length data, length/size fields,
polymorphism, encodings, whatever your domain's sharp edges are.}

## Verification (do it — don't just claim it works)
- Write tests proving a round trip / correct behavior.
- {An INDEPENDENT cross-check — e.g. interop with a standard external tool, a
  reference implementation, or the platform's own library — and capture its
  actual output.}
- Run the formatter/linter/vet.

## Deliverable: the evaluation report (write it to {path}/EVALUATION.md)
Be candid and specific; criticism is more useful than praise. Cover:
- Discoverability: how easy was it to learn how to use this? Did you find
  agent-oriented docs, and did they help?
- API & ergonomics: how well did it map onto the real task? Intuitive vs
  surprising.
- Friction log (most valuable): every point you got stuck, guessed wrong, hit a
  confusing error, or had to read source. Symptom + how you resolved it.
- {Domain-specific sharp edge}: how did you handle {the error-prone part}? Manual
  and fragile, or did the API help?
- Any signature/argument-order/mode confusion.
- Overall: a 1–5 ease-of-use rating FOR AN LLM AGENT, with justification, and a
  short list of concrete improvements you'd want.

## Final message back
A concise summary: the rating, whether the build fully worked (with the
verification results), the top 3 friction points, and any feature that notably
reduced friction. Include paths to the code and the report.
```

## Reading the result

- A **high rating with a specific gap** is the ideal outcome — it both validates
  the work and hands you the next task. (In practice an eval has scored highly
  *and* surfaced a concrete missing feature that became the next release.)
- If the agent **re-derived a trap you thought was documented**, the doc exists
  but isn't discoverable — fix placement, not content.
- If it **read your source to make things work** (not just to double-check), a
  doc or API gap sent it there.
