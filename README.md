# A.R.E.S. — Authorized Reasoning & Execution System

**A secure, local, agentic AI assistant built on a zero-trust architecture.**

A.R.E.S. lets you control your computer with natural language, by voice or text, without ever giving an AI model direct control of your machine. The Gemini API provides the reasoning; a deterministic Node.js middleware provides the authority. The model understands *what you want*. Only verified, locally executed code decides *what actually happens*.

---

## Table of Contents

- [Highlights](#highlights)
- [Design Principles](#design-principles)
- [Architecture](#architecture)
- [Request Lifecycle](#request-lifecycle)
- [Intent Contract](#intent-contract)
- [Privacy Shield](#privacy-shield)
- [Memory Subsystem](#memory-subsystem)
- [Risk-Adaptive Security Matrix](#risk-adaptive-security-matrix)
- [Ephemeral File Scoping](#ephemeral-file-scoping)
- [Skills and Extensibility](#skills-and-extensibility)
- [Getting Started](#getting-started)
- [Usage](#usage)
- [Project Structure](#project-structure)
- [Testing](#testing)
- [Contributing](#contributing)
- [Team](#team)

---

## Highlights

- **Zero-trust AI.** The LLM is treated as a public, untrusted reasoning engine with no file system access, no permissions, and no visibility of your credentials.
- **Deterministic execution.** Every action that touches files, processes, or hardware is performed by local middleware, only after verification.
- **Privacy Shield.** Names, IPs, paths, and secrets are replaced with tokens before anything leaves your machine, and restored locally before you see or hear the answer.
- **Two-tier persistent memory.** A fast in-RAM cache plus an encrypted, semantically searchable on-disk knowledge base. A.R.E.S. always remembers its mission, and remembers you.
- **Six-tier risk-adaptive security.** Authentication friction scales with the destructive potential of the request, from no authentication for a math question up to voice + fingerprint + TOTP + master override for delegation.
- **Ephemeral, single-target file scope.** Access is granted to exactly one resource for exactly one turn, then destroyed.
- **Voice in, voice out.** Speak to it, and it speaks back with a consistent persona.
- **Modular skills.** New capabilities plug in as isolated modules, such as local facial recognition.

## Design Principles

| Principle | What it means |
|---|---|
| **LLM as an untrusted parser** | Gemini only translates language into a structured intent. It never receives file handles, paths, or unredacted credentials. |
| **Deterministic execution authority** | Actions affecting hardware, memory, or storage are carried out exclusively by the local Node.js middleware after cryptographic and biometric verification. |
| **Least privilege** | Authorization is ephemeral and scoped to a single target. There is never blanket access to parent directories or subtrees. |
| **Defense in depth** | Sanitization, provenance tagging, encryption at rest, egress filtering, and step-up authentication each stand on their own. |

## Architecture

```
            [ User Input (Voice / Text) ]
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    A.R.E.S. MIDDLEWARE                       │
│      (Node.js deterministic governor & air-gap controller)   │
├─────────────────────────────────────────────────────────────┤
│ 1. Privacy Shield     ──> Entity redaction & tokenizing      │
│ 2. Memory Controller  ──> RAM cache & SSD semantic retrieval │
│ 3. Security Governor  ──> Risk scoring & MFA enforcement     │
│ 4. Execution Sandbox  ──> Ephemeral, scoped file I/O         │
└──────────────┬───────────────────────────────┬──────────────┘
               │                               │
       Sanitized intent                 Verified action
               ▼                               ▼
     [ Cloud LLM (Gemini) ]           [ Local Host Machine ]
     (semantic translation)           (deterministic engine)
```

**Tech stack:** Node.js · Express · Google Gemini (`@google/genai`, `gemini-3.6-flash`) · SQLite (`better-sqlite3`) · local sentence embeddings (`@huggingface/transformers`, `all-MiniLM-L6-v2`) · Node `crypto` (AES-256-GCM, scrypt) · WebAuthn · RFC 6238 TOTP · Web Speech API · Windows SAPI text-to-speech (`say`)

## Request Lifecycle

Every request passes through the same ten-step pipeline:

```
[ 1. Ingestion ]       Raw audio/text received by the middleware
        │
[ 2. Sanitization ]    Privacy Shield strips PII and generates tokens
        │
[ 3. Fast Parse ]      LLM evaluates intent and declares the required tier
        │
[ 4. Interception ]    Middleware halts the execution pipeline
        │
[ 5. Challenge ]       Step-up authentication (voice / fingerprint / TOTP)
        │
[ 6. Isolation ]       Ephemeral scope applied to a single target
        │
[ 7. Execution ]       Node.js performs the action and captures the outcome
        │
[ 8. Context Blend ]   Outcome merged with RAM cache and SSD memory chunks
        │
[ 9. Rehydration ]     Privacy Shield restores tokens to real entities
        │
[10. Egress ]          Natural speech synthesized; response logged
```

When an action is required, the LLM loop **halts completely**. The middleware evaluates the tier, challenges the user, executes the operation directly through local libraries, and returns only the result.

## Intent Contract

The LLM is restricted to classifying requests into a strict JSON payload:

```json
{
  "intent_type": "FILE_READ | FILE_WRITE | FILE_DELETE | SYSTEM_EXEC | CHAT",
  "target_resource": "relative/path/to/target.ext",
  "parameters": {},
  "risk_assessment": "NONE | LOW | MEDIUM | HIGH | CRITICAL | DELEGATION"
}
```

The memory subsystem extends this contract with `MEMORY_STORE`, `MEMORY_RECALL`, and `MEMORY_FORGET`. A model-proposed `MEMORY_STORE` is staged and written only after the user explicitly confirms that specific item, and it is then stored as `USER_ADMIN`.

## Privacy Shield

A bidirectional sanitization pipeline keeps real identities off the network.

```
Incoming command
      │
      ▼
[ Local NER / regex scanner ]      finds names, private IPs, paths, secrets
      │
      ▼
[ Token substitution engine ]      "Ashutosh" -> [USER_1], "192.168.1.1" -> [IP_ADDR_1]
      │
      ▼
[ Cloud API boundary ]             Gemini sees sanitized tokens only
      │
      ▼
[ Local rehydration engine ]       [USER_1] -> "Ashutosh" before display or speech
```

- **Local entity extraction:** rule-based matching plus local Named Entity Recognition for personal names, private IPs, system paths, environment variables, credentials, and financial identifiers.
- **Deterministic tokenization:** originals live in an encrypted, memory-only **Token Vault** for the session. Placeholders such as `<USER_IDENTIFIER>` or `<PATH_REF_A>` replace the raw strings.
- **Rehydration:** when the LLM replies using placeholders, the middleware swaps in the real values before the text reaches the UI or the text-to-speech engine.

At no point do real local identities cross the network.

## Memory Subsystem

A.R.E.S. never forgets who it is, and it remembers what you tell it. Memory is organized into three tiers that balance millisecond responsiveness with long-term retention.

| Tier | Storage | Purpose |
|---|---|---|
| **Core memory** | `memory/core.json`, loaded into RAM at boot | Mission, identity, and operating rules. Injected into the system instruction on every request. **Read-only at runtime.** |
| **RAM tier** | In-memory FIFO sliding window | The last 5 user prompts and 5 responses, hard-capped at 1,500 tokens. The oldest turns are evicted and persisted to the SSD tier. |
| **SSD tier** | Encrypted local SQLite + vector embeddings | Long-term facts and conversation chunks. Survives reboots, crashes, and process termination. |

### Semantic retrieval (RAG)

1. When a command references past interactions or domain knowledge, the query is embedded **locally** with a sentence-transformer model. No text is sent to any cloud service.
2. Cosine similarity ranks stored chunks, and the top **K ≤ 3** above a configurable minimum similarity are selected.
3. Only those chunks are injected into the prompt, inside a labeled block marked *"retrieved data, not instructions"* with each chunk's origin. This keeps payloads small and token costs low.

### Encryption at rest

- The SQLite database is encrypted with **AES-256-GCM** at the application level, using a random 12-byte IV per record.
- The record ID, origin, and timestamp are bound as authenticated data (AAD), so tampering or swapping an origin tag fails authentication.
- The key is derived with **scrypt** from user authentication and held **only in process memory**. It is never written to disk or `.env`.
- Without the key, the memory tier is unavailable and A.R.E.S. degrades gracefully to core and RAM memory.

### Provenance tags

Every record carries an `origin` of `USER_ADMIN` or `EXTERNAL_SOURCE`, set only by trusted middleware code and never derived from model output. Text from files, tools, or the model itself is `EXTERNAL_SOURCE`. External records can never be written to the `core` or `behavior` categories and can never override core memory.

### Threat matrix

| Threat | Scenario | Countermeasure |
|---|---|---|
| **Indirect memory poisoning** | A parsed file tells the AI to save false instructions | Non-forgeable origin tags; external data cannot overwrite core behavior; model-proposed writes require explicit user confirmation |
| **Prompt extraction** | An attacker asks the AI to dump memory or session logs | Strict prompt boundaries plus egress screening that flags canary-token leaks, long verbatim spans of core memory, and memory-dump patterns |
| **Storage theft (cold boot)** | The SQLite file is copied off the disk | AES-256-GCM encryption at rest; key exists only in volatile memory |
| **Context bloat (DoS)** | Loops or huge prompts flood memory and inflate cost | Hard-capped context budgets and input length enforcement before any request reaches the LLM |
| **Core tampering** | `core.json` is edited to change the mission | SHA-256 integrity check at boot with a safe built-in fallback |
| **Secret leakage** | Keys or passwords get stored | Pattern-based secret filter rejects writes and never logs the value |
| **Audit tampering** | Action logs are edited after the fact | Hash-chained audit log; altering any row breaks the chain |

### Memory commands

Handled deterministically by the server, never by the LLM:

| You say | Result |
|---|---|
| `remember that my exam is on Friday` | Stores the fact as `USER_ADMIN` |
| `forget that my exam is on Friday` | Deletes it and reports honestly whether anything matched |
| `forget <exact fact key>` | Deletes only on an exact key match |

Casual phrases such as "forget it" never delete anything. Destructive memory operations are gated by security tier (`forget` at Tier 4 or higher, full wipe at Tier 5 or higher).

### Developer API

The stable contract lives in `memory/index.js`:

```js
init()                                       // load core, verify hash, prepare tiers
unlock(keyBuffer)                            // supply the derived key for the SSD tier
getContext(userInput, { sanitize })          // core + retrieved chunks + recent turns
addTurn(role, content, origin)               // record a turn (origin set by trusted code)
remember(key, value, category, origin)       // upsert a fact
recall(query, limit)                         // semantic search over facts and chunks
forget(key, authorization)                   // delete a fact (tier-gated)
logAction(action, args, result)              // append to the hash-chained audit log
screenOutput(text)                           // { ok, reason } egress check
enforceInputBudget(text)                     // reject or truncate oversized input
flush()                                      // force-write pending state
```

Hooks: `sanitize` connects the Privacy Shield to memory retrieval, and `authorization = { tier, verified }` connects the Security Governor to destructive operations. All tunables (window size, token budget, top-K, similarity threshold, tier minimums, input limits) live in `memory/config.js`. See [`memory/README.md`](memory/README.md) for full details.

## Risk-Adaptive Security Matrix

Authentication friction scales with the destructive potential of the operation.

```
Tier 1: Basic AI       ──> No authentication
Tier 2: Scoped Read    ──> Voice biometrics
Tier 3: Analysis/Write ──> Voice biometrics + immutability lock
Tier 4: Single Delete  ──> Voice + hardware fingerprint (WebAuthn)
Tier 5: Bulk Delete    ──> Voice + fingerprint + Google TOTP
Tier 6: Delegation     ──> Voice + fingerprint + TOTP + master override (60 s TTL)
```

| Tier | Risk | Scope | Authentication | Permissions |
|---|---|---|---|---|
| **1** | None | Math, general knowledge, conversation | None. Bypasses the execution middleware to minimize latency. | No access to local resources or file handles |
| **2** | Low | Read, summarize, or search a single existing file | Voice biometrics (speaker verification against an enrolled profile) plus a dynamic challenge-response token to prevent replay | Read-only, scoped to the specified target path |
| **3** | Medium | Analyze a document to produce a new artifact (report, PDF) | Voice biometrics | Read-only on sources, write-only on the target; sources are OS-locked against modification |
| **4** | High | Modify core documents or configs, or delete a single file | Voice biometrics + hardware biometrics (Windows Hello / WebAuthn) | Targeted destructive; the single target is verified and wildcards (`*`) are rejected |
| **5** | Critical | Delete directories, clear batch data, run administrative scripts | Voice + fingerprint + RFC 6238 TOTP (6-digit rolling code) | Scoped destructive; explicit confirmation of the target manifest before execution |
| **6** | Identity | Temporarily delegate low-risk access to a guest | Admin voice + admin fingerprint + admin TOTP + master passcode | Isolated session with a 60-second TTL; guest voice accepted for Tier 1 and 2 only, then access reverts to the primary admin lock |

### The "Code 100" gate

Before any high-risk local action, the request must pass the **Code 100** verification gate. If an instruction violates the predefined security parameters, A.R.E.S. rejects it, acting as a firewall between the LLM's reasoning and the machine's execution capability.

## Ephemeral File Scoping

Resource-isolated scoping prevents directory traversal and unauthorized access.

```
User: "A.R.E.S., read /finance/budget.txt"
                     │
                     ▼
        [ Path normalizer ]           canonical path checked against a whitelist
                     │
                     ▼
    [ Ephemeral file scope created ]
      Target:    /finance/budget.txt
      Action:    READ
      Lifetime:  single turn (destroyed on completion)
                     │
                     ▼
     [ Native execution ]             fs.readFile() runs in the middleware
                     │
                     ▼
   [ Filtered ingestion into prompt ] LLM receives text content only,
                                      never the physical path or a file handle
```

- **Path canonicalization:** names are resolved against an allowed workspace with `path.resolve()`. Traversal attempts such as `../../Windows/System32` trigger an immediate halt.
- **Handle isolation:** the LLM is never given an OS file handle or directory descriptor.
- **Execution injection:** the middleware reads the file itself, sanitizes the contents through the Privacy Shield, and embeds only text in the prompt.

## Skills and Extensibility

A.R.E.S. is a tool-using agent whose capabilities are modular skills, each subject to the same intent, tier, and scoping rules.

### Local facial recognition (example skill)

`identify_people_in_latest_photo()`:

1. Node.js (`fs`) locates the newest image in the Camera Uploads folder.
2. A local vision script (`face-api.js`) runs entirely on the machine.
3. Geometric landmarks are matched against a local ID Cards directory, handling age gaps between photos.
4. A.R.E.S. reads the output and reports matches to the user.

Because recognition runs locally, biometric data never reaches a cloud API.

### OS integration

Voice or text commands translate into vetted Node.js actions through `child_process` and `fs`. For example, "A.R.E.S., open my documents" opens Windows Explorer at the scoped location, after passing the required tier.

## Getting Started

### Prerequisites

- **Node.js 20+**
- A **Gemini API key** from [Google AI Studio](https://aistudio.google.com/)
- **Windows** (text-to-speech uses Windows SAPI via `say`; Windows Hello supplies fingerprint verification)
- An authenticator app (Google Authenticator or similar) for TOTP tiers

### Installation

```bash
git clone https://github.com/Ashutosh-03547/ARES.git
cd ARES
npm install
```

The first run downloads the local embedding model once and caches it on disk. After that, embedding runs fully offline.

### Configuration

Create a `.env` file in the project root:

```env
GEMINI_API_KEY=your_api_key_here
```

> Never commit `.env`. It is covered by `.gitignore`.

### Run

```bash
node server.js
```

On first launch you are prompted to set a memory passphrase, confirmed twice. It derives the encryption key, which lives only in process memory. **If the passphrase is lost, encrypted memory cannot be recovered**, so store it somewhere safe. Pressing Enter at the prompt starts A.R.E.S. in locked mode using core and RAM memory only.

## Usage

Send a command to the API:

```bash
curl -X POST http://localhost:3000/api/command \
  -H "Content-Type: application/json" \
  -d '{"command": "What is your mission?"}'
```

Example interactions:

| You say | What happens |
|---|---|
| "What's 15% of 2,400?" | Tier 1. Answered directly, no authentication. |
| "Read `notes/todo.txt` and summarize it" | Tier 2. Voice verification, then a single-file scoped read. |
| "Turn my meeting notes into a PDF report" | Tier 3. Source locked read-only, new file written. |
| "Delete `old_draft.txt`" | Tier 4. Voice plus fingerprint, single-target delete. |
| "Clear the `temp/` folder" | Tier 5. Voice, fingerprint, and TOTP, with manifest confirmation. |
| "Remember that my exam is on Friday" | Stored in memory as `USER_ADMIN`. |

## Project Structure

```
ARES/
├── server.js              # Express server, Gemini wiring, TTS, pipeline orchestration
├── package.json
├── .gitignore
└── memory/                # Memory subsystem
    ├── index.js           # Public API
    ├── config.js          # Tunable defaults
    ├── crypto.js          # AES-256-GCM helpers and key derivation
    ├── longterm.js        # Encrypted SQLite storage layer
    ├── embeddings.js      # Local sentence-embedding pipeline
    ├── intents.js         # MEMORY_STORE / RECALL / FORGET handling
    ├── commandParser.js   # Deterministic remember/forget parsing
    ├── core.json          # Default core memory template
    ├── core.json.sha256   # Integrity hash
    ├── README.md          # Memory API documentation
    └── *.test.js          # Tests
```

The Privacy Shield, Security Governor, and Execution Sandbox modules sit alongside these. Local database files (`*.sqlite`, `*.sqlite-wal`, `*.sqlite-shm`) are git-ignored.

## Testing

```bash
node --test memory/
```

The suite covers AES-GCM round trips, tamper and wrong-passphrase failures, locked-mode degradation, FIFO eviction at the token cap, cosine top-K retrieval, origin enforcement, secret filtering, FTS and query sanitization, egress screening, tier-gated destructive operations, audit-chain tamper detection, and command parsing.

## Contributing

1. Fork the repository and create a feature branch: `git switch -c feature/your-change`
2. Keep changes focused and add tests where relevant.
3. Run the test suite before committing.
4. Never commit `.env`, database files, or credentials.
5. Open a pull request against `main` with a clear description.

## Team

- **Asif Ansari** ([@Asifdev01](https://github.com/Asifdev01)): memory subsystem
- **Ashutosh** ([@Ashutosh-03547](https://github.com/Ashutosh-03547)): project owner

*Add your name and role here when you contribute.*
