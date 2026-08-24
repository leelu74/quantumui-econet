# Problem log

Everything that stopped the platform working, in the order it was found, with
how it was diagnosed and what fixed it. Grouped by who owns the problem, so the
first section is the only one worth taking to QpiAI — the rest were ours.

Written 2026-08-24.

---

## 1. QpiAI SDK — worth raising upstream

All verified against `qpiai-quantum 0.2.0` on the local statevector simulator.

### 1.1 Gate arguments are qubit-first, unlike Qiskit and Cirq

```python
c.ry(0, math.pi)          # works   — ry(qubit, angle)
c.ry(math.pi, 0)          # fails   — reads the angle as a qubit index
c.cp(0, 1, math.pi)       # works   — cp(control, target, angle)
```

Every other mainstream SDK puts the angle first. The failure is silent when the
angle happens to land inside the qubit range, and otherwise surfaces as:

```
CircuitError: Qubit index 3.141592653589793 out of range. Valid range: 0-0
```

**Impact here:** the lab material is adapted from QWorld notebooks written
against Qiskit and Cirq. Copied code reads an angle as a qubit index, so a
learner pasting a published solution gets an error about qubit indices for code
that contains no obvious index error. It also caught us out — a first pass at
authoring solutions got the order wrong and only the verification harness
found it.

**Ask:** is the ordering intentional and stable? A keyword form
(`ry(qubit=0, theta=pi)`) or accepting both orders would remove a whole class
of confusion for anyone porting Qiskit material.

### 1.2 `QuantumRegister` is not subscriptable

```python
q = QuantumRegister(8)
qc.x(q[i])                # TypeError: 'QuantumRegister' object is not subscriptable
```

Qiskit's register objects index into individual qubits, and QWorld's material
relies on it heavily. Here the workaround is plain integer indices, which is
fine — but the exported name `QuantumRegister` sets an expectation the object
does not meet.

**Ask:** either support indexing, or consider not exporting a name that
collides with Qiskit's.

### 1.3 Bell basis labels are transposed

Asking for `|Φ+>` builds `(|01>+|10>)/√2`, which is textbook `|Ψ+>`. And vice
versa. Verified across all four Bell states.

We correct it at the boundary (`catalog.py`, `_SDK_BELL_LABEL`) rather than
teach learners the wrong labels — but a platform that teaches the Bell basis
cannot leave this to chance.

**Ask:** confirm whether this is fixed in a later release, so the workaround
can be dropped rather than silently double-correcting.

### 1.4 Bit ordering is not self-consistent between algorithms

Grover reports counts reversed relative to the target you searched for — ask
for `110`, get `011`. Bernstein–Vazirani and Simon report in the same order as
their input.

Because it varies per algorithm, we carry a per-entry `reverse_bits` flag
rather than one global setting.

**Ask:** is there a documented convention? A single consistent ordering, or a
documented per-algorithm one, would let us delete the flag.

### 1.5 Local execution is the only usable tier without a key

Four of five advertised backends need `QPIAI_API_KEY`:

| Backend | Kind | Available |
| --- | --- | --- |
| `QpiAI-QSV-Local` | statevector | yes |
| `QpiAI-QSV-Simulator` | statevector, cloud | no |
| `QpiAI-QDM-Simulator` | density matrix | no |
| `QpiAI-QTN-Simulator` | tensor network | no |
| `QpiAI-Indus-1` | physical QPU | no |

Notably there is no local **density-matrix** option, so noise and mixed states
cannot be taught at all without cloud access.

**Asks:** licensing for unlimited local execution in an educational
deployment; limits of the local simulator (qubits, shots, memory); whether
density-matrix simulation can run locally; pricing and quota for the cloud
tiers and Indus-1; and whether a classroom/shared key model exists, since our
key is currently one process-wide variable that would unlock cloud access for
every visitor at once.

---

## 2. Integration and deployment

Ours, not QpiAI's — recorded because each one cost real time.

| Problem | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| Executor unreachable from the app | every circuit "temporarily unavailable" | `QUANTUM_EXECUTOR_URL` unset, so the app called `localhost:8080` inside its own serverless container | point it at the Railway URL |
| Executor rejected every call | `/health` green, real calls 401 | `EXECUTOR_API_KEY` differed between Railway and Vercel | align to Railway's existing key |
| CORS pointed at a dead URL | — | `CORS_ORIGINS` still named an old preview deployment | set to the live domains |
| Production env vars empty | login impossible, no DB | `DATABASE_URL`, `DIRECT_URL`, `AUTH_SECRET`, `EXECUTOR_API_KEY` were all set to `""` | provision Neon, set real values |
| Site unreachable | every URL 307'd to a hostname with no DNS | `www.sroobservotary.com` was the production domain but had no A record | DNS at the registrar |
| Vercel token rejected | `Could not retrieve Project Settings`, then `User not found (404)` | wrong token **type** — a `vcp_` project token, not an account token from `/account/tokens` | create the right kind |
| Railway token rejected | `Invalid RAILWAY_TOKEN` while the secret was plainly set | two token kinds read from different variables: `RAILWAY_TOKEN` wants a **project** token, `RAILWAY_API_TOKEN` an **account** one | project token; workflow now accepts either |
| Deploys silently had no effect | new deployment `Ready` but not serving | a prior `vercel rollback` **pins** production; a later `vercel deploy --prod` does not override the pin | `vercel promote` |
| Two unrelated git histories | — | `leelu74/QuantumUI`'s `main` shared no ancestor with the deployed code and was a stale MySQL-era app | moved to `econetvision/quantumUI`, whose `main` is a direct ancestor |

---

## 3. Application bugs found and fixed

| Problem | Impact | Root cause |
| --- | --- | --- |
| Signed-in users bounced to `/login` | nobody could stay logged in | over HTTPS the cookie is `__Secure-authjs.session-token`; `getToken` only looks for the prefixed name when told to, and its default keys off `AUTH_URL`, which this app deliberately does not set |
| Login loop after correct credentials | sign-in looked broken | `router.push` replayed a redirect the App Router had cached from a signed-out prefetch, without asking the server. `router.refresh()` ran after the push had already used the stale entry |
| `import math` failed at runtime | 22 of 94 shipped solutions died on their first line | `ALLOWED_MODULES` advertised math/numpy/random, but the restricted builtins table had no `__import__` at all — the allowlist was decorative |
| Lesson text invisible | the whole curriculum unreadable in the default theme; on mobile a lesson scrolled as a blank white screen | 1,439 hardcoded dark-theme colours across 12 lesson files, written before the light theme existed |
| 63 questions had no solution | — | QWorld solution notebooks mark tasks with an HTML anchor, not a `Task N` heading; the harvester matched only the heading and found nothing in 51 of 170 notebooks |
| Streak never advanced | learners lost their streak despite practising | credited only successful runs, used UTC dates (an IST evening fell on the previous day), and never read the server value back — a second browser showed 0 over a real record |
| Google sign-in `invalid_client` | a button that could only ever fail | the provider was registered unconditionally, so with no credentials NextAuth still advertised `google` and sent an empty `client_id` |
| Developer copy in production | visitors told to run `cd quantum-executor && ./run.sh` | written for a local checkout, shown to everyone |

---

## 4. The one that matters most

**A green pipeline deployed a broken production and the smoke test certified
it.**

`generator client` declared no `binaryTargets`, so `prisma generate` emits an
engine only for the platform that ran it. Vercel building its own deployments
always got that right. `vercel build` on `ubuntu-latest` followed by
`vercel deploy --prebuilt` shipped a **debian** engine to a runtime that is
**Amazon Linux** — Prisma could not start and every query failed.

The deploy went green. Static pages served, the executor answered, `/tracks`
still redirected. Sign-in returned `error=Configuration` and registration
returned `DB_UNAVAILABLE`, for roughly forty minutes, with three real user
accounts on the site.

The smoke test passed throughout, because **nothing it asserted touched the
database**: `/` and `/login` are static, `/tracks` 307s from a cookie check the
proxy makes without a database, and the executor is a separate service. So the
rollback it exists to trigger never fired.

Fixed by adding `rhel-openssl-3.0.x` to `binaryTargets`, and — the more
important half — by making the smoke test register an address that already
exists: a read with no side effect, 409 when the database answers and 503 when
it does not. That single assertion would have caught it and rolled back
automatically.

**The lesson:** a health check that only exercises what cannot fail is worse
than none, because it converts an outage into a green tick.

---

## 5. Where things stand

- 152 lab questions, 108 with solutions, **all 108 verified to execute** against
  the live backend
- 44 questions still have no solution — all 8 of `quantum-error-correction`
- 12 finished labs (`qpiai-sdk` ×10, `cirq-sdk` ×2) that no track points at
- both deploy pipelines have completed successfully end to end, smoke test
  including the database check
