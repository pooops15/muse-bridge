# muse-bridge Tutorial — Complete, Beginner-Friendly

> 🇮🇩 Versi Bahasa Indonesia: [TUTORIAL.id.md](TUTORIAL.id.md)

This tutorial shows you how to make **Muse (an AI agent)** work as a model
inside **9Router**, so a Hermes agent can use it like any real API model
(ChatGPT, DeepSeek, Claude) — **including native tool calls**. It is written
slowly, in plain language. If you can copy-paste and read carefully, you can
do this.

---

## Part 0 — What are we building?

Think of a restaurant:

| Thing in this tutorial | In the restaurant it is |
|---|---|
| **Hermes** | The waiter. Talks to you (on Telegram), owns the hands that do work |
| **9Router** | The smart order book. Every model request goes through it |
| **Bridge** | A small notebook in the VPS kitchen. Orders are written here as files |
| **Courier (hook)** | A kid who checks the notebook every 5 seconds and wakes the cook when there's an order |
| **Worker / cook** | The Muse agent. Does the thinking, writes the answers |
| **Tool calls** | A note back from the cook: "waiter, please run this for me" |
| **VPS** | The waiter's house. The kitchen and all tools live here |

The core problem: the cook lives in another house and **has no phone**
(there is no API for the agent). So we use a notebook and a courier. Requests
become files; the courier carries them back and forth. It is slower than a
real API, but it works — and the cook is genuinely smart, not a canned model.

End goal:

- Your 9Router has a model named **`ms/muse`**
- Hermes can select it like any other model
- It can **chat**, **request tools** (Hermes executes them for real on the
  VPS), and **see images** you send

## Part 1 — Ingredients (you must already have these)

Check them one by one, don't skip:

1. **One VPS** (Linux, you hold root) — the waiter's house
2. **9Router** installed and running on that VPS (usually port `20128`)
3. **Hermes** installed on the same VPS and connected to that 9Router
   (any model at first, as long as it works)
4. **One Muse** somewhere else (in this tutorial: Muse on a sandbox
   VM) with a **hook** feature — a small script that runs every few seconds
   and can wake the agent
5. **SSH from the agent's machine to the VPS** must work (key-based, not
   password)

> No Muse account for item 4 yet? First follow
> **[How to Claim Muse — With a VPN or Without One](CLAIM-MUSE.md)** and
> redeem referral code **`MZMT4Y`** within 48 hours after joining.

If items 2 and 3 are not done yet, finish those first. This tutorial starts
from "9Router and Hermes are alive".

Terms used often:

- **Queue**: the folder where order files wait
- **Pending**: an order not yet answered
- **Done**: a finished answer
- **Provider**: a "model brand" inside 9Router. Ours has prefix `ms` and
  model `muse`, so the full name is `ms/muse`

## Part 2 — Install the bridge on the VPS

The bridge is a small program (one Python file, no exotic libraries) that:

1. Receives requests from 9Router at `http://127.0.0.1:8765/v1/chat/completions`
2. Saves each request as a file in `queue/pending/`
3. Waits for an answer file to appear in `queue/done/`
4. Returns that answer to 9Router like a normal model response

> **Fast way:** steps 2.1–2.4 below are automated by the installer scripts —
> `scripts/install-linux.sh` (Linux), `scripts/install-macos.sh` (macOS),
> `scripts/install-windows.ps1` (Windows). They create the folders, install
> the bridge, register it as an always-on service, and run the health
> check. The manual steps below are the same thing, spelled out so you
> know what the scripts do.

### Step 2.1 — Create the folders

On the VPS:

```bash
mkdir -p /opt/muse-bridge/queue/pending /opt/muse-bridge/queue/done
```

### Step 2.2 — Place the bridge file

Copy `bridge/bridge.py` from this repo to:

```
/opt/muse-bridge/bridge.py
```

### Step 2.3 — Make it a service so it stays up

Create `/etc/systemd/system/muse-bridge.service`:

```ini
[Unit]
Description=Muse bridge for 9Router
After=network.target

[Service]
ExecStart=/usr/bin/python3 /opt/muse-bridge/bridge.py
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

Then start it:

```bash
systemctl daemon-reload
systemctl enable --now muse-bridge
```

### Step 2.4 — Check it is alive

```bash
curl http://127.0.0.1:8765/health
curl http://127.0.0.1:8765/v1/models
```

If both answer normally, the bridge is healthy. **The bridge listens on
localhost only (127.0.0.1)** — it is not exposed to the internet, on purpose.
Only 9Router on the same machine talks to it.

## Part 3 — SSH key from the cook's house to the VPS

The courier and the worker live on the agent's machine. They fetch and
deliver queue files **over SSH**, so the agent machine must be able to log
into the VPS with a key.

### Step 3.1 — Create a key on the agent machine (if you don't have one)

```bash
ssh-keygen -t ed25519
cat ~/.ssh/id_ed25519.pub
```

The long line that comes out is your **public key**. That one is safe to share.

### Step 3.2 — Put the public key on the VPS

On the VPS, append that public-key line to:

```
~/.ssh/authorized_keys
```

### Step 3.3 — Make wrappers so you don't type long commands forever

On the agent machine, keep the two scripts from this repo's `tools/` folder:

- `tools/ssh-vps.sh` — run a command on the VPS:
  `ssh-vps.sh "ls /opt/muse-bridge/queue/pending"`
- `tools/scp-vps.sh` — copy files to/from the VPS

Inside both files, replace the VPS address with your own VPS address
(the samples say `YOUR_VPS_IP`).

### Step 3.4 — Test

From the agent machine:

```bash
./ssh-vps.sh "echo hello from the vps"
```

If `hello from the vps` comes back, the path works.

> **Honest note:** in the original setup, SSH travels through a special
> proxy because the agent machine lives inside a sandbox. On a normal
> machine, direct SSH is fine. Adjust to wherever your agent lives.

## Part 4 — Register the provider in 9Router

Now tell 9Router: "there is a new model called muse, ask the bridge".

> ⚠️ **Expensive lesson, read this:** register the provider **through the
> 9Router dashboard**, NOT by injecting the database directly. We tried DB
> injection first and got a half-broken result: the model showed up in
> pickers but requests didn't route. The dashboard writes the correct
> format. The database is only safe for creating the *combo* (step 4.3).

### Step 4.1 — Create the provider node in the dashboard

In the 9Router dashboard (from your VPS localhost), add an
**OpenAI-compatible** provider:

| Field | Value |
|---|---|
| Name | `muse` |
| Prefix | `ms` |
| Base URL | `http://127.0.0.1:8765/v1` |

### Step 4.2 — Create a connection

On that provider, add a connection:

| Field | Value |
|---|---|
| Name | `test` (anything) |
| Auth type | API key |
| API key | Anything (this local bridge doesn't verify it — the field just needs to exist and be active) |

### Step 4.3 — Create the combo

Create a combo named **`muse`** containing exactly one model:

```
["ms/muse"]
```

The combo lets clients call the model simply as `muse`.

### Step 4.4 — Create a 9Router API key for clients

In the 9Router dashboard, create one API key. Hermes (or any other client)
uses it to talk to 9Router. Store it somewhere safe with file permission
`0600` — **never** in this repo or in chat.

## Part 5 — Install the courier + the worker's rulebook (agent side)

The most important part. Two things get installed on the machine where the
agent lives:

### Thing 1 — The courier script (hook)

Its job is only three things, and it is **dumb** (it does not think at all):

1. Every **5 seconds**, it lists `queue/pending/` on the VPS over SSH
2. If empty → stay silent. Done. **No agent wakes up, no tokens burn.**
   Checking is free
3. If there are new files → it marks them "claimed" (so nobody wakes twice
   for the same file) and **wakes one worker agent**

The script is roughly: one SSH listing command, count unclaimed files,
decide "silent" or "wake the worker".

### Thing 2 — The worker's rulebook (worker prompt)

When a worker wakes, it is given a rulebook. The current one lives in
`worker/worker-prompt-v4.txt`. In human words, it says:

**The worker's thinking order (must be in this order):**

1. **Read the order file.** It contains the Hermes request: chat history,
   the list of tools Hermes offers, and more
2. **Images/video present?** Extract them and **actually look** before
   answering. Nothing the client sends may be silently dropped
3. **Stale order?** If the order is older than 220 seconds, delete it and
   don't answer — the requester is no longer waiting
4. **Tool results already inside the order?** (this is round two) → write
   the final answer **from those real results**. Never invent results
5. **No tool results yet, and the task needs a real machine?** (check disk,
   read a file, inspect services) → **do not answer in text**. Write a
   tool-calls letter: the list of commands Hermes should run
6. **Otherwise** → answer in plain text like a normal model

**Rules for the tool-calls letter:**

- Only use tool names Hermes offered in the request
- **No limit on the number of calls** — but only batch calls that are
  **independent** of each other (not waiting on each other's results).
  Dependent steps stay one per round, because step two needs step one's
  result
- If the information already present is enough to answer, answer. Stop
- **State-changing** calls (delete, overwrite, restart, kill, install) only
  when the task actually asks for exactly that. Never add your own

**Honesty rules (the most expensive lesson in this project):**

- Never invent numbers, filenames, sizes, or service states
- Claim "I checked" only when there is a real result to show
- If asked "how do you access the VPS?", answer truthfully: requests arrive
  as queue files over SSH; tool calls are the official path through Hermes;
  SSH is the worker's own side channel

Why do these rules exist? Because a worker once **flip-flopped** inside a
single conversation: first refusing, then claiming it had checked the
machine with partly fabricated results, then denying everything. Two root
causes: every request is served by a **fresh** worker that sees only
message text (it doesn't remember what earlier workers did), and the old
prompt was ambiguous. The new rules fix both.

### How the worker sends answers

- Text answer → it writes `done/<id>.json` containing `{"content": "..."}`
- Tool-calls letter → the same file, containing `{"tool_calls": [...]}`
- The id is **always the same id as the order**. That is what makes answers
  impossible to swap between sessions: every order and answer carry the
  same label, and the bridge matches labels only

## Part 6 — Connect Hermes

Finally, tell Hermes to use its new model.

In your Hermes profile config, point the model at the local 9Router with
model name `muse` (the combo from step 4.3), using the 9Router API key from
step 4.4. Everything else (9Router base URL, etc.) is the same as for that
profile's other 9Router models.

### The approval safety gate (IMPORTANT, do not skip)

Hermes has a safety gate: before running a dangerous command (delete a
file, restart a service, overwrite a system file, and friends) it **asks
you first** in chat. On our setup, verified contents:

| Setting | Value |
|---|---|
| `approvals.mode` | `manual` — must ask first |
| Approval timeout | 60 seconds — no answer from you = it does not run |
| Commands from cron/schedules | `deny` — refused, cannot ask |
| Subagents approving themselves | Off |

This gate is the reason tool calls in this system can safely be
**unlimited in number**: the worker may propose as many commands as it
wants, but **you** remain the one who decides at Hermes's door whether
dangerous ones run.

One mode cancels all of that: **`/yolo`**. With yolo on, Hermes runs
everything without asking. Use yolo only when you fully mean it and are
watching. **Do not leave yolo on.**

## Part 7 — Test one by one (do not skip!)

Run them in order. Each test has a "pass sign".

### Test 1 — Bridge is alive

```bash
curl http://127.0.0.1:8765/health
```

✅ Pass: the bridge answers with an OK status.

### Test 2 — The model is visible through 9Router

From the VPS, list 9Router's models with your API key.

✅ Pass: `muse` (and `ms/muse`) are in the list.

### Test 3 — Plain chat

Send one chat completion with model `muse`, an easy question ("what is
2+2?").

✅ Pass: a real answer comes back ("4"). Be patient — the first answer can
take 20–30 seconds. That is normal for this system.

### Test 4 — Tool calls (the most important test)

From Hermes (Telegram), using model `ms/muse`, say:

> "Check the VPS disk now, using a tool"

✅ Pass looks like this:

1. Hermes **shows a tool step** (e.g. "💻 terminal `df -h`") — meaning
   Hermes executed the command itself on the VPS; it is not made up
2. The final answer quotes numbers **exactly matching** what you get when
   you run `df -h` yourself

If Hermes only gives you text commands to run manually, tool calls are not
connected — re-check Parts 4 and 5.

### Test 5 — Many checks at once

Tell Hermes:

> "Check disk, RAM, uptime, and service status all at once, using tools"

✅ Pass: in **one turn**, Hermes runs several tools (not drip-fed one per
answer).

### Test 6 — Image

Send any image to Hermes (model `ms/muse`) and ask what is in it.

✅ Pass: the answer matches the image — the worker really looked at it
instead of guessing.

### Test 7 — Two sessions at once (anti-swap)

Open two different topics/sessions and make both work at almost the same
time.

✅ Pass: each answer belongs to its own topic. The ID-label system makes
swaps impossible by design — this test is proof, not hope.

## Part 8 — Day-to-day use

- **Good for:** work where the result must come from real data (checking
  the VPS, reading logs, summaries, heavy thinking). A bit slow is fine
- **Not great for:** rapid back-and-forth chat, or tasks needing a dozen
  dependent steps — each step adds ±30 seconds
- **Patience rule:** under **2 minutes** is normal, not stuck. Stuck is
  when **4 minutes** pass with no answer
- **Remember the cost:** the **worker agent** pays the tokens, and it pays
  on every real request (each round it re-reads the whole session history,
  100–220 KB). Empty queue polls are free. So use this model as a
  "specialist", not for every small chat

## Part 9 — Troubleshooting

| Symptom | Most likely cause | Fix |
|---|---|---|
| `Muse bridge busy` error (429) | Queue full: 20 orders already waiting | Wait for running ones to finish, resend. It's a queue limit, not a breakage |
| Timeout / 504 after ±4 minutes | Worker answered too late — agent machine down, or queue backed up | Check the agent machine is alive; check the queue on the VPS (`ls /opt/muse-bridge/queue/pending`) |
| Hermes only answers text, never uses tools | Wrong provider/combo, or an old worker prompt | Check Part 4 (combo `muse` → `["ms/muse"]`) and use worker prompt v2+ |
| Answers are invented / "I checked" without proof | The honesty rules got lost from the worker prompt | Reinstall the prompt from `worker/worker-prompt-v4.txt` |
| Everything dead after the agent machine restarts | That machine resets easily? The agent-side stack (including the hook) must be restored | Keep a rebuild script + a watchdog that checks stack health and rebuilds automatically |
| Image sent but answer is a guess | Worker prompt has no media rule | Use prompt v4 (it contains the media rule) |
| SSH to the VPS is down | Key/network problem | The courier has an alarm: after repeated failures it notifies the owner. Test `ssh-vps.sh "echo ok"` manually |

## Part 10 — Safety & honest final words

**Safety:**

- Secrets (API keys, tokens) **never** enter this repo. Keep them in
  separate `0600` files on each machine
- The bridge listens on VPS localhost only. Never expose it to the
  internet
- Back up before tinkering: the old bridge file is kept as
  `bridge.py.bak-<date>`, the old worker prompt as a backup file.
  Rollback = restore the backup + restart the service. One minute
- Hermes's manual approval gate (Part 6) is the main safety key. Treat
  `/yolo` like a sharp knife: fine to use, never leave it lying around on

**Honest limits (so nobody is surprised):**

- This will **never be as fast as a real API** (2–5 s). The thing answering
  is an agent that must wake up and read context first. The realistic
  target is "90% of the API feel": complete protocol, stable, fast enough
- Orders are processed **in parallel**: a coordinator spawns one
  worker per order (max 8 at once, max 20 orders waiting). Heavy bursts
  still queue behind the per-round thinking time
- Video: the worker can only **sample a few frames**, not watch a whole
  video. Anyone claiming full video watching here is bluffing
- Every round re-reads the entire session history. Very long sessions get
  slower and more token-hungry. Starting a fresh session now and then is
  healthy

---

*This tutorial was written from a system that actually runs and was tested,
not from theory. The delays and sizes quoted above are real numbers from
the system's own logs. Good luck — if you get stuck, re-read Part 9 slowly.*
