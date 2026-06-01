# Apache OpenDAL oli (CLI) — Threat Model (v1 draft)

**Status:** v1 draft, authored by the ASF Security Team for OpenDAL PMC
review. Drafted against `apache/opendal-oli` (default branch `main`),
2026-06-01.

**Provenance legend:** *(documented)* — repo docs/README; *(maintainer)* —
ratified by an OpenDAL PMC member; *(inferred)* — reasoned from structure /
domain norms, **not yet confirmed** (each has a matching §14 item).

**Draft confidence:** documented ~8 · maintainer 0 · inferred ~11.

**Relationship to the core model:** `oli` is a command-line front end over the
OpenDAL core. The data-access trust boundaries (OpenDAL→backend, credentials,
paths, hostile-backend handling) are modelled in **`apache/opendal`'s
`THREAT_MODEL.md`** (PR apache/opendal#7641); this document covers only the
CLI-specific surface and defers the rest.

**Revision triggers:** a new subcommand that changes the data/credential flow;
a change to the config/profile format or how credentials are sourced; a change
that lets `oli` be driven by a less-trusted caller.

---

## §1 Header

- **Project:** `apache/opendal-oli` — `oli`, the OpenDAL Command Line
  Interface: "a unified and user-friendly way to manipulate data stored in
  various storage service" via subcommands `ls`, `cat`, `stat`, `cp`, `rm`,
  `bench`. *(documented — README)*
- **Scope:** this repository only — the CLI (`src/`), its config/profile
  loading, and argument handling. The OpenDAL core it links is out of scope
  (covered by `apache/opendal`#7641). *(documented)*

## §2 Scope and intended use

`oli` is run **interactively or in scripts by an operator** who already holds
the backend credentials, to move/inspect data across OpenDAL backends.
*(inferred — Q1)* It is a thin CLI veneer: the storage logic is the core's;
`oli` adds command parsing, a config/profile mechanism for naming backends +
credentials, and local-filesystem interaction for `cp` to/from local paths.
*(inferred — Q2)*

### Component families

| Family | Role / trust notes |
|---|---|
| Subcommands (`ls`/`cat`/`stat`/`cp`/`rm`/`bench`) | Map CLI args to core `Operator` calls. *(documented — README)* |
| Config / profile loader | Resolves named backends + their credentials from a config file (and/or env). The credential-bearing surface specific to `oli`. *(inferred — Q3)* |
| Core dependency (`opendal`) | The actual data access; out of scope here. *(documented)* |

## §3 Out of scope (explicit non-goals)

- **The data-access threat model** — credentials-in-transit, per-backend auth,
  signed URLs, hostile-backend handling, path traversal at the backend: all in
  `apache/opendal`#7641. *(inferred — Q3)*
- **Build/release/supply-chain** (signing, reproducible builds). Per rubric §1.
  *(documented — rubric)*
- `tests/`, `dev/`, `scripts/`, `skills/` — not shipped as the product.
  *(inferred — Q3)*

## §4 Trust boundaries

1. **Operator → `oli` process.** The operator invoking `oli` is **trusted** —
   they hold the credentials and choose the commands (analogous to the core's
   in-process caller). *(inferred — Q1)*
2. **`oli` → config/profile + credential source.** A local file (and/or env)
   the operator controls; holds backend definitions and secrets. *(inferred — Q3)*
3. **`oli` → backend (network) and `oli` ↔ local filesystem.** The network hop
   defers to the core model; local-FS interaction (`cp` to/from local paths)
   uses OS path semantics + the invoking user's permissions. *(inferred — Q4)*

## §5 Assumptions about the environment

Rust binary on the operator's host; the invoking OS user's filesystem
permissions bound local reads/writes; system trust store for backend TLS.
*(inferred — Q5)*

## §6 Inputs — per-parameter trust

| Input | Source | Trusted? | Notes |
|---|---|---|---|
| CLI args (source/dest URIs, paths, flags) | Operator (or a wrapping script) | Trusted-by-default; untrusted if `oli` is script-driven on attacker input | URI/path parsing + which backend a URI resolves to. *(inferred — Q6)* |
| Config / profile file (backends + credentials) | Operator | Trusted (secret) | File permissions + not echoing secrets are the CLI-specific concerns. *(inferred — Q3,Q7)* |
| Object data | Backend / local file | Pass-through | Same as core. *(inferred — defer)* |

## §7 Adversary model

- **Out of scope:** the operator running `oli` (trusted, holds the creds).
  *(inferred — Q1)*
- **In scope (defer to core #7641 §7):** whoever controls the data/remote, a
  network MITM, a malicious backend.
- **Conditional (CLI-specific):** if `oli` is invoked by a *less-trusted*
  automation layer with attacker-influenced args, arg/URI handling becomes a
  boundary. *(inferred — Q6)*

## §8 / §9 Properties provided / not provided

`oli` provides **no data-access security property of its own** beyond the
core's (§8/§9 of #7641). CLI-specific points pending confirmation:
- It should **not** print raw credentials in normal or verbose/debug output.
  *(inferred — Q7)* — violation symptom: a secret appears in stdout/stderr/
  logs; severity: high.
- It provides **no sandboxing** of the operations an authorized operator runs
  (`rm` deletes, `cp` overwrites — by design). *(inferred — Q8)*

## §10 Downstream (operator) responsibilities

- Protect the config/profile + credential file (permissions, not committing
  it). *(inferred — Q3)*
- Don't drive `oli` from untrusted input without validating args/URIs.
  *(inferred — Q6)*
- Same backend responsibilities as the core (#7641 §10): endpoint trust,
  TLS, path/size discipline. *(inferred — defer)*

## §11 Known misuse / §11a known non-findings (seed — Q18a)

- Misuse: scripting `oli rm`/`cp` over attacker-controlled URIs/paths without
  validation. *(inferred — Q6)*
- Non-finding: "`oli` reads credentials from a config file / env" — that is the
  operator's secret store, by design, not a vulnerability. *(inferred)*
- Non-finding: "`oli rm` deletes data" — authorized destructive ops are the
  tool's purpose. *(inferred — Q8)*

## §13 Triage dispositions

| Disposition | When |
|---|---|
| **DEFER-TO-CORE** | Data-access logic / credentials-in-transit / backend handling → `apache/opendal`#7641 |
| **OUT-OF-MODEL** | The trusted operator; `tests/`/`dev/`/`scripts/`; build/release |
| **DOWNSTREAM-RESPONSIBILITY** | Protecting the config/credential file; validating script-supplied args |
| **VALID** | CLI-specific defect: credential echoing, arg/URI parsing, config loading, under the §7 adversary |
| **MODEL-GAP** | Plausible, not covered → escalate to PMC |

## §14 Open questions for the maintainers

1. **(Q1)** Confirm the operator running `oli` is the trust anchor (trusted; holds credentials).
2. **(Q2)** Confirm `oli` is a thin veneer over the core (no first-party data-access logic to model here).
3. **(Q3)** How are backends + credentials configured (config file path/format, env)? Is that the CLI-specific credential surface?
4. **(Q4)** Local-FS interaction for `cp` to/from local paths — anything beyond "OS path semantics + invoking-user permissions"?
5. **(Q5)** Any environment assumptions worth stating?
6. **(Q6)** Is "driven by a less-trusted automation layer on attacker args" a real concern, or is `oli` always operator-invoked? Shapes the arg/URI-handling stance.
7. **(Q7)** Does `oli` ever print raw credentials (verbose/debug/error paths)? Confirm it must not.
8. **(Q8)** Confirm authorized destructive ops (`rm`, overwriting `cp`) are by-design, not in scope as a vuln.
9. **(Q18a)** What do scanners report against `oli` that you consider non-findings?
10. **(meta)** OK to land this thin model as `THREAT_MODEL.md` with the `AGENTS.md → SECURITY.md → THREAT_MODEL.md` chain, deferring data-access specifics to the core model?

## §15 Machine-readable companion

Not generated for v1.
