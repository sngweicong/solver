# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```sh
# Build (stable, default features)
cargo build --release

# Build with all features
cargo build --release --features zstd

# Build with nightly-only custom allocator
cargo +nightly build --release --features custom-alloc

# Run tests
cargo test --release --features zstd
cargo +nightly test --release --features custom-alloc -- --test-threads 1

# Run a specific test
cargo test --release <test_name>

# Run examples
cargo run --release --example basic
cargo run --release --example file_io
cargo run --release --example node_locking

# Lint (suppressing one allowed lint)
cargo clippy --release --features zstd -- -A clippy::needless_range_loop

# Format check
cargo fmt --all --check

# Auto-format
cargo fmt --all

# Build docs
cargo doc --release
```

CI runs with `RUSTFLAGS: --deny warnings` and `RUSTDOCFLAGS: --deny warnings`, so all warnings are treated as errors.

## Architecture

This is a postflop Texas hold'em GTO solver library. The primary consumers are GUI apps (WASM Postflop, Desktop Postflop). The public API is flat — all public items are re-exported from `src/lib.rs`.

### Core data flow

1. **Configure** — Create `CardConfig` (ranges, board) and `TreeConfig` (pot, stack, bet sizes)
2. **Build** — `ActionTree::new(tree_config)` builds the action tree; `PostFlopGame::with_config(card_config, action_tree)` initializes the game
3. **Allocate** — `game.allocate_memory(compression: bool)` allocates storage for CFR data
4. **Solve** — `solve(&mut game, max_iterations, target_exploitability, print_progress)` runs Discounted CFR
5. **Interpret** — Navigate with `game.play(action_index)`, inspect with `game.strategy()`, `game.equity()`, `game.expected_values()`, etc.

### Key modules

- **`src/interface.rs`** — `Game` and `GameNode` traits that abstract over the solver. The solver (`solver.rs`) is generic over any `Game` implementor.
- **`src/game/`** — `PostFlopGame` implementing the `Game` trait:
  - `mod.rs` — Struct definitions (`PostFlopGame`, `PostFlopNode`, `State` enum)
  - `base.rs` — `Game` trait impl: tree building, memory allocation, isomorphism, bunching
  - `evaluation.rs` — Terminal node evaluation (showdowns, folds, rake)
  - `interpreter.rs` — Navigation API (`play`, `back_to_root`, `apply_history`, `strategy`, `equity`, etc.)
  - `node.rs` — `GameNode` trait impl for `PostFlopNode`, including compressed storage accessors
  - `serialization.rs` — bincode Encode/Decode for save/load (behind `bincode` feature)
- **`src/solver.rs`** — Discounted CFR implementation. `solve()` runs full solve; `solve_step()` runs one iteration. `compute_exploitability()` and `finalize()` are in `utility.rs`.
- **`src/action_tree.rs`** — `ActionTree`, `TreeConfig`, `Action`, `BoardState`. The tree can be edited manually after construction (add/remove lines).
- **`src/bet_size.rs`** — `BetSizeOptions` / `DonkSizeOptions` parsing (e.g. `"60%, e, a"` / `"2.5x"`).
- **`src/range.rs`** — Hand range parsing and representation.
- **`src/card.rs`** — Card/suit/rank primitives; `card_from_str`, `flop_from_str`, `holes_to_strings`.
- **`src/bunching.rs`** — Bunching effect (dead cards from folded players) support.
- **`src/sliceop.rs`** — SIMD-friendly slice arithmetic used throughout the hot path.
- **`src/mutex_like.rs`** — `MutexLike<T>` abstraction: real `Mutex` in single-threaded builds, `UnsafeCell` wrapper when rayon is enabled for lock-free parallelism.
- **`src/alloc.rs`** — `StackAlloc` custom allocator (nightly-only `custom-alloc` feature).
- **`src/file.rs`** — Save/load via `save_data_to_file` / `load_data_from_file` (behind `bincode` feature; optionally compressed with `zstd`).

### Memory model

`PostFlopGame` owns flat byte arrays (`storage1`, `storage2`, `storage_ip`, `storage_chance`) that back all per-node data. `PostFlopNode` holds raw pointers into these arrays. `unsafe impl Send/Sync` is applied manually. The `MutexLike` abstraction in `mutex_like.rs` makes the rayon parallel traversal safe.

### Compression

When `allocate_memory(true)` is called, strategies and regrets are stored as `u16`/`i16` with a per-node `f32` scale factor, halving memory use at the cost of some precision.

### Isomorphism

Suit-isomorphic turn/river cards are collapsed: only one representative is solved, and results are mapped back via swap lists stored in `PostFlopGame`.

### Crate features

| Feature | Default | Purpose |
|---|---|---|
| `bincode` | yes | Save/load game trees |
| `rayon` | yes | Multithreaded solving |
| `zstd` | no | Compressed file I/O |
| `custom-alloc` | no | Nightly-only stack allocator for hot path |
