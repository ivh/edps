# Installing edps-gui

## The Problem

The `edpsgui` package ships a bash script as its entry point:

```bash
#!/bin/bash
# ...
app_dir=$(python -c "import edpsgui as _; print(_.__path__[0])")
panel serve "$app_dir"/edps-gui.py --show --plugins edpsgui.pdf_handler "$@"
```

This script calls `python` and `panel` directly, assuming they exist in PATH. This breaks in isolated environments (uv, pipx) because:

1. macOS doesn't have `python` in PATH (only `python3`)
2. `panel` is installed inside the virtual environment, not globally

## The Proper Fix (Upstream)

The package should use a Python entry point instead of a bash script. In `pyproject.toml`:

```toml
[project.scripts]
edps-gui = "edpsgui.cli:main"
```

With a corresponding `edpsgui/cli.py`:

```python
import subprocess
import sys
from pathlib import Path

def main():
    app_config = Path.home() / ".edps" / "application.properties"
    log_config = Path.home() / ".edps" / "logging.yaml"

    if subprocess.run(["which", "edps"], capture_output=True).returncode != 0:
        print("edps command not found in PATH")
        sys.exit(1)

    if not app_config.exists() or not log_config.exists():
        subprocess.run(["edps"])

    import edpsgui
    app_dir = Path(edpsgui.__path__[0])

    sys.exit(subprocess.run([
        sys.executable, "-m", "panel", "serve",
        str(app_dir / "edps-gui.py"),
        "--show", "--plugins", "edpsgui.pdf_handler",
        *sys.argv[1:]
    ]).returncode)
```

This is the correct approach because:

- **`sys.executable`** always points to the Python that's running the script (the venv's Python)
- **`python -m panel`** invokes panel from the same environment, no PATH lookup needed
- **pip/uv generate the wrapper** with the correct shebang pointing to the venv's Python
- **Works everywhere**: pip, pipx, uv, conda, system installs

Additionally, `edpsgui` should declare its actual dependencies in `pyproject.toml`:

```toml
[project]
dependencies = [
    "edps",
    "edpsplot",
    "adari_core",
    "panel",
    # ... other deps
]
```

Currently users must manually specify `-w edpsplot -w adari_core` and have `edps` installed separately. Proper dependency declaration would make installation a simple `uv tool install edpsgui` or `pip install edpsgui`.

Until upstream fixes this, use one of the workarounds below.

## Option A: Install Script (Global Install)

Install globally with `uv tool install`, then patch the wrapper script.

```bash
#!/bin/bash
set -e

uv tool install --index https://ftp.eso.org/pub/dfs/pipelines/libraries/ \
  -w edpsplot -w adari_core edpsgui

VENV_BIN="$HOME/.local/share/uv/tools/edpsgui/bin"

cat > ~/.local/bin/edps-gui << EOF
#!/bin/bash
app_config=\$HOME/.edps/application.properties
log_config=\$HOME/.edps/logging.yaml

if ! command -v edps >/dev/null 2>&1; then
  echo "edps command not found in PATH"
  exit 1
fi

if [ ! -f "\$app_config" ] || [ ! -f "\$log_config" ]; then
  edps
fi

app_dir=\$("$VENV_BIN/python" -c "import edpsgui as _; print(_.__path__[0])")
"$VENV_BIN/panel" serve "\$app_dir"/edps-gui.py --show --plugins edpsgui.pdf_handler "\$@"
EOF

chmod +x ~/.local/bin/edps-gui
echo "Installed edps-gui"
```

**Pros:**
- Global command, works from anywhere
- Familiar "install once, use anywhere" pattern

**Cons:**
- Requires patching after install
- Must re-patch after `uv tool upgrade`

## Option B: Project Environment (uv run)

Create a `pyproject.toml` and use `uv run` to execute within the virtual environment.

```toml
[project]
name = "edps-env"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "edpsgui",
    "edpsplot",
    "adari_core",
]

[[tool.uv.index]]
url = "https://ftp.eso.org/pub/dfs/pipelines/libraries/"
```

Then run:
```bash
uv sync
uv run edps-gui
```

This works because `uv run` prepends the venv's bin directory to PATH, so `python` and `panel` resolve correctly.

**Pros:**
- No patching required
- Standard uv workflow
- Updates via `uv sync`

**Cons:**
- Not a global command; must `cd` to project directory or use `--project`
- Semantically a "project" rather than a "tool"

**Tip:** Add an alias for convenience:
```bash
alias edps-gui='uv run --project ~/path/to/edps-env edps-gui'
```

## Comparison

| Aspect | Install Script | uv run |
|--------|----------------|--------|
| Global command | Yes | No (needs alias) |
| Patching needed | Yes | No |
| User maintains files | No | Yes (toml + lockfile) |
| Updates | `uv tool upgrade` + re-patch | `uv sync` |
