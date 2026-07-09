<div align="center">

# 🌅 Aurora

### Run AI agents that take **real actions** — safely.

Aurora is a runtime for AI agents that do more than chat: they call APIs, run
commands, move money, change production. It makes every action an agent takes
**recorded, crash‑proof, reversible, and approval‑gated** — so you can trust an
agent with real authority without trusting it blindly.

*Built in Go and Rust, on WebAssembly and capability security.*

</div>

---

## The problem Aurora solves

The moment you let an AI agent run *real* commands, three things go wrong — and no
ordinary stack answers them:

> **1. You can't trust the logs.** The agent just ran twelve actions against
> production. Show me the record — not the output it *chose* to print, a record it
> **couldn't skip or forge**.
>
> **2. Crashes double‑charge you.** It died at step 7 of 12, and step 3 was a
> payment. You restart it… and steps 1–6 run again. **You just paid twice.**
>
> **3. The agent guards its own gate.** "A human must approve this delete" lives in
> the agent's prompt — which the AI controls. **The prisoner writes the prison
> rules.**

Aurora is the runtime where those three have answers:

- ✅ **Every side effect goes through one recorded gate** the agent can't bypass —
  an un‑forgeable audit trail.
- ✅ **Crashes replay to the exact instruction**, serving already‑committed results
  from the journal — no double‑charges, no repeated payments.
- ✅ **Approval lives *outside* the sandbox**, where the agent can't approve for
  itself.

The agent's decision‑making runs as a **WebAssembly guest with zero ambient
authority**: it can't touch the network, disk, or an LLM on its own. It can only
*ask* the host — one capability‑checked, journaled syscall at a time.

## How it fits together

Aurora is a small family of repositories that stack from a kernel up to the tools
you actually talk to:

```
                    you (a human)
                          │
        ┌─────────────────┴──────────────────┐
     aurora-cli                    aurora-slack-connector      ← clients you talk to
        └─────────────────┬──────────────────┘
                          │  HTTP /v1
                     aurora-dist                               ← the server (one binary you run)
                          │  assembled from…
        ┌─────────────────┼──────────────────┐
   aurora-capcompute   aurora-dispatchers   capcompute
   (orchestration)     (capability drivers)  (the kernel)
                          │
                   aurora-brains                               ← the WASM agent "programs" that run inside
```

Read it bottom‑to‑top: **capcompute** is the kernel that sandboxes and records;
**aurora-capcompute** turns it into a durable agent runtime; **aurora-dispatchers**
and **aurora-brains** plug in the tools and the cognition; **aurora-dist** bundles it
all into one server; and **aurora-cli** / **aurora-slack-connector** are how you drive
it.

## The repositories

| Repo | What it is |
| --- | --- |
| 🖥️ **[aurora-dist](https://github.com/aurora-capcompute/aurora-dist)** | **Start here to run Aurora.** One binary that bundles the whole runtime and serves it over an HTTP `/v1` API. |
| ⌨️ **[aurora-cli](https://github.com/aurora-capcompute/aurora-cli)** | A terminal shell that drives a server — browse sessions like a filesystem, spawn agents, approve their actions. |
| 💬 **[aurora-slack-connector](https://github.com/aurora-capcompute/aurora-slack-connector)** | Turns a Slack channel into an on‑call duty bot backed by Aurora, with in‑thread Approve/Deny buttons. |
| 🧠 **[aurora-brains](https://github.com/aurora-capcompute/aurora-brains)** | The agent "programs" (Rust → WebAssembly): the general‑purpose agent, a prompt‑injection‑resistant planner, and more. |
| 🔌 **[aurora-dispatchers](https://github.com/aurora-capcompute/aurora-dispatchers)** | The capability drivers — internet, filesystem, shared memory, and any OpenAI‑compatible LLM — each scoped, gated, and recorded. |
| ⚙️ **[aurora-capcompute](https://github.com/aurora-capcompute/aurora-capcompute)** | The orchestration runtime: sessions, retries, rollback, human‑in‑the‑loop approvals, and delegation to sub‑agents. |
| 🛡️ **[capcompute](https://github.com/aurora-capcompute/capcompute)** | The kernel — a tiny "operating system" where WASM guests are processes and host features are syscalls. The foundation everything else is built on. |

Every repo has its own beginner‑friendly README with a 5‑minute quick start.

## Get started as fast as possible

Go from zero to a running agent you can talk to from your terminal. **You need:**
Go 1.26+, a Rust toolchain with the `wasm32-wasip1` target, and an OpenAI‑compatible
API key.

**1. Build an agent program** (from `aurora-brains`):

```sh
git clone https://github.com/aurora-capcompute/aurora-brains
cd aurora-brains
rustup target add wasm32-wasip1
sh programs/agent/build.sh          # → programs/agent/dist/{agent.wasm, agent.json}
```

**2. Build and run the server** (`aurora-dist`):

```sh
git clone https://github.com/aurora-capcompute/aurora-dist
cd aurora-dist
mkdir -p programs && cp ../aurora-brains/programs/agent/dist/agent.* programs/
go build ./cmd/aurora-dist

export AURORA_TASK_SECRET=change-me-at-least-16-bytes   # required, ≥ 16 bytes
export AURORA_SECRET_OPENAI_KEY=sk-...                  # your LLM key, resolved host-side
./aurora-dist -addr :8080 -data ./data -programs ./programs
```

The server is now up on `http://localhost:8080` (`curl localhost:8080/healthz` → `ok`).

**3. Talk to it** (`aurora-cli`):

```sh
git clone https://github.com/aurora-capcompute/aurora-cli
cd aurora-cli && go build ./cmd/aurora

cat > manifest.json <<'EOF'
{"version": 4, "syscalls": [
  {"syscall": "core.openaiApi", "hidden": true,
   "base_url": "https://api.openai.com/v1", "api_key": {"secret": "OPENAI_KEY"},
   "default_model": "gpt-4o", "capabilities": [{"operation": "chat"}]},
  {"syscall": "core.internet", "capabilities": [{"methods": ["GET"], "domain": "example.com"}]}
]}
EOF

./aurora mount http://127.0.0.1:8080
./aurora mkdir demo && ./aurora cd demo
export AURORA_MANIFEST=manifest.json
./aurora spawn "fetch example.com and summarize it"
```

That last line runs a real agent: it calls the LLM, fetches the page (only because
the manifest granted it `GET example.com`), and prints its answer — with every step
journaled and replayable. 🎉

## How it works, in one minute

- **Manifests are permission slips.** An agent can only make the syscalls its
  manifest grants, checked against a JSON schema. Nothing else is reachable.
- **The journal is the source of truth.** Every syscall is written down *before* it
  runs and *before* the agent sees the result, hash‑chained. Durable state is a
  *fold of that append‑only log* — no mutable database.
- **Replay = crash recovery + audit + exactly‑once**, all from one structure. A
  restarted process resumes at the exact instruction and serves committed effects
  from the journal instead of re‑running them.
- **Approvals and rollback are first‑class.** A sensitive syscall can pause the
  agent as a durable task until a human approves; agents register undo actions and
  can cleanly roll back partial work (saga‑style compensation).
- **Delegation is safe.** An agent can spawn sub‑agents, but a child can never be
  granted more authority than its parent holds.

## Tech at a glance

- **Languages:** Go (kernel, runtime, server, clients) and Rust (agent programs).
- **Sandbox:** WebAssembly via [Extism](https://extism.org/) / [wazero](https://wazero.io/).
- **Model:** capability security + event sourcing + deterministic replay — ideas
  from seL4/KeyKOS, Temporal, and CaMeL, in one runtime.

---

<div align="center">

**New here?** → Run it in 5 minutes with **[aurora-dist](https://github.com/aurora-capcompute/aurora-dist)**
· Drive it with **[aurora-cli](https://github.com/aurora-capcompute/aurora-cli)**
· Understand the foundation in **[capcompute](https://github.com/aurora-capcompute/capcompute)**

</div>
