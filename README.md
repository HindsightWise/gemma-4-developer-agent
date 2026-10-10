# Google - The Gemma 4 Developer Agent Competition
**Competition ID:** `149921` | **Paper Track ID:** `163111`  
**Host:** Google DeepMind / Google Research (Elan Markowitz, Bryan Perozzi, Benedek Rózemberczki, Glenn Cameron, Hadi Hemmati, Yuchen Li, Michael Galkin, Majid Farhadi, Ryan Holbrook, Ashley Oldacre)  
**Total Prize Purse:** $100,000 USD ($65,000 Main Track + $35,000 Paper Track)  
**Deadline:** December 2, 2026 at 11:59 PM UTC (Paper Track: November 12, 2026; Merger: November 25, 2026)

---

## 1. System Specification & Hardware Environment

- **Evaluation Hardware:** 4× NVIDIA L4 GPUs (24GB VRAM each = 96GB aggregate VRAM).
- **Base Model Variant:** Strictly `gemma-4-31b-it-qat-w4a16-ct` (INT4 QAT weights, FP16 activations).
- **Execution Budget:** Exactly **720 minutes (12 hours)** wall-clock time sequentially across ~120 hidden benchmark tasks.
- **Per-Task Pacing Budget:** Allocate $\le 4.0\text{--}5.0$ minutes per issue to prevent global timeout truncation.
- **Grading Sandbox:**
  - **Phase 1 (Container A):** Git repository extracted at `base_commit`, baseline commit `_swegemma_baseline` created. Agent interacts via 9 host tools and invokes `submit_patch()` to capture unified diff.
  - **Phase 2 (Container B):** Fresh container without agent history. Repository checked out at `base_commit`. Agent patch applied via `git apply`.
  - **Anti-Tampering Reset:** `git checkout HEAD -- tests/ conftest.py pytest.ini setup.cfg pyproject.toml` and `git clean -f` wipe any test modifications before validation. Hidden unit tests (`test_patch`) are applied and evaluated with `python3 -m pytest`. Resolution requires exit code `0` and 100% test pass.

---

## 2. Benchmark Dataset Topology

The public development set contains **129 benchmark tasks** in `tasks.jsonl`:
- `fastapi/fastapi`: **67 tasks (51.9%)** — Async route handlers, Starlette request lifecycles, and Pydantic validation schemas.
- `Textualize/rich`: **48 tasks (37.2%)** — Console markup, syntax highlighting, layout tables, and ANSI terminal rendering.
- `psf/requests`: **13 tasks (10.1%)** — HTTP session hooks, authentication handlers, redirect logic, and proxy adapters.
- `encode/httpx`: **1 task (0.8%)** — Async HTTP transport and connection pooling.

The hidden evaluation set contains **~120 instances** (50% Public Leaderboard / 50% Private Leaderboard).

---

## 3. The 6 Critical Benchmark Traps & Autonomous Intercepts

1. **The 0-Byte Hard-Link ZIP Archival Failure:**
   - *Trap:* `graphs/*.json` and `embeddings/*.npz` were archived using hard links. Standard ZIP decompression flattens hard links into 0-byte empty files for ~50% of the dataset.
   - *Intercept:* Do not rely exclusively on graph tools (`get_code_neighbors`, `search_similar_code`). Enforce a source-first reconnaissance doctrine using `run_command("git grep -n ...")` and `read_file`.
2. **The FastAPI Async AST Void:**
   - *Trap:* The AST generator omitted `async def` nodes from the multigraph. For `fastapi` (52% of dataset), route handlers and async dependencies are completely absent from graph tools.
   - *Intercept:* Prohibit graph tool calls for `fastapi` instances; localize symbols directly via bash grep and syntax search.
3. **Python 3.13 Runtime Migration:**
   - *Trap:* The evaluation image runs Python 3.13, which removed legacy modules `imp` and `telnetlib`.
   - *Intercept:* Use isolated pytest target execution or lightweight compatibility shims.
4. **Double-JSON Escaping in Tool Results:**
   - *Trap:* Tool results are wrapped in JSON, causing quotes and newlines to double-escape (`\"` becomes `\\\"`). Gemma 4 copies this escaped text into `edit_file`, triggering `"old_string not found"` errors.
   - *Intercept:* Anchor `edit_file` with short, 1–2 line distinct target strings. For complex modifications, overwrite cleanly via `write_file` or bash python scripts.
5. **LoRA KV Cache Starvation:**
   - *Trap:* Default LoRA configuration in vLLM 0.19.1 reserves ~8.8GB per GPU, shrinking the KV cache to 7,600 tokens and causing hangs.
   - *Intercept:* Use rank $r=16$ or $r=32$ and mount at most 1 adapter, or deploy prompt-only zero-LoRA.
6. **Context Overflow Diff Loss:**
   - *Trap:* `ContextWindowExceededError` in `agent_runner.py` bypasses the fallback git diff capture block, resulting in empty patches and zero scores.
   - *Intercept:* Always call `submit_patch()` immediately after applying edits before running verification or long explanations.

---

## 4. Submission Directory Architecture (`submission.zip`)

```
submission.zip
├── agent.yaml                       # Root LlmAgent (swe_coder)
├── configs/
│   └── sampling.yaml                # Generation & thinking config
├── eval_config.yaml                 # Per-task execution budget
├── prompts/
│   ├── coder_system.md              # Root coder system instructions
│   └── analyzer_system.md           # Analyzer sub-agent instructions
├── sub_agents/
│   └── code_analyzer.yaml           # Sub-agent declaration (AgentTool)
├── skills/                          # ADK Skill directories (optional)
│   └── repo_navigation/
│       ├── SKILL.md
│       ├── scripts/
│       └── resources/
└── adapters/                        # PEFT LoRA adapters (optional)
    └── gemma4_swe_lora/
        ├── adapter_config.json
        └── adapter_model.safetensors
```
