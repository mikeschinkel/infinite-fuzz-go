# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains `infinite-fuzz.sh`, a standalone bash script for continuous Go fuzz testing. It's designed to be copied into any Go project's test directory to enable parallel, continuous fuzzing that automatically restarts after finding issues.

**Key Point**: This repo contains NO Go code - it's purely a bash script utility. The script is meant to be used in other Go projects, not run from this repository.

## Script Architecture

### Core Functionality
- **Auto-discovery**: Scans `*_test.go` files for `func Fuzz*` patterns
- **Parallel execution**: Runs multiple fuzz targets simultaneously in background processes
- **Infinite loop**: Each target continuously restarts after completion/crash
- **Process management**: Tracks PIDs and handles clean shutdown via traps

### Key Functions
- `discover_fuzz_targets()`: Uses grep to find all `func Fuzz*` in test files (infinite-fuzz.sh:58)
- `fuzz_target()`: Infinite loop wrapper around `go test -fuzz` (infinite-fuzz.sh:156)
- `kill_fuzzing()`: Kills all running fuzz processes by name pattern (infinite-fuzz.sh:9)
- `cleanup()`: Trap handler for SIGINT/SIGTERM (infinite-fuzz.sh:122)

### Important Design Decisions
- Uses `set -u` but NOT `set -e` - expects test failures (infinite-fuzz.sh:3)
- GOEXPERIMENT environment variable is explicitly supported (infinite-fuzz.sh:165)
- Uses `pkill -9` for forceful cleanup to ensure all processes die (infinite-fuzz.sh:19-35)
- Small 1-second sleep between runs to avoid CPU hammering (infinite-fuzz.sh:176)

## Testing the Script

Since this script is meant for use in other projects:

```bash
# Manual testing approach
# 1. Create a test directory with sample fuzz tests
mkdir -p test
cat > test/example_test.go <<'EOF'
package test

import "testing"

func FuzzExample(f *testing.F) {
    f.Add("test")
    f.Fuzz(func(t *testing.T, s string) {
        if len(s) > 0 {
            // Simple fuzz test
        }
    })
}
EOF

# 2. Copy script to test directory
cp infinite-fuzz.sh test/

# 3. Run fuzzing
cd test && ./infinite-fuzz.sh

# 4. In another terminal, kill it
cd test && ./infinite-fuzz.sh -k
```

## Common Development Tasks

### Testing Script Changes
After modifying `infinite-fuzz.sh`:
1. Ensure script is executable: `chmod +x infinite-fuzz.sh`
2. Verify shellcheck passes: `shellcheck infinite-fuzz.sh` (if available)
3. Test auto-discovery in a real Go project with fuzz tests
4. Test kill flag functionality: start fuzzing, then run `./infinite-fuzz.sh -k`
5. Test Ctrl+C handling (should cleanly kill all background processes)

### Script Modifications
When modifying process management:
- Always test PID tracking and cleanup (pids array at infinite-fuzz.sh:119)
- Verify trap handling works for SIGINT and SIGTERM (infinite-fuzz.sh:153)
- Test both graceful (`kill`) and force (`kill -9`) termination paths

## Key Constraints

- **Bash 3.2+ compatibility**: Script must work on macOS default bash
- **No external dependencies**: Only bash and standard Unix tools (grep, sed, pkill)
- **Portability**: Must work identically across macOS and Linux
- **Go 1.18+ requirement**: Native fuzzing requires Go 1.18 or later
