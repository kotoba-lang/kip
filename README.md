# kip

**KIP — Kotoba Improvement Proposal.** The numbered, cross-repository queue for
changes to the normative surfaces of the kotoba-lang workspace: EIP and IPIP's
job, done with the parts this workspace already has.

The name is an acronym, not a word, so this paragraph is the naming: `kip` holds
the *registry* (`kips/kip-NNNN.edn`), the *process authority*
(`lang/kip-process.edn`, machine-normative), and the *validator* that reads
them. It does not hold decisions — those stay in ADRs — and it does not hold
agent-effect approvals, which are
[`kotoba-issue`](https://github.com/kotoba-lang/kotoba-issue)'s job.

## What was missing, precisely

Measured 2026-08-16 across all 4,187 west projects: no repository and no
document describes a proposal process. What already exists, and is in several
places better than the thing being copied:

| EIP/IPIP's job | already here | verdict |
|---|---|---|
| the proposal document | ADRs — 43 in `kotoba-lang`, 221 in the compiler, thousands in the superproject | `surface-status.edn` already says widening a security constraint *"requires an ADR"* |
| editors | `docs/authority-map.edn` names an owner per normative surface | **better than EIP** — it is machine-checked |
| what Final costs | `lang/version-policy.edn`: 180-day deprecation floor, removal-requires-major, signed release tags binding the profile | **EIP has no equivalent** |
| propose → review → merge machinery | `kotoba-issue` + `kotoba-ledger`, with an append-only ledger | exists; its subject is agent effects |
| threshold voting | `kagami govern`, N-of-M from an allow-list | exists; its subject is west pins |

So the organs were all there. The gap is narrow and specific: **there is no
state between "someone is thinking about it" and "it is decided and written
up."** An ADR is written by the party that already decided. A KIP is the thing a
second person can object to *before* the fact — and because no queue existed,
nothing could check that a change to a normative surface went through one.

## The process

```
 draft ──author──▶ review ──editor──▶ last-call ──quorum──▶ final
   │                 ▲  │               │  │                  │
   │          editor ┘  │        editor ┘  │           quorum │
   │        (changes    │       (objection)│                  ▼
   │        requested)  │                  │              superseded
   └──────author────────┴──────────────────┴──▶ withdrawn
                        └──────editor──────┴──▶ rejected
```

- **standards** track changes a normative surface: last call *and* quorum.
- **process** and **informational** tracks: editors close them out of review.
  There is no last call because on those tracks the editors are the affected
  party.
- **Last call is 14 days minimum**, measured from `:kip/last-call-started`. The
  clock is the load-bearing part — a window with no minimum closes whenever the
  author is ready, which is the same as not having one. 14 days is the shortest
  interval that survives one person being away, which is the only failure mode a
  last call prevents.
- **`:editor` is not a standing committee.** For a given KIP it is the set of
  authority owners named by `lang/normative-surfaces.edn` for the surfaces that
  KIP touches. Who may advance it is a function of what it changes.
- **`:quorum` is 2 distinct signers** from `manifest/fleet-keys.edn`
  `:canonical :allow` in the superproject — the same list `kagami govern` reads.
  That list is *referenced, never copied*: two copies of a key list is two
  answers to who may sign.

`lang/kip-process.edn` is what the validator reads and is authoritative;
`kips/kip-0000.edn` is the human-readable statement of the same thing. If they
disagree the EDN wins — the same split `kotoba-lang/kotoba-lang` keeps between
`docs/lang/semantics-ssot.md` and `lang/guest-grammar.edn`.

## Layout

```
lang/kip-process.edn        machine-normative: states, transitions, guards, diagnostics
lang/normative-surfaces.edn which paths are under the process (:adopted / :proposed)
kips/kip-NNNN.edn           the registry — DataScript tx-data, :source/dataset "kip"
kotoba/kip_gate_core.kotoba the decision core: word-typed, native-admissible
src/kotoba/kip/core.cljc    codes, requirements, dates, quorum, reading documents
src/kotoba/kip/registry.cljc numbering, duplicates, dangling references
scripts/check-kips.cljs     the gate (nbb)
```

## Running it

```sh
npx --yes nbb --classpath src scripts/check-kips.cljs . \
  --surfaces ../kotoba-lang/docs/authority-map.edn

clojure -M:test          # 45 tests — includes the .kotoba parity sweep
clojure -M:test-pure     # the .cljc suite alone, no compiler dependency
```

`<root>` must be the **first** positional argument. Several gates in this
workspace take the tree as `(first (remove #(str/starts-with? % "--") argv))`,
so `check-kips.cljs --surfaces x .` would silently treat `x` as the tree.

### Exit codes

| code | meaning |
|---|---|
| 0 | checked, no errors |
| 1 | checked, errors found |
| **2** | **could not answer** — inputs missing or unreadable |

2 exists because 0 and 1 are both claims about the registry and "I never saw the
registry" is not one of them. `SCANNED` and `READ-OK` print every run and an
empty `kips/` exits 2 rather than reporting a clean sweep of nothing. Fourteen
instances of a check returning its pass value when it could not run were found
in this workspace in a single day (ADR-2608136000); an evidence floor plus a
third exit code is what fixed them.

Measured 2026-08-16, all four paths:

```
healthy tree         exit 0   SCANNED 2   RESULT pass
empty registry       exit 2   SCANNED 0   RESULT could-not-answer
missing process file exit 2               RESULT could-not-answer
planted duplicate    exit 1   SCANNED 3   RESULT fail
                              ERROR :kip/duplicate-number ["kips/kip-0000.edn" "kips/kip-0002.edn"]
```

## The decision core is `.kotoba`

`kotoba/kip_gate_core.kotoba` holds the judgement — may this KIP move from this
state to that one, and if not, which diagnostic — in `:i64`, `:bool` and two
sealed scalar records, which is what `kotoba-kir`'s
`only-native-word-typed-features?` admits. Collections, IO and the keyword
mapping stay in `.cljc`. That is the decision-core split, the same shape
murakumo's `kotoba/*_core.kotoba` use.

`test/kotoba/kip/kip_gate_kotoba_parity_test.clj` compiles the real source and
compares it against the `.cljc` on the **full** cross product — 441 state/role/
track cases and 160 guard cases, not a sample — and then compiles a
deliberately broken copy and asserts the sweep catches it, and that the cases
which flip are the ones the mutation touched.

Two measurements worth keeping:

- **`max-parameters` is 5** (amu `1e21a1f`). A 12-parameter `transition-code` is
  rejected at `:subset` with `:kotoba.error/max-parameters`. Two sealed records
  carry twelve fields through two parameters. Packing the flags into a bitmask
  would also fit and is the wrong answer — murakumo already went the other way
  (T5.3 replaced base-N packing with records), because a packed `i64` loses the
  field names that make the guard readable.
- `record-new` is a special form and is **not** subject to that limit
  (`max-record-fields` is 32).

## What is deliberately not here

**Signature verification is delegated.** `kotoba.kip.core/quorum-signers` takes
an optional `:verify-fn`; without one it counts *claimed* signers, reports
`:verified? false`, and the gate prints `QUORUM-VERIFY skipped:no-verifier` on
its own line rather than folding an unchecked signature into a pass. Wiring it
to kagami's ed25519 verification is a later KIP. **Until that lands, no KIP can
honestly reach `:final`, and none has** — kip-0000 and kip-0001 are both
`:review`.

**Coverage is not a fleet gate.** "Did this change to a normative surface
actually have a KIP" needs another repository's git history and this registry at
the same time, and a fleet gate is handed exactly one repository's tree. That
check lives operator-side at `scripts/verify-kip-coverage.cljs` in the
superproject. The precedent is `root-permit-index`, landed as a fleet gate whose
generator reads `orgs/cloud-itonami` — a path the root repository's git tree
does not contain. It has run about 300 times and has never been green. A gate
that cannot see its input is not strict, it is silent.

## Relationship to the neighbours

- **ADRs** keep doing exactly what they do. A KIP precedes a decision; an ADR
  records one. A Final KIP will usually be followed by an ADR in the repository
  that implements it.
- **`kotoba-issue` / `kotoba-ledger`** share the `issue → proposal → review →
  merge → audit` vocabulary by convention, not by dependency (the same
  arrangement those two have with each other). Their subject is agent effects.
- **`kagami govern`** owns the key list and the signature scheme. This
  repository references its allow-list and will call its verifier; it does not
  restate either.

Apache-2.0.
