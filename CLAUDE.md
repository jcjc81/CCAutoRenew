# CCAutoRenew

Daemon that automatically renews Claude Code 5-hour billing blocks by sending a minimal session at the right time.

## Commands

```bash
./claude-daemon-manager.sh start                          # start daemon (renews immediately on start)
./claude-daemon-manager.sh start --at "09:00"            # start with scheduled activation time
./claude-daemon-manager.sh start --at "09:00" --stop "17:00"
./claude-daemon-manager.sh start --message "continue X"  # custom renewal message
./claude-daemon-manager.sh stop
./claude-daemon-manager.sh restart                        # also triggers immediate renewal
./claude-daemon-manager.sh status
./claude-daemon-manager.sh dash                           # live dashboard (updates every 60s)
./claude-daemon-manager.sh logs [-f]

./test-quick.sh                    # fast smoke test (14 checks, ~5s)
./test-start-time-feature.sh       # comprehensive integration tests
```

## Architecture

```
claude-daemon-manager.sh     # user-facing CLI: start/stop/status/dash/logs
claude-auto-renew-daemon.sh  # long-running daemon process
lib/ccusage-utils.sh         # shared: ccusage queries, state file I/O
```

The manager launches the daemon via `nohup` and communicates through state files in `$HOME`.

## Key Design Decisions

- **`unset CLAUDECODE`** before invoking `claude` — required to allow renewal from inside an existing Claude Code session
- **`get_block_end_epoch`** (any active block) vs **`verify_session_active`** (active block with >60 min remaining) — use the former for "does a session exist", the latter for post-renewal verification
- **Renewal model**: `RENEWAL_MODEL="claude-haiku-4-5-20251001"` at top of daemon — cheap model for ping sessions
- **Renew on start**: manager touches `~/.claude-auto-renew-renew-on-start` before launching; daemon skips scheduling on first iteration and renews immediately, bypassing any stale billing block ccusage may report
- **Usage limit handling**: captures claude output, detects "hit your limit", parses reset time + timezone (handles same-day, day-of-week, and specific date formats), sleeps until 5 min past reset

## State Files (`$HOME`)

| File | Purpose |
|------|---------|
| `~/.claude-auto-renew-daemon.pid` | Daemon PID |
| `~/.claude-auto-renew-daemon.log` | Main activity log |
| `~/.claude-auto-renew-state` | Current block end epoch (for dashboard) |
| `~/.claude-auto-renew-start-time` | Scheduled start epoch |
| `~/.claude-auto-renew-stop-time` | Scheduled stop epoch |
| `~/.claude-auto-renew-message` | Custom renewal message |
| `~/.claude-auto-renew-renew-on-start` | Marker: renew immediately on next active iteration |
| `~/.claude-auto-renew-limit-reset` | Persisted limit reset epoch (survives crash, cleared on graceful stop) |
| `~/.claude-last-activity` | Timestamp of last successful renewal |

## Git

- Fork remote: `myfork` (jcjc81/CCAutoRenew)
- Upstream: `origin` (aniketkarne/CCAutoRenew) — do not push here
- Push to `myfork/main` for shipping
