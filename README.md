# muse-bridge

![muse-bridge banner](docs/images/banner.svg)

![License: MIT](https://img.shields.io/badge/license-MIT-green)
![Python 3](https://img.shields.io/badge/python-3.x-blue)
![Tests](https://img.shields.io/badge/tests-9%2F9_unit_%2B_3%2F3_live-brightgreen)
![Tool calls](https://img.shields.io/badge/tool__calls-unlimited-purple)
![Tutorials](https://img.shields.io/badge/tutorials-EN_%2B_ID-orange)

> 🇮🇩 Versi Bahasa Indonesia: [README.id.md](README.id.md)

**Turn Muse (an AI agent) into a model inside 9Router, so Hermes can use
it exactly like a real API model — including tool calls.**

---

<div align="center">

## 🎁 DON'T HAVE MUSE YET? CLAIM A 1 BILLION TOKEN BONUS

### Referral code: `MZMT4Y`

Redeem this code in Muse Settings **within 48 hours after joining** and
we both receive **1 billion Muse tokens**.

**👉 [Read the Muse claim guide — with a VPN & without one](docs/CLAIM-MUSE.md)**

</div>

---

This document is written for **complete beginners**. It is fine if this is
your first time seeing these words — everything is explained slowly, from
zero. Read top to bottom; don't skip around on your first pass.

---

## 1. The 30-second explanation (plain words)

Imagine a restaurant:

- **Hermes** is the **waiter**. He talks to you on Telegram, and he owns
  the **hands** that do work on the computer (the VPS)
- **Muse** is a **smart cook**. He does the **thinking**. But there is a
  problem: the cook lives in **another house**, and he **has no phone** —
  there is no API you can call to make him think
- This project builds a workaround: the waiter's orders are **written as
  files** in a "notebook" (called the **bridge**), a **courier** checks
  that notebook every 5 seconds and wakes the cook, the cook writes the
  answer, and the courier carries it back

The end result: your 9Router gains a model named **`ms/muse`**. Hermes
can pick it just like ChatGPT or DeepSeek. It can chat, it can **ask for
real commands to be executed on the VPS** (that is what *tool calls*
means), and it can **see images** you send.

Why is this cool? Because the one answering is a **real agent** — he has
memory, can think deeply, and can actually work — not a raw model that
can only talk.

## 2. Why would anyone want this?

- You have a smart AI agent (Muse), but another program (Hermes) cannot
  call him because he has no API
- You want that agent to appear as **one of the model choices** in your
  9Router, usable from Telegram through Hermes
- You want the agent to **really check your machine** (is the disk full,
  is a service up, what does the log say) — not invent answers
- You want all of that **without opening a new port to the internet** and
  **without handing your house keys to strangers** — dangerous commands
  still need your approval first (explained in section 9)

## 3. Who this is for — and who it is not for

**This fits you if:**

- You already have a VPS with 9Router + Hermes running on it
- You have access to a Muse whose environment supports **hooks**
  (small scripts that run every few seconds and can wake the agent)
- You can wait roughly 20–30 seconds per round for answers that come from real
  data

**Don't use this if:**

- You need real-API speed (2–5 seconds). This system will **never** be
  that fast. Section 10 explains why
- You don't have 9Router and Hermes at all yet — set those up first,
  then come back
- You don't hold root/admin access to your own VPS

## 4. Small glossary (words you will keep meeting)

Read this first so the rest doesn't confuse you:

| Word | What it means, in human words |
|---|---|
| **VPS** | A rented computer that stays on 24/7 on the internet. Your "server" |
| **API** | The official way one program talks to another with requests and answers |
| **Model** | The "brain" that answers chats. Examples: GPT, DeepSeek, Claude — and now: `ms/muse` |
| **Provider** | A model "brand"/source inside 9Router. Ours is branded `ms` |
| **Combo** | A short nickname. The combo `muse` contains `ms/muse`, so you can call it just `muse` |
| **9Router** | The program that manages all models on your VPS — like a smart order book that knows which model each order is for |
| **Hermes** | The agent program you chat with (for example on Telegram). It owns the tools/hands |
| **Bridge** | The small go-between program. In this project: receives requests and stores them as queue files |
| **Queue** | The folder where order files wait (`pending`) and finished answers sit (`done`) |
| **Tool** | An ability Hermes has, e.g. a terminal that runs commands on the VPS |
| **Tool calls** | A letter from the model saying "please run this tool". The model doesn't run it — Hermes does |
| **SSH** | A safe way to enter another computer from far away. The courier uses it to travel to the VPS |
| **Hook / courier** | A tiny script that runs every few seconds and checks "any new orders?" |
| **Worker** | The Muse who wakes up to answer one order |
| **Prompt** | The rulebook/instructions a worker reads before working |
| **Token** | The unit of AI "thinking fuel". The more it reads/writes, the more tokens it uses |
| **Approval** | Hermes asking you first ("run this command?") before doing anything dangerous |
| **`/yolo`** | A Hermes mode that turns approval questions off — everything just runs. Dangerous if left on |

## 5. What you must have first (check one by one)

Do not start installing before all of these are ticked:

- [ ] **One Linux VPS** and you hold its **root** access
- [ ] **9Router installed and running** on that VPS (usually port
  `20128`). How to check: its dashboard opens
- [ ] **Hermes installed** on the same VPS and **already connected to
  that 9Router** (any model at first, as long as chat works)
- [ ] **One Muse** somewhere else whose environment supports
  **hooks** (a script that can run every few seconds and wake the agent).
  Don't have a Muse account yet? Follow the
  [Muse claim guide](docs/CLAIM-MUSE.md) — VPN and no-VPN routes are
  included — and redeem referral code **`MZMT4Y`** within 48 hours for
  the 1 billion token bonus
- [ ] **SSH from the Muse machine to the VPS works**, with a key, not
  a password. How to check: `ssh user@your-vps-address "echo hello"`
  succeeds from the Muse machine

If something is unticked, that is not your fault — the order simply
matters. Finish the missing piece first (9Router and Hermes have their
own tutorials), then come back here.

## 6. What is inside this repo, file by file

So the folders never scare you — here is everything and what it is for:

| File / folder | What it is, in human words |
|---|---|
| `README.md` | This document |
| `README.id.md` | This document in Indonesian |
| `docs/TUTORIAL.md` | **The complete step-by-step tutorial** — the main reading for installing |
| `docs/TUTORIAL.id.md` | The same tutorial in Indonesian, in super-simple language |
| `docs/CLAIM-MUSE.md` | **How to claim a Muse account with a VPN or without one**, including the highlighted `MZMT4Y` referral code |
| `docs/CLAIM-MUSE.id.md` | The same Muse claim guide in Indonesian |
| `docs/FASE1-SAFETY-SPEC.md` | Design/safety notes for the next phase (parallel workers, bigger queue) |
| `bridge/bridge.py` | **The bridge program that runs today (v5)** — tool calls, queue cap 20, usage estimates, chunked pseudo-streaming |
| `bridge/bridge-v1.py` | The first bridge (text-only). Kept as history and comparison |
| `worker/worker-prompt-v4.txt` | **The worker rulebook in use today** — parallel coordinator (one worker per request), tool_choice honored, unlimited tool calls, media rules, honesty rules |
| `worker/worker-prompt-v3.txt` | The previous rulebook (serial processing). Kept for rollback/comparison |
| `worker/worker-prompt-v2-backup-20261002.txt` | The previous rulebook. A backup if you ever want to roll back |
| `scripts/auto-install.sh` | **Auto-installer for Linux & macOS** — one command, the rest is guided |
| `scripts/auto-install.ps1` | **Auto-installer for Windows** |
| `scripts/install-linux.sh` | Bridge-only installer for Linux (called by the auto-installer) |
| `scripts/install-macos.sh` | Bridge-only installer for macOS |
| `scripts/install-windows.ps1` | Bridge-only installer for Windows |
| `tools/ssh-vps.sh` | SSH shortcut (wrapper) to the VPS, used by the courier |
| `tools/scp-vps.sh` | Shortcut for copying files to/from the VPS |
| `tests/test_bridge_v2.py` | Bridge unit tests (with a fake worker) — 9/9 pass |
| `tests/test_bridge_v5.py` | Bridge v5 unit tests (v4 suite + usage estimates, chunk splitting, queue cap 20) — 14/14 pass |
| `tests/test-on-vps.sh` | Live test script against the bridge on the VPS (chat, streaming, tool calls) |
| `tests/test-via-9router.py` | Test through 9Router (proof tool calls are not stripped in the middle) |
| `tests/test_prompt_v3.py` | Tests for worker rules v3: batched tool calls, image understanding, no-swap concurrency — 3/3 pass live |

## 7. How to install — pick your road

Two roads. Same destination.

### The fast road — auto-install script (recommended)

The script does the bridge part automatically (creates folders, installs
the program, registers it as an always-on service, health-checks it),
then **guides you** through the 3 short steps only a human can click
(register the provider in the 9Router dashboard, install the courier on
the Muse side, point Hermes at the model). Every value you must paste
is written out exactly in the guide — just copy-paste.

**Linux / macOS** (from this repo's folder):

```bash
bash scripts/auto-install.sh
```

**Windows** (from this repo's folder, in PowerShell):

```powershell
powershell -ExecutionPolicy Bypass -File scripts\auto-install.ps1
```

### The manual road — the full tutorial

If you want to understand every bolt (or the script errors on your
machine), follow the tutorial. It has 10 parts, from "what are we
building" to "when things break":

No Muse account yet? Start with the
**[Muse claim guide](docs/CLAIM-MUSE.md)** — it covers VPN and no-VPN
routes and highlights referral code **`MZMT4Y`**.

- 🇬🇧 **[docs/TUTORIAL.md](docs/TUTORIAL.md)** — plain English, beginner-proof
- 🇮🇩 **[docs/TUTORIAL.id.md](docs/TUTORIAL.id.md)** — Indonesian version, super-simple language

### The big steps (both roads pass through these)

1. **Install the bridge on the VPS** — queue folders + program + service
2. **Prepare the SSH key** from the Muse machine to the VPS
3. **Register the `muse` provider in 9Router** — through the
   **dashboard** (do NOT inject the database directly; we tried, and the
   result was half-broken. The tutorial explains why)
4. **Create the `muse` combo**, containing `["ms/muse"]`, plus one
   **9Router API key** for clients
5. **Install the courier + worker rulebook on the Muse side** (a hook
   long-poll, near-instant pickup + worker prompt v4 (parallel coordinator))
6. **Point Hermes** at the `muse` model in one of its profiles
7. **Run the tests one by one** — the tutorial's Part 7 has seven tests,
   each with a clear pass sign

## 8. How it works, day to day

One message from you travels like this (slowly, step by step):

1. You chat on Telegram: "check the VPS disk, using a tool"
2. Hermes forwards it to 9Router, and 9Router forwards it to the bridge
   (because the model is `ms/muse`)
3. The bridge saves it as **one file** in the `pending` folder, then the
   bridge **waits**
4. Within at most 5 seconds, the **courier** on the Muse side sees the
   new file and **wakes one worker**
5. The worker reads the file. Inside is not just your message — it is
   **the whole chat history of that session + the list of tools Hermes
   has**. That is why it is big (real sessions reach 100–220 KB per
   request). Everything is forwarded **whole, nothing is cut**
6. The worker thinks, then decides:
   - If the task needs the machine → he **does not answer in text**. He
     writes a **tool-calls letter**: "please run `df -h`"
   - That letter travels back: bridge → 9Router → Hermes
7. **Hermes executes the command for real on the VPS** (Hermes's hands).
   If the command is dangerous, Hermes **asks you first** (see section 9)
8. Hermes sends the command's result as a **new** request to the worker
   (round two)
9. The worker reads the real result and writes the final answer in human
   language
10. That answer travels back to your Telegram

**Numbers you should know:**

| Thing | The number |
|---|---|
| One round (plain chat / one tool request) | ±20–30 seconds (light requests measured ≈21 s) |
| A tool task (two rounds) | ±40–60 seconds |
| Each extra dependent tool step | +±20–30 seconds |
| Give-up limit per round | 240 seconds (4 minutes) |
| Courier checks the queue | Long-poll — a new order is spotted within ±1 second; **empty checks are free**, they burn no tokens |
| Orders allowed to wait at once | Max 20 — the 21st is politely refused with a "busy" message; just resend. Orders are processed in parallel, one worker each |
| Tokens burn | Only on real orders. Each round, the worker re-reads the whole session history — which is why this model is best used as a "specialist", not for every tiny chat |

**About mix-ups:** every order has a **unique ID** and its answer is
written to a file with the same ID. So an answer for session A **cannot
wander into session B by design** — including its tool calls. This was
live-tested with two concurrent sessions carrying different secret
codes: clean, zero crossing.

**About images & video:** images you send are **actually looked at** by
the worker before answering (tested: a plain blue image was answered
"Blue."). Video can only be **peeked at through a few sampled frames** —
the worker cannot watch a full video, and its rules require it to admit
when it cannot process something instead of pretending.

## 9. Safety — explained for beginners

Three layers of protection, strongest first:

1. **Hermes's approval door (the important one).** Before Hermes runs a
   dangerous command — deleting files, restarting services, overwriting
   system files, and friends — it **stops and asks you first** in chat.
   No answer from you within 60 seconds = the command **does not run**.
   This is called `manual` mode, and on our setup it is verified active
   on all profiles.
   - ⚠️ **`/yolo`** mode turns this door off — every command just runs,
     no questions. Use it only when you fully mean it and are watching.
     **Never leave `/yolo` on.**
2. **The worker's rules.** In its rulebook, the worker may only propose
   state-changing commands **when your task asks for exactly that**, and
   it is forbidden from inventing data (claims require real evidence).
3. **The bridge only listens on localhost.** The bridge program on the
   VPS can only be reached from that same machine (127.0.0.1); it is
   **not open to the internet**. Only 9Router on the same machine can
   reach it.

And about secrets: **API keys and tokens NEVER enter this repo.** In the
real system, all secrets live in separate `0600` files (readable only by
their owner) on each machine. Host-specific values in this repo's sample
files (VPS address, domains) have been replaced with placeholders like
`YOUR_VPS_IP` — replace them with your own values when you set it up.

## 10. Honest limits (so nobody is surprised)

This project is not perfect and does not claim to be:

- **It will never be as fast as a real API.** A real API is a raw model
  standing by in a datacenter (2–5 seconds). Here, the one answering is
  an agent who must wake up and read context first. The realistic target
  is "90% of the API feel": complete protocol, stable, fast enough —
  just not the fastest
- **The queue is processed one at a time.** Five orders may wait
  together, but the worker handles them in order. Many active sessions
  at once = the later ones wait longer. (A parallel version is the
  planned next phase — notes are in `docs/FASE1-SAFETY-SPEC.md`)
- **Long sessions get heavier.** Because every round re-reads the whole
  history, a very long session gets slower and more token-hungry.
  Starting a fresh session now and then is healthy
- **Video is limited** to frame peeking, as explained in section 8
- **It depends on the Muse machine.** If the machine the agent lives on
  is down, the `ms/muse` model cannot answer until it recovers. Your
  other API models in 9Router are **not affected**

## 11. Beginner FAQ

**"Is this free?"**
The project is free and open-source (fully yours). What may cost money
is the ingredients: renting the VPS and your Muse usage — this
system itself adds no extra fee.

**"I can't code. Can I use this?"**
Yes, if you can copy-paste commands and read slowly. Use the fast road
(section 7); the tutorial was written for you.

**"Can this break my VPS?"**
Dangerous commands stop at Hermes's approval door (section 9) as long
as the mode is `manual` and `/yolo` is off. The biggest real risk is a
human tapping "approve" without reading, or leaving `/yolo` on.

**"Why is it slow?"**
Because of the file courier and an agent who must wake up per order
(section 8). Under 2 minutes is normal. Past 4 minutes means stuck.

**"Will my answer get mixed up with another session?"**
No. The ID-label system makes that impossible by design, and it was
live-tested. See section 8.

**"I don't have Hermes. Can I still use it?"**
The bridge itself works with any OpenAI-compatible client that supports
tool calls. But this tutorial is written from the Hermes angle. Other
clients may need small adjustments on the courier/prompt side.

**"Are images I send really read?"**
Yes — extracted and looked at by the worker before answering. Tested
live.

**"What if something errors halfway?"**
Stay calm. Every big change in this system follows the same pattern:
**the old file is backed up first** (`.bak`), so rolling back takes a
minute. For error symptoms, the tutorial's Part 9 has a troubleshooting
table — symptom, likely cause, fix.

**"Is my data safe?"**
Your chats travel through your own machines (your VPS + your agent).
This repo contains no chat data, no secrets, and no real addresses of
the running deployment.

## 12. Short history (proof this comes from real use)

- **2026-10-01** — Version one live: the bridge could only answer text
- **2026-10-02 (morning)** — A worker flip-flopped inside one chat
  (refused → claimed checks with partly invented results → denied
  everything). Root causes found: a fresh worker per request + an
  ambiguous rulebook. The **honesty rules** were born here
- **2026-10-02 (midday, "Option A")** — Bridge v4: a real **tool-calls
  loop** working end-to-end from Telegram (Hermes executed `df -h`
  itself; results matched a manual check)
- **2026-10-02 (afternoon, prompt v3)** — The tool-call count limit was
  **removed** (unlimited, with batching discipline) and the media rule
  was added. Live tests 3/3 passed: 6 commands batched in one answer,
  an image understood correctly, two concurrent sessions with zero
  crossing
- **2026-10-07 (Fase 1+2, bridge v5 + prompt v4)** — Requests are now
  processed **in parallel** (a coordinator spawns one worker per
  request, max 8 at once; queue cap 5 → 20), `tool_choice` is honored,
  usage fields carry honest estimates, streamed answers arrive in
  small chunks, and the courier became a **long-poll** (pickup ≈1 s).
  Live tests: 8 parallel requests all answered correctly in 39 s total
  with zero mix-ups; `tool_choice` required and none both obeyed

Every number in this document (delays, request sizes, test results) is
a real number from the running system's logs — not marketing invention.

## 13. Closing

A personal infrastructure project, shared as-is for anyone who wants to
learn from it or use it. If it helps you, a ⭐ on this repo means a lot
to its maker. If something errors or the tutorial confuses you halfway,
open an *issue* — tell us which part you got stuck on.

Take it slowly. Nothing here is a race.
