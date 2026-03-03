# Setup Environment

Automated environment setup for pypto-lib development. Detects platform, clones/updates
dependency repos (pypto, PTOAS, simpler), installs Python packages, and validates the
environment by running an example.

## Prerequisites

- Git
- Python 3.10+
- `python3 -m pip` (do NOT use bare `pip3` — it may point to a different Python)
- Network access to GitHub

## Workflow

Execute each step sequentially. Skip steps whose preconditions are already satisfied.

### Step 1: Detect Platform

Run the following and record the results for later use (PTOAS wheel matching):

```bash
uname -s          # Darwin → macos, Linux → linux
uname -m          # arm64 / x86_64
python3 --version # e.g. Python 3.11.x → PY_VER=cp311
```

Derive the wheel platform tag:
- macOS arm64  → `macosx_*_arm64`
- macOS x86_64 → `macosx_*_x86_64`
- Linux aarch64 → `manylinux_*_aarch64`
- Linux x86_64  → `manylinux_*_x86_64`

### Step 2: Clone / verify pypto

```bash
WORKSPACE_DIR="$(cd "$(dirname "$PWD")" && pwd)"   # parent of current repo

if [ ! -d "$WORKSPACE_DIR/pypto" ]; then
    git clone https://github.com/hw-native-sys/pypto.git "$WORKSPACE_DIR/pypto"
fi
```

If the directory exists, verify the origin remote points to the correct URL:

```bash
cd "$WORKSPACE_DIR/pypto"
git remote get-url origin   # expect https://github.com/hw-native-sys/pypto.git
```

### Step 3: Check pypto installation & decide whether to update

```bash
python3 -m pip show pypto 2>/dev/null
```

- If pypto is **already installed** and its Location matches the active Python, **ask the
  user** whether they want to pull the latest code and reinstall. If the user says no,
  **skip Step 3a and Step 3b** and proceed to Step 4.
- If pypto is **not installed** (or Location points to a different Python), proceed with
  Step 3a and Step 3b.

### Step 3a: Pull latest pypto main

```bash
cd "$WORKSPACE_DIR/pypto"
git fetch origin
git checkout main
git pull origin main
```

### Step 3b: Install pypto

**Important:** This step requires full permissions (`required_permissions: ["all"]`) because
the build involves CMake/Ninja compilation. Also clean the `build/` directory first to avoid
stale CMake cache conflicts:

```bash
cd "$WORKSPACE_DIR/pypto"
rm -rf build/
python3 -m pip install -e .
```

### Step 4: Clone / verify PTOAS

```bash
if [ ! -d "$WORKSPACE_DIR/PTOAS" ]; then
    git clone https://github.com/zhangstevenunity/PTOAS.git "$WORKSPACE_DIR/PTOAS"
fi
```

### Step 5: Install ptoas from release wheel

```bash
python3 -m pip show ptoas 2>/dev/null
```

If not installed, download and install the matching wheel from
`https://github.com/huawei-csl/PTOAS/releases`.

**Important:** This step requires full permissions (`required_permissions: ["all"]`) for
both the download and the `pip install`.

Use the helper script for automated download:

```bash
bash .claude/skills/setup_env/scripts/setup_env.sh install-ptoas
```

Or manually:

1. List latest release assets:
   ```bash
   gh release view --repo huawei-csl/PTOAS --json assets -q '.assets[].name'
   ```
2. Pick the wheel matching `cp{PY_VER}` + `{os}_{arch}` from Step 1.
3. Download and install:
   ```bash
   gh release download --repo huawei-csl/PTOAS -p '<matched_wheel_name>' -D /tmp
   python3 -m pip install /tmp/<matched_wheel_name>
   ```

**Slow download?** The wheel is ~45 MB. If the download speed is very slow (< 50 KB/s)
or the command hangs for more than 2 minutes, **stop the download and ask the user to
manually download the wheel** from `https://github.com/huawei-csl/PTOAS/releases` to
their `~/Downloads` folder. Then install from there:

```bash
python3 -m pip install ~/Downloads/<matched_wheel_name>
```

### Step 6: Clone / verify simpler

```bash
if [ ! -d "$WORKSPACE_DIR/simpler" ]; then
    git clone https://github.com/ChaoWao/simpler.git "$WORKSPACE_DIR/simpler"
fi
```

### Step 7: Checkout simpler to stable branch

```bash
cd "$WORKSPACE_DIR/simpler"
git fetch origin
git checkout stable
git pull origin stable
```

### Step 8: Validate environment

Run the example from the pypto-lib working directory:

```bash
cd "$WORKSPACE_DIR/pypto-lib"   # or the current workspace root
python3 examples/paged_attention_example.py
```

Expected output includes:
- `[1] IR Preview:` followed by IR text
- `[2] Compiling...`
- `Output: <path>`
- `[3] Generated files:` followed by a file listing

If the script completes without errors, the environment is correctly configured.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `ModuleNotFoundError: No module named 'pypto'` | Re-run Step 3b — make sure `python3 -m pip` matches the active `python3` |
| `ModuleNotFoundError: No module named 'ptoas'` | Re-run Step 5 — check that the wheel matches your platform |
| `pip3 show pypto` shows wrong Location (e.g. Python 3.9 path) | Uninstall first (`python3 -m pip uninstall pypto`) then reinstall |
| `gh: command not found` | Install GitHub CLI or use `curl` with the GitHub API instead |
| Git clone fails with permission denied | Check SSH keys or switch to HTTPS URL |
| ptoas wheel download is extremely slow | Ask user to download manually from GitHub releases to `~/Downloads`, then `python3 -m pip install ~/Downloads/<wheel>` |
| `Wheel ... is invalid` after download | Incomplete download — delete the file and re-download |
