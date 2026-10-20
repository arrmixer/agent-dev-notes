# Python virtual environments on this Mac

A per-project cheat sheet for macOS + Homebrew Python. Written against this machine's
actual setup: Homebrew Python **3.14.6** at `/opt/homebrew/bin/python3`, no pyenv, no
conda, no poetry, no uv.

---

## Why Homebrew forces this on you

You're not imagining it. Homebrew's Python ships a marker file:

```
/opt/homebrew/Cellar/python@3.14/3.14.6/Frameworks/.../lib/python3.14/EXTERNALLY-MANAGED
```

That's [PEP 668](https://peps.python.org/pep-0668/). Its presence makes `pip install`
outside a virtual environment fail with:

```
error: externally-managed-environment
```

The reason is real, not bureaucratic: Homebrew itself installs formulae that depend on
that Python. If you `pip install` into it and a version conflicts, you can break unrelated
brew packages — and the breakage shows up somewhere confusing days later.

So: **one venv per project.** That's the intended workflow, not a workaround.

> There's a `--break-system-packages` flag that bypasses the error. Don't. It's named that
> way on purpose.

---

## The whole workflow — 4 commands

From the root of any project:

```bash
cd ~/Developer/my-project

python3 -m venv .venv                 # 1. create (once per project)
source .venv/bin/activate             # 2. activate (every new shell)
pip install requests                  # 3. install freely — you're isolated now
deactivate                            # 4. leave, when you're done
```

That's it. Everything below is refinement.

**How to tell it worked:** your prompt gains a `(.venv)` prefix, and:

```bash
which python    # → /Users/you/Developer/my-project/.venv/bin/python
```

Note that bare `python` and `pip` *work inside a venv* even though they don't exist
globally on this machine (globally it's `python3`/`pip3` only). The venv creates both
names. That's why tutorials saying `python script.py` suddenly start working.

---

## Conventions worth adopting

**Name it `.venv`, put it at the project root.** Not `venv`, not `env`. The leading dot
hides it, and it's the name every tool auto-detects — VS Code, PyCharm, `uv`, most CI.

**Always gitignore it.** A venv is ~50MB of platform-specific binaries and is fully
rebuildable from `requirements.txt`. Add to the project's `.gitignore`:

```
.venv/
__pycache__/
*.pyc
```

**Record dependencies, don't commit them.** Two schools:

```bash
pip freeze > requirements.txt          # exact pins, everything incl. transitive deps
```

Fine for coursework and apps you deploy — fully reproducible. The downside is you can't
tell what you actually asked for vs. what came along as a dependency. For anything
longer-lived, hand-maintain `requirements.txt` with just your direct deps (`boto3`,
`requests`) and let pip resolve the rest.

**Rebuilding on another machine (or after deleting `.venv`):**

```bash
python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt
```

**Upgrade pip in a fresh venv** if it complains — venvs are seeded with whatever pip
version shipped with the interpreter:

```bash
pip install --upgrade pip
```

---

## Do you need VS Code for this?

**It's not either/or, and VS Code isn't a replacement.** Its `Python: Create Environment`
command runs `python -m venv` under the hood — the same four commands above, behind a
menu. There's no separate "VS Code way" to learn.

What VS Code genuinely adds, once a `.venv` exists at the project root:

- **Auto-detection.** It finds `.venv` and offers to use it — no config needed.
- **Auto-activation in the integrated terminal**, so you skip `source .venv/bin/activate`.
- **Correct IntelliSense.** Autocomplete and go-to-definition resolve against *that
  project's* packages, which is the biggest day-to-day win.
- **Debugger + test runner** that use the right interpreter automatically.

**My suggestion:** learn the CLI commands as your foundation, and let VS Code be the
convenience layer on top. Reasons in that order:

1. The CLI always works — over SSH, in CI, in a Docker build, in a plain terminal, on a
   machine with no editor installed. The venv is a property of the *project*, not the
   editor.
2. When VS Code picks the wrong interpreter — and it eventually will — you can only debug
   it if you know what it was trying to do.
3. It's four commands. The abstraction isn't buying you much.

If the interpreter is ever wrong in VS Code: `Cmd+Shift+P` → **Python: Select
Interpreter** → pick the one whose path ends in `.venv/bin/python`. Note the `code` CLI
isn't on your PATH right now; if you want `code .` to work, run **Shell Command: Install
'code' command in PATH** from the same palette.

---

## Troubleshooting

| Symptom | Cause & fix |
|---|---|
| `error: externally-managed-environment` | You're not in a venv. `source .venv/bin/activate`. |
| `pip install` worked but `import` fails | Different venv active than you think, or you installed before activating. Check `which python` and `which pip` agree. |
| Renamed/moved the project folder, venv broke | Venvs hardcode absolute paths in `pyvenv.cfg` and `.venv/bin/*`. They are **not** relocatable — delete `.venv` and recreate. Same applies if you re-clone a repo. |
| `command not found: python` | Normal here — no bare `python` globally. Use `python3`, or activate a venv (which provides `python`). |
| Copied a `.venv` from another project | Don't. Same absolute-path problem. Create a fresh one and `pip install -r requirements.txt`. |
| Want to start over | `rm -rf .venv` and recreate. Nothing else to clean up — that's the appeal of venvs. |
| Which Python is a venv pinned to? | `cat .venv/pyvenv.cfg` — shows the version and the exact interpreter it was built from. |

---

## Two related tools, for when you hit their use case

**`pipx`** — for installing Python *applications* you want available everywhere
(`ruff`, `httpie`, `black`) rather than libraries for one project. It makes a hidden venv
per tool automatically, so you get global commands without polluting brew's Python.

```bash
brew install pipx && pipx ensurepath
pipx install ruff
```

Rule of thumb: **library you `import`** → project venv. **Command you type** → pipx.

**`uv`** — a much faster drop-in for venv+pip (`uv venv`, `uv pip install`). Worth a look
if venv creation and installs start feeling slow, but the workflow above is what it
replaces, so learn this first. Neither tool is installed right now.

---

## Quick reference

```bash
python3 -m venv .venv              # create
source .venv/bin/activate          # activate
deactivate                         # leave
pip install -r requirements.txt    # restore deps
pip freeze > requirements.txt      # record deps
cat .venv/pyvenv.cfg               # what Python is this?
which python                       # is my venv actually active?
rm -rf .venv                       # nuke and start over
```
