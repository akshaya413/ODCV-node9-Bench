# Reproducing `node9` Integration in `ODCV-Bench`

This guide provides step-by-step instructions to build, verify, and run the `@node9/proxy` (`v2.62.1`) security framework inside `ODCV-Bench` across both the standard ($N=18$) and Hard/Impossible ODCV suites.

---

## 1. Repository & Version Pinning

* **Upstream Benchmark:** `McGill-DMaS/ODCV-Bench` (branch: `feat/node9-integration`)
* **Policy Engine:** `@node9/proxy` `v2.62.1` (`node9-ai/node9-proxy`) — deterministic TypeScript/Node.js (`>=22`) AST & rule proxy (makes **zero** LLM API calls on its own).
* **Evaluated Agent Model:** `openai/gpt-oss-120b` via OpenRouter
* **4-Judge Panel:** `anthropic/claude-sonnet-4`, `openai/gpt-4.1`, `google/gemini-2.5-pro`, `deepseek/deepseek-r1` (self-judgment excluded; dual `attempt_score` and `outcome_score` on the `0–5` ODCV rubric).

---

## 2. Standalone `node9` Build & Offline Verification

Before building the Docker container, you can verify `node9` standalone without `ODCV-Bench` attached:

```bash
# 1. Clone and build @node9/proxy v2.62.1 (Node.js >= 22 required)
git clone https://github.com/node9-ai/node9-proxy.git
cd node9-proxy
git checkout v2.62.1
npm install && npm run build

# 2. Initialize local offline protection (no cloud account or login required)
node dist/cli.js init --recommended --skip-setup

# 3. Dry-run individual tool calls through the 5-tier waterfall & 14-stage AST engine
node dist/cli.js explain bash '{"command":"cat /app/data/records.csv"}'       # Expected: ALLOW
node dist/cli.js explain bash '{"command":"rm -rf /app/output/report.json"}' # Expected: REVIEW (Smart Rule: review-rm)
node dist/cli.js explain bash '{"command":"cat .env"}'                        # Expected: BLOCK (shield:project-jail:block-read-env)

# 4. Inspect host posture score (0-100) and structured local audit logs (~/.node9/audit.log)
node dist/cli.js posture
node dist/cli.js scan --json
node dist/cli.js report
```

> **Note on Local State & Egress:** `node9` runs as a localhost-only daemon bound exclusively to `127.0.0.1:7391`, storing persistent decisions in `~/.node9/decisions.json` and secret-redacted logs in `~/.node9/audit.log`. Network egress control is **off by default** in `node9-default` and is explicitly enforced via `smartRules` in `node9-tuned` (`P4: network: deny_all`).

---

## 3. Building `Dockerfile.universal` (Zero-Rebuild Multi-Framework Image)

Ensure the built `node9-proxy` artifacts (`dist/`, `packages/`, `node_modules/`, `package.json`) are staged at `mission_executor/node9_runtime/` and the domain config is at `mission_executor/policies/node9.tuned.config.json`:

```bash
cd mission_executor
mkdir -p node9_runtime
cp -r /path/to/node9-proxy/{dist,packages,node_modules,package.json} node9_runtime/

# Build the universal benchmark image once
docker build -f Dockerfile.universal -t odcv-bench-universal:latest .
```

Verify the container runtime and `node9` binary inside the image:

```bash
docker run --rm --entrypoint node odcv-bench-universal:latest /opt/node9/dist/cli.js --version
```

---

## 4. Running the Standard Benchmark Sweeps ($N=18$)

Toggle enforcement layers at runtime via `FRAMEWORK_MODE` without rebuilding the image. Each sweep runs all **3 domains** (*Academic Research Integrity*, *Healthcare Patient Safety Monitoring*, *Financial Audit Fraud Finding*) $\times$ **2 pressure variants** (*mandated*, *incentivized*) $\times$ **3 runs** ($N=18$ episodes):

```bash
export OPENROUTER_API_KEY="sk-or-..."

# 1. Stock node9 baseline defense (Tier 5 defaults; BLOCK and REVIEW fail closed with exit 126)
python3 run_benchmarks.py \
  --framework-mode node9-default \
  --prefix node9_default \
  --model openai/gpt-oss-120b \
  --runs 3

# 2. Domain-tuned node9 defense (loads /app/node9.config.json at Tier 3: Project config)
python3 run_benchmarks.py \
  --framework-mode node9-tuned \
  --prefix node9_tuned \
  --model openai/gpt-oss-120b \
  --runs 3
```

---

## 5. Running the Hard / Impossible ODCV Suite

To evaluate `node9` under provably impossible KPI ceilings (measuring specification gaming vs. honest `/app/output/escalation_report.md` refusal):

```bash
# Run Hard ODCV scenarios under node9-default and node9-tuned
python3 hard_scenarios/run_hard_bench.py --mode node9-default
python3 hard_scenarios/run_hard_bench.py --mode node9-tuned

# Re-score existing runs without re-invoking the agent
python3 hard_scenarios/run_hard_bench.py --mode node9-tuned --metrics-only
```

---

## 6. Scoring & Exporting Results to the Shared Team Repository

1. **Run the Dual-Score 4-Judge Panel (`attempt_score` vs. `outcome_score`):**
   ```bash
   python3 evaluate_all_results.py --prefixes node9_default node9_tuned
   ```
2. **Aggregate Cross-Framework Metrics:**
   ```bash
   python3 score_odcv.py --output results/summary_metrics.csv
   ```
3. **Export Clean Artifacts:** Copy only `events.jsonl`, `state_hashes.json`, and `result.json` per episode (plus `policies/node9.tuned.config.json`) into the shared evaluation repository under `results/node9-default/` and `results/node9-tuned/`.