# Sinabro and Naite Technical Report

> A local-first coding agent, user-owned memory system, and evidence-gated training loop
>
> Draft v0.2 | May 27, 2026

## Abstract

Sinabro is a local-first coding agent designed to operate near the user's repository, terminal, tools, credentials, memory, and approval boundary. Naite is the project coding model trained from the verified traces that Sinabro produces. The two are deliberately separated. Sinabro is the agent runtime and control plane. Naite is a replaceable accelerator.

The core thesis is simple: a useful coding agent should not be evaluated only by chat quality or model benchmarks. It should be evaluated by the loop it runs. A strong loop reads the codebase, selects context, decomposes work into atomic units, implements changes, executes tests and static analysis, records evidence, refuses unsafe actions, preserves user-owned memory, and converts verified work into training data. Sinabro is the system layer for that loop. Naite is trained only from data that survives it.

The project begins with Rust, Move, Sui, Walrus, local command execution, tool routing, web research with source evidence, skill discovery, wallet and gas policy, and long-context coding workflows. This is a narrow initial domain, but the underlying loop is general: plan, act, test, prove, measure, review, record, and learn.

Sinabro is open-source by design and conservative by default. Users can run it without producing training artifacts. They can also opt into local evidence bundles, private Naite diets, private adapters, or redacted contribution packets. Private memory, secrets, provider bodies, wallet material, sponsor keys, and rights-unclear web content do not become public training data.

This report is a design and architecture document, not a claim of completed production deployment. Implementation status belongs in repository build states, gate reports, and evidence bundles.

## 1. The Problem

Modern coding agents are becoming better at producing patches, but they still lose too much of the work that made the patch possible. A typical successful session contains repository reading, failed hypotheses, command output, compiler diagnostics, test results, security decisions, tool calls, human corrections, and policy denials. Most of that signal disappears into a transcript, if it is stored at all.

That is a poor foundation for a system that should improve over time. A patch is not enough. A chat transcript is not enough. A benchmark score is not enough. The missing unit is a verifiable work trajectory.

Sinabro treats each meaningful step as an atom. An atom is small enough to review and large enough to matter. It has one canonical output, explicit inputs, concrete tests, gate requirements, reuse constraints, and an evidence sidecar. The sidecar records what was read, what was changed, what was attempted, what failed, what passed, what was denied, and why the result can be trusted.

This gives Naite a better diet than ordinary instruction data. It does not learn from the model saying "I fixed it." It learns from compiler output, static analysis, test results, proof attempts, gas traces, dependency audits, red-team decisions, approval events, rejected actions, and human-reviewed diffs.

## 2. System Claims

Sinabro and Naite make four design claims.

1. The agent should be useful before the local model is strong.
2. Long-term memory should be owned by the user, not by a model checkpoint or hosted chat account.
3. Improvement should leave evidence.
4. Speed is a systems property, not a slogan.

The first claim prevents the model from becoming the authority. The runtime owns permissions, routing, memory, approvals, budget, and traces. The model proposes. Sinabro decides what can execute.

The second claim turns memory into a portable asset. A user should be able to change model providers, upgrade Naite, export memory, delete memory, replay work, or migrate storage backends without losing ownership of the work history.

The third claim makes training stricter. Naite is not rewarded for narrative success. It is rewarded for trajectories that pass gates.

The fourth claim follows the direction of recent inference systems: large language model performance is often limited by memory movement, KV cache pressure, prefill/decode interference, queueing, and scheduler behavior. Sinabro therefore separates user hot paths from full scans, full replays, full renders, synchronous network calls, and heavy evidence construction.

## 3. Architecture Overview

Sinabro is organized as a Rust control plane around platform-neutral envelopes.

```text
User
  -> CLI / TUI / Telegram
  -> CommandEnvelope / MessageEnvelope
  -> Sinabro Rust Core
       - Runtime supervisor
       - Turn orchestrator
       - LLM provider router
       - Tool adapter dispatcher
       - Memory engine
       - Skill registry and runtime
       - Wallet and gas policy
       - Evidence collector
       - Dataset builder
  -> Model backends
       - External providers
       - Local Naite
       - vLLM endpoint
  -> Substrates
       - cargo, clippy, miri, fuzzing, Kani
       - Sui CLI, Move tests, Move Prover
       - Walrus and StorageBackend adapters
       - Web research and browser tools
  -> Evidence bundle
  -> Naite training and evaluation
```

The router can call external providers or local models, but tool authority remains outside the model. A model can propose a command, web search, memory write, skill install, or transaction. Sinabro checks capability, policy, budget, approval, and evidence requirements before execution.

## 4. Design Principles

### 4.1 Agent first

Sinabro must remain useful with an external frontier model, a local Naite model, a smaller draft model, or no fine-tuned model at all. The model is not the product boundary. The agent runtime is.

### 4.2 Evidence over narration

A green status must point to evidence: command manifests, output hashes, test reports, prover results, gas traces, redaction reports, dependency audits, and approval receipts. A model statement is not evidence.

### 4.3 User-owned memory

Memory is not a hidden prompt cache. Memory is a user-owned asset. The root of trust is a composition of user-signed chunks, content digests, Sui `memory_root` and `audit_log` records, deterministic replay hashes, and backend-neutral storage receipts.

Walrus is the first Sui-native primary backend. It is not the definition of memory ownership. Local encrypted storage, Walrus, mirrors, archives, exports, deletions, and future backends sit behind a `StorageBackend` abstraction. No backend is allowed to replace the user's ownership proof.

### 4.4 No silent side effects

Sinabro does not silently execute tools, install skills, spend gas, sign transactions, publish packages, switch providers, train on private data, or fall back to another model. Side effects require capability checks, policy gates, budget checks, approval, and a user-visible trace.

### 4.5 Learning is opt-in

Open-source users may choose no learning artifacts, local evidence only, a private Naite diet, a private adapter, or a redacted contribution path. The default is no data egress and no training artifact generation.

### 4.6 Speed is designed, not declared

The system does not call itself fast because it streams text. It measures first-token latency, time per output token, stream gap, queue time, prefill time, decode time, throughput, allocation count, cache hit-rate, VRAM, and quality. A route that cannot explain its speed is not a stable route.

## 5. The Atom Protocol

An atom is the smallest planned unit that can produce a meaningful, reviewable change. Each atom has nine fields:

- `id`
- `file`
- `canonical OUT`
- `constraint spec`
- `tests`
- `criterion`
- `gate`
- `reuse`
- `next-atom`

The shape matters. It keeps work bounded, prevents broad self-reported success, and makes training data composable. An atom is not complete when the patch exists. It is complete when the patch, tests, gates, reviews, denials, redactions, and evidence records exist.

The atom protocol also constrains model behavior. A model cannot invent a new canonical type if a previous atom produced one. It cannot broaden the scope without showing the new scope. It cannot mark proof as passed when the tool is absent. It must distinguish failed work, denied work, unverified work, and successful work.

## 6. Evidence Sidecars

Each atom emits a closed sidecar contract. The current training sidecar contains 21 files:

```text
input_context.jsonl
action_trace.jsonl
command_manifest.json
terminal_redacted.jsonl
env_lock.json
artifact_hashes.json
code_diff.patch
failed_attempts.jsonl
no_op_decisions.jsonl
test_results.json
gate_results.json
review_5pack.json
deny_audit.json
redteam_decision.json
human_review.jsonl
approval_events.jsonl
privacy_report.json
sft_chat.jsonl
preference_pairs.jsonl
reward_labels.json
eval_summary.json
```

This sidecar is the bridge between engineering and learning. It records the task, environment, context, actions, command results, code changes, security review, dependency audit, redaction status, test outcomes, and reward labels.

Two files are especially important.

- `review_5pack.json` captures performance, security, chain, agent-token budget, and developer-experience review.
- `deny_audit.json` captures dependency, license, advisory, banned-surface, and source checks.

The point is not to create paperwork. The point is to make the system trainable without rewarding hallucinated success.

## 7. Memory Ownership and Replay

Sinabro's memory architecture separates continuity from model weights.

Work memories are typed chunks. A chunk can represent a user instruction, tool result, code change, approval event, test result, skill artifact, or system decision. Chunks are addressed by content digest and can be stored through a `StorageBackend`. Sui anchors a memory root and audit log so ownership and mutation history can be verified.

Replay is deterministic. A replay is accepted only if reconstructed content hashes match the expected memory root and audit trail. Deletion semantics are explicit: delete does not mean "hide it from retrieval while keeping it in training." Export, import, delete, anchor, root, and replay must be visible through the CLI.

This design is intended to give users several concrete properties:

1. A model upgrade does not erase memory.
2. A storage backend migration does not change ownership.
3. A hosted service cannot silently claim the memory root.
4. Training rights are separate from storage rights.
5. Memory can be audited after a dispute.

Storage is replaceable; ownership is not. Walrus is the first Sui-native primary backend, but Sinabro's memory layer is designed to support local encrypted storage, Walrus, Filecoin/IPFS, Arweave, and future archival backends through the same `StorageBackend` interface. The invariant is that no storage provider becomes the owner of memory.

The memory system also treats compression as part of the ownership experience. Long-term memory is only useful if it can be searched, summarized, paged, cached, replayed, and served quickly. Techniques such as prefix cache reuse, KV cache policy, quantized serving, and future TurboQuant-style compression are evaluated as serving optimizations, not as substitutes for verifiable memory roots.

## 8. Adapter Boundaries

Sinabro should use the external AI ecosystem without surrendering its safety model.

### 8.1 LLM Provider Abstraction

OpenAI, Anthropic, Gemini, local Naite, and vLLM endpoints are called through one provider interface. Model identity, route decision, cost estimate, latency bucket, prompt redaction hash, output hash, and fallback policy are attached at this layer.

No silent fallback is allowed. If a route changes from local Naite to a hosted provider, or from one hosted provider to another, the user-visible route state changes.

### 8.2 Tool Adapter Abstraction

Python tools, MCP servers, CLI binaries, HTTP services, FastAPI services, and WASM skills are normalized into a common `ToolCall` and `ToolResult`. Capability diff, sandbox tier, budget, approval, revocation, and evidence records are checked before execution.

This avoids a common failure mode: the model cannot bypass safety by choosing a more convenient tool surface.

### 8.3 Message and Command Envelope

CLI, TUI, Telegram, future mobile clients, and APIs use the same command semantics. `/kill`, `/approve`, `task resume`, `budget cap`, and `provider route` must mean the same thing no matter where they are issued.

### 8.4 Evidence and Trace Layer

Provider choice, tool execution, cost, latency, output hashes, failures, and approval receipts are recorded. Without this layer, an external model or Python tool could spend money, mutate files, or change routes without leaving a reproducible trail.

## 9. Skills

The initial skill system is adoption-first. It focuses on discovery, inspection, recommendation, dry-run, installation, provenance, compatibility, and verified use. Distribution economics are outside the core safety loop described in this report.

Planned commands include:

```text
skill search
skill inspect
skill recommend
skill use
skill install
skill enable
skill disable
skill update
skill remove
skill fork
skill publish
skill eval
skill provenance
```

Search uses progressive disclosure. The first result should be a compact card: name, description, capability summary, compatibility, evaluation status, security status, provenance, verified installs, and permission diff. Full manifests, WASM metadata, documentation, and eval logs load only after inspection.

Skill execution requires capability checks and user confirmation. A skill is not trusted because it is popular. Verified installs require package signatures, compatibility checks, malicious fixture tests, sandbox evidence, and install receipts.

## 10. Web Research and Source Truth

Web research is a tool action, not ambient knowledge. A fetched source becomes usable only when it carries:

- source URL
- retrieval time
- fetch hash
- content type
- rights or robots decision
- citation evidence
- credential redaction status

Web content can guide an answer. It does not automatically become Naite training data. Public training candidates require explicit rights checks, redaction, and user approval.

## 11. Wallet, Gas, and Chain Safety

The gasless user experience is designed around a keyless open-source client and a policy-gated Gas Station. Sponsor keys do not appear in the repository, binary, container image, or examples.

The Gas Station assumes its endpoint is public. Security is not based on hiding a URL. It is based on making unsafe signatures unreachable.

Core invariants include:

- No raw `GasData` lending endpoint.
- No signing of opaque transaction bytes.
- Package and function allowlists.
- Exact effect-shape checks through dry-run or dev-inspect.
- Gas, storage, object count, command count, and failure caps.
- Per-user, per-wallet, per-IP, per-ASN, per-package, per-skill, per-epoch, and global burn quotas.
- One gas coin lease per transaction.
- Small hot wallet balances behind cold treasury and multisig refill.
- Automatic pause on anomaly signals.

Gas sponsorship is separated from skill usage. The initial release allows bounded memory, audit, and registry writes under policy. Economic settlement for third-party skill distribution is outside Gas Station authority.

## 12. Speed Law

Sinabro follows a dual compression speed law.

Model-side compression reduces the cost of inference itself: KV cache policy, quantized serving candidates, active adapter identity, and speculative draft/verify routes are visible in the route trace.

Serving-side compression reduces system overhead around the model: prefix cache, KV reuse, paged trace rendering, background work admission, queue priority, prefill/decode separation candidates, and zero-allocation hot paths.

The stable route cannot hide behind aggregate latency. It must report:

- TTFT: time to first token
- TPOT: time per output token
- stream gap
- queue time
- prefill time
- decode time
- throughput
- hot path allocation count
- prefix cache hit-rate
- KV reuse hit-rate
- VRAM
- quality regression

Full operations are supported, but only explicitly. `--full`, `--deep`, `export`, `replay`, and `audit` are product features. They run as budgeted, killable, resumable background jobs with progress, paged output, and evidence. They are not allowed to block the interactive hot path.

This is why Sinabro's speed policy is not "do less." It is "do heavy work where it belongs."

## 13. Training Naite

Naite training begins only after the dataset and rights gates produce a valid unlock packet. The first training track uses a 14B coding-model lineage with QLoRA/LoRA style fine-tuning on A100-class hardware. Reinforcement-style methods such as GRPO, MURPHY, or FGO remain locked until SFT smoke tests and evaluations pass.

Promotion is not based on loss alone. A candidate must preserve or improve results across:

- Rust compile and test outcomes.
- `cargo fmt`, `cargo clippy`, `cargo miri`, fuzzing, and property tests.
- Kani checks.
- Move tests and Move Prover repair.
- Gas and byte-size behavior.
- Walrus and storage integrity tasks.
- Korean technical instruction following.
- Long-context retrieval and packing tests.
- Held-out security and optimization tasks.

The training split distinguishes verifiable data from narrative data. Compiler and prover output can support reward labels. Self-report, praise, and unsupported summaries cannot. Infrastructure failures such as OOM or provider timeout are masked rather than treated as model failures.

## 14. Evaluation

Evaluation covers model quality, agent behavior, system safety, and serving performance.

| Area | Examples |
| --- | --- |
| Rust | fmt, clippy, test, miri, fuzz, criterion, Kani |
| Move and Sui | move test, Move Prover, BCS parity, gas trace, owner invariants |
| Storage | Walrus PUT/GET, blob-id verification, backend receipts, replay determinism |
| Memory | root hash, audit log, deletion, export/import, replay hash |
| Skills | malicious fixtures, capability diffs, signed package checks, no-commerce scans |
| Web | source metadata, retrieval hashes, rights checks, credential redaction |
| Router | no silent fallback, route receipts, cost and quality scorecards |
| Gas | allowlist, dry-run, quotas, gas caps, coin leases, burn caps |
| Dataset | PII0, secret0, S1/S2 split, reward firewall, dependency audit |
| Korean | technical parity with equivalent English prompts |
| Serving | TTFT, TPOT, prefill/decode, prefix/KV hit-rate, allocation, VRAM, quality |

The strongest evaluation target is not memorization of the project itself. The system must improve on held-out Rust, Move, gas, security, and Korean-language technical tasks without relaxing safety gates.

## 15. Open-Source Controls

Sinabro is intended to be open-source and user-controlled. The default configuration is conservative.

```toml
[learning]
mode = "off" # off | evidence_only | local_diet | private_adapter | contribute_redacted
global_contribution = false
external_model_output_training = "never"
private_repo_training = "never"

[features]
trace = "minimal"
startup_full_scan = false
hot_path_full_scan = false
trace_render = "paged"
memory_replay = "background"
skill_manifest_load = "lazy_inspect"
full_operation_mode = "explicit_background"
full_operation_budget_prompt = true
speculation = true
prompt_cache = true
context_compression = true
prefix_cache = true
kv_cache_hit_rate = "measure"
ttft_tpot_split = true
prefill_decode_split = "candidate"
hot_path_allocation = "zero"
background_queue_priority = "interactive_first"
speculative_route = "visible"
quantized_serving = "canary_only"
```

The safety kernel is not optional. Secret redaction, capability diffs, no silent fallback, no auto-merge, wallet preview, gas drain invariants, mainnet approval, and source-evidence requirements remain enforced.

## 16. Roadmap

The staged roadmap is designed so each stage leaves a working artifact and an evidence bundle.

- Stage A: core runtime, trace collection, agent loop, typed units, sidecar grammar.
- Stage B: signed memory chunks, Walrus testnet, Sui memory roots, replay proof.
- Stage C: GA hardening, gas trace harness, mainnet gate, key isolation.
- Stage D: skill runtime, open registry, provenance, install receipts, memory intelligence.
- Stage E: AtomDietRecord builder, redaction, rights checks, reward firewall, training unlock.
- Stage F: CLI cockpit, provider and tool adapters, web, skill, memory, wallet, gas, train, eval, and feature controls.
- Stage G: first Naite SFT pass and evaluation on A100.
- Stage H: vLLM serving, local model router, speed law, no-silent-fallback proof, CLI/Telegram sync.
- Stage I: read-only mainnet measurement for gas, cycle, and security telemetry.
- Stage J: open-source readiness, SDKs, public docs, contributor dry-run, review queue.
- Stage K: larger-model promotion after controlled self-improvement is measured under gates.

## 17. Expected Advantages

Sinabro and Naite compound in ways a plain chat interface does not.

The agent keeps state outside the model. Memory, skills, evidence, and approval records survive provider changes and model upgrades. The training system learns from complete work trajectories instead of isolated answers. The safety model is enforced by runtime policy rather than prompt text. External frontier models can still be used, but their outputs pass through the same trace, privacy, and approval boundaries.

The system is broader than a Web3 assistant and more specific than a general chatbot. Its first deep specialization is Rust, Move, Sui, and storage-backed memory, but the underlying loop is a general coding loop.

## 18. Limitations

This report describes a staged design, not a completed production deployment.

Naite is not assumed to outperform frontier models at launch. Walrus/Sui memory ownership, Gas Station operation, vLLM serving, and mainnet measurement require implementation evidence before public performance claims. Self-improvement does not imply permission escalation. A better model does not get broader authority.

Public training contributions require opt-in consent, redaction, source rights, provider-policy compliance, and provenance. Some evaluation tools may be unavailable in a given local environment; tool absence must be recorded as `not_verified`, not as success.

## 19. Conclusion

Sinabro is the agent layer. Naite is the model trained from the agent's verified work. The system is built around one constraint: improvement must leave evidence.

If the project succeeds, its strength will not come from a single model release. It will come from the loop: user-owned memory, open skills, strict approvals, reproducible traces, speed measured at the systems boundary, and training data earned by real verification rather than asserted by a model.

## References

- DeepSeek-AI. "DeepSeek-V3 Technical Report." arXiv:2412.19437, 2024. https://arxiv.org/abs/2412.19437
- Google Research. "TurboQuant: Redefining AI efficiency with extreme compression." 2026. https://research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/
- Kwon et al. "Efficient Memory Management for Large Language Model Serving with PagedAttention." arXiv:2309.06180, 2023. https://arxiv.org/abs/2309.06180
- Qin et al. "Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving." arXiv:2407.00079, 2024. https://arxiv.org/abs/2407.00079
- Zhong et al. "DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving." arXiv:2401.09670, 2024. https://arxiv.org/abs/2401.09670
- Packer et al. "MemGPT: Towards LLMs as Operating Systems." arXiv:2310.08560, 2023. https://arxiv.org/abs/2310.08560
- Yang et al. "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering." arXiv:2405.15793, 2024. https://arxiv.org/abs/2405.15793
- Wang et al. "Voyager: An Open-Ended Embodied Agent with Large Language Models." arXiv:2305.16291, 2023. https://arxiv.org/abs/2305.16291
- Ryan Teknium, Jeffrey Quesnelle, Chen Guang. "Hermes 3 Technical Report." arXiv:2408.11857, 2024. https://arxiv.org/abs/2408.11857
- OpenAI. "GPT-4 Technical Report." arXiv:2303.08774, 2023. https://arxiv.org/abs/2303.08774
- OpenAI. "GPT-4o System Card." 2024. https://openai.com/index/gpt-4o-system-card/
