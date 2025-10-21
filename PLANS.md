# llama-serve: Future Plans

## v2 Features

### Nicknames & Aliases
**CLI:** `--nickname foo -m model.gguf [--no-load]`, `--nickdel foo -m pattern`, `-n nick` to load by nickname  
**Why:** Typing model filenames is tedious. Store nicknames in stats.json alongside model data. Simple string-to-path mapping that persists across sessions. `--no-load` lets you manage names without starting server.

### Multiple Nickname Types (nick1/tags)
**CLI:** `--nick1 tool -m model.gguf`, `-n1 tool` to load the single model tagged "tool"  
**Why:** Global semantic slots. When testing "use new tool model everywhere", just change which model has `nick1=tool`. Only one model can hold each nick1 value - provides stable references that update atomically when you reassign them.

### Nickname Addition (not replacement)
**CLI:** `--nickadd alt_name` or `--na alt_name`  
**Why:** Models may have multiple natural names. `--nickname` replaces all, `--nickadd` appends. Lets you build up aliases organically: `--na mixtral --na mix --na moe`.

### Smart Cache Expiry
**Current:** Models copy to SSD but never expire  
**Plan:** When SSD needs space, expire lowest-ranked model that hasn't been used in 7+ days. Check `ssd_min_free_gb` before copying. Warn user in RED when expiring.  
**Why:** Finite SSD space requires eviction policy. Low-rank + stale = safe to expire. User stays informed about what's happening to their cache.

### Enhanced Ranking Algorithm
**Current:** `total_uses - (days_since_last_use * 2)`  
**Proposed:** `(recent_7d * 10) + (recent_30d * 3) + (total_uses * 0.5) + (days_recent_bonus) - (days_since * 2)`  
**Why:** Better captures usage patterns. Heavy recent use signals active work (weight it high). Ancient history matters less. Bonus for <7 days keeps actively-tested models cached even with low total uses.

### Runtime Duration Tracking
**Current:** Start time recorded, but not duration  
**Plan:** On SIGINT/SIGTERM, calculate `time() - $current_runtime_start`, add to `total_runtime_seconds` in stats.  
**Why:** Long-running models indicate real use vs quick tests. A model that runs 8 hours daily for a week deserves higher cache priority than one loaded 20 times for 30 seconds each. Complements frequency-based ranking.

### Reload Throttling
**Config:** `min_reload_seconds: 30`  
**Behavior:** Check PID file mtime. If last reload was <30s ago on this port, exit with message: "Too soon to reload. Wait Ns."  
**Why:** Prevents flutter when rapidly testing model switches. Simple mtime check = no state to track. Configurable per-user tolerance. Helps catch accidental double-invocations.

### Multi-match Handling
**Current:** Multiple matches pick highest-ranked  
**Plan:** Add `--rank1` flag to auto-pick top rank. Add `--no-multi-match` to exit(1) on ambiguity.  
**Why:** Interactive use wants smart defaults (pick best). Scripts want explicit failure on ambiguity. Both workflows supported with flags.

### Process Self-Management
**Behavior:** Script finds its own running instances (by grepping ps or reading all PID files), compares ports, kills conflicts by default  
**Flags:** `--nokill` preserves existing if ports differ  
**Why:** Eliminates manual port conflict resolution. Most use case: "load this model now" means "replace whatever's running". Advanced users with multi-port setups opt into `--nokill`.

## v3+ Features

### Simulation Mode
**CLI:** `--simulate scenario_name` or `--sim builtin1`  
**Format:** Scenarios as offset arrays: `["0s", "1h", "1d", "2d", "7d"]` = usage timestamps  
**Output:** Color-coded actions (GREEN=keep, YELLOW=copy to SSD, RED=expire from SSD)  
**Why:** Algorithm changes need validation. Built-in scenarios test edge cases (e.g., "old heavy use vs new frequent use"). Visual feedback builds confidence before real-world use. Color helps scan decisions quickly.

### Predictive Pre-caching
**Trigger:** When cache expiry makes space, check if any HDD model now ranks high enough to deserve SSD  
**Action:** Proactively copy high-rank HDD models to newly-freed SSD space  
**Why:** Cache churn creates opportunity. If model B expires, and model C on HDD is now top-5 ranked, copy C. Anticipates next user request. Turns maintenance into optimization.

### Background Copy Progress
**Current:** Silent fork, only shows on completion if verbose  
**Plan:** Progress bar or percentage logged to separate file/stdout  
**Why:** Large models take time. Feedback prevents "is it working?" anxiety. Low priority since copy happens in background - you can use the model immediately from HDD anyway.

### Multiple HDD Paths
**Config:** `"hdd_paths": ["/hdd1/models", "/hdd2/models"]`  
**Why:** Some users split models across drives. Scan all, prioritize SSD > HDD1 > HDD2 when finding models. Minimal code change - just loop `find()`.

### Model Metadata Cache
**Current:** Stats only track usage, not model properties  
**Plan:** Store parameter count, quant type, context size (parsed from filename or metadata)  
**Display:** Show in `--list` output: `model.gguf  7B Q4_K_M  rank:42 ...`  
**Why:** Helps select appropriate model at a glance. Filename parsing heuristics get 90% coverage. Future: read GGUF headers for perfect accuracy.

## Non-goals

**Web UI:** llama-swap already does this. We're a CLI tool.  
**Model downloads:** Use `huggingface-cli` or similar. Scope creep.  
**Quantization:** External tool territory. We manage, not modify.  
**Multi-user:** Single-user tool. System-wide deployment needs different architecture (daemon, proper access controls, etc).

## Implementation Notes

**Stats file format:** Keep JSON human-readable. Add version field for future migrations. Consider size limits (trim old timestamps after 100 entries already implemented).

**Config validation:** Warn on first run if paths don't exist. Don't auto-create model dirs (user might typo path).

**Error handling:** Use color-coded severity. Red BG for errors, yellow for warnings, green for success, blue for info. Consistent visual language.

**Testing:** Simulation mode doubles as test suite. Add more scenarios as edge cases discovered. Consider shipping test scenarios in `~/.config/llama-serve/scenarios/`.

**Dependencies:** Keep minimal. Current deps (JSON::PP, File::Find, Time::Piece) are all core Perl. Avoid CPAN dependencies where possible for easy deployment.

## Philosophy

Keep the CLI intuitive for daily use while exposing power-user flags. Sane defaults should "just work" - explicit flags enable alternate behaviors. Colors guide attention to important info. Smart automation (background caching, auto-expiry) stays invisible until it matters. When it matters, explain clearly what happened and why.