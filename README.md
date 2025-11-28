# llama-pack

Smart SSD caching for llama.cpp models. Automatically copies frequently-used models from slow HDD to fast SSD.

## Quick Start
```bash
# Alias for convenience
alias llp=llama-pack

# First run creates default config
llp --update
# Edit ~/.config/llama-pack/config.json - set your ssd_path, hdd_path, and server_binary

# Scan your model directories
llp --update

# Load a model by partial name match
llp -m gemma5
# Finds "Gemma2-9B-Q5_K.gguf" if it's the only match, starts llama-server

# Override llama-server args once
llp -m gemma5 -- -c 8192

# Store custom args for this model permanently
llp -m gemma5 --store -- -c 8192 -ngl 40
# Next time: llp -m gemma5 uses stored args automatically

# After a few uses, model gets cached to SSD
llp -m gemma5
# [Detected multiple recent uses, copying to SSD in background...]
# [Server starts from HDD while copy happens]

# Next run loads from SSD (faster!)
llp -m gemma5
# [Loading from SSD: ~/.../models/ssd/Gemma2-9B-Q5_K.gguf]

# Kill running server
llp -k
# Or kill all: llp --killall

# List all models with stats
llp -l
# Shows: rank, SSD/HDD status, usage stats, last used date
# SSD = cached & hot, {SSD} = should cache, HDD = permanent storage, st = stats only

# Show currently running servers
llp --status
```

## How It Works

- **Automatic caching**: 3+ uses within 7 days → copies to SSD
- **Smart expiry**: Low-ranked models removed from SSD when space is tight
- **Rank-based loading**: Multiple matches? Picks most recently/frequently used
- **Per-model args**: Store custom llama-server flags per model with `--store`

## Config

Edit `~/.config/llama-pack/config.json`:
- `ssd_path` / `hdd_path`: Where models live
- `ssd_min_free_gb`: Minimum free space to maintain on SSD
- `global_llama_args`: Args passed to ALL models
- `server_binary`: Path to llama-server binary

## Tips

Run with `-v` for verbose output showing paths and decisions.
Run with `-v -v` for even more detail.
```

---

Now, want me to implement the `-v` path display feature for `-l`? It would show indented paths under each model like:
```
Gemma2-9B-Q5_K.gguf
   6.5  SSD  hdd  2025-11-28    6  2025-11-20
   ssd: /home/user/.../models/ssd/Gemma2-9B-Q5_K.gguf
   hdd: /home/user/.../models/hdd/llm/Gemma2-9B-Q5_K.gguf