+++
date = '2025-11-11T17:15:00-05:00'
draft = false
title = 'Python Environment Managers: pip + venv vs poetry + pyenv vs uv'
+++

If you've been writing Python for a while, you've probably felt the pain of dependency hell. Your project works on your machine, breaks on your colleague's, and then explodes in production. The culprit? Inconsistent Python environments and dependency management.

There are several ways to handle this. Let me walk you through the main approaches, their trade-offs, and why I've landed on `uv` as my go-to tool.

---

## What Are These Tools?

Before we compare, let's clarify what we're actually talking about. There are two separate concerns here:

**Environment Managers** handle which Python version you're using. They let you have Python 3.10 for one project and Python 3.12 for another without conflicts.

**Package Managers/Dependency Resolvers** handle your project dependencies. They install packages, track versions, and ensure reproducibility across machines.

Most setups combine both. Here's what we're comparing:

- **pip + venv**: The standard library approach. Minimal, built-in, but manual.
- **poetry + pyenv**: The "professional" setup. Opinionated, feature-rich, battle-tested.
- **uv**: The new kid. Fast, simple, and refreshingly straightforward.

---

## The Contenders

### pip + venv

**What it is**: `venv` is Python's built-in virtual environment tool. `pip` is the standard package installer.

**Pros**:
- Zero external dependencies. It's all in the standard library.
- Simple to understand. Create a venv, activate it, install packages.
- Lightweight. No extra tooling to learn.
- Works everywhere Python works.

**Cons**:
- Manual version management. You have to download Python versions yourself.
- No lock file by default. `requirements.txt` doesn't guarantee reproducibility.
- Slow dependency resolution. pip doesn't always find the best version combinations.
- Tedious workflow. Activate/deactivate venvs manually.

**Setup**:
```bash
python3.10 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
```

**Lock file**: `requirements.txt` (manual, not guaranteed to be reproducible)

---

### poetry + pyenv

**What it is**: `pyenv` manages Python versions. `poetry` handles dependencies and project packaging.

**Pros**:
- Automatic Python version management. `.python-version` file keeps everyone on the same version.
- Lock files are first-class. `poetry.lock` guarantees reproducibility.
- Dependency resolution is smart. Poetry understands complex version constraints.
- Great for publishing packages. Poetry handles versioning and publishing to PyPI.
- Isolated by default. Each project gets its own environment automatically.

**Cons**:
- Steep learning curve. Multiple tools, multiple concepts.
- Slower than you'd expect. Dependency resolution can take time on large projects.
- Opinionated structure. Poetry expects a certain project layout.
- Overkill for simple scripts. Adding complexity for small projects.
- Configuration friction. `pyproject.toml` has a lot of options.

**Setup**:
```bash
# Install pyenv
brew install pyenv  # or your OS equivalent

# Install Python version
pyenv install 3.12.0
pyenv local 3.12.0

# Install poetry
curl -sSL https://install.python-poetry.org | python3 -

# Create project
poetry new my_project
cd my_project
poetry add requests
poetry install
```

**Lock file**: `poetry.lock` (guaranteed reproducibility)

**File size**: Typically larger lock files due to detailed metadata.

---

### uv

**What it is**: A single tool written in Rust that handles both Python version management and package management. Think of it as "poetry but faster and simpler."

**Pros**:
- Blazingly fast. Dependency resolution is orders of magnitude faster than poetry.
- Simple mental model. One tool does both jobs.
- Lock files included. `uv.lock` is reproducible by default.
- Minimal configuration. Works out of the box with sensible defaults.
- Transparent. You understand what's happening under the hood.
- Compatible with standard tools. Uses `pyproject.toml`, works with pip-style workflows.

**Cons**:
- Newer tool. Less battle-tested than poetry (though rapidly improving).
- Smaller ecosystem. Fewer integrations and plugins.
- Still evolving. APIs and behavior might change.

**Setup**:
```bash
# Install uv (one tool for everything)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create project
uv init my_project
cd my_project

# Add dependencies
uv add requests

# Run Python with the right environment
uv run python script.py

# Or activate the venv
source .venv/bin/activate
```

**Lock file**: `uv.lock` (guaranteed reproducibility, compact format)

---

## Practical Comparison

### Dependency Resolution Speed

I ran a benchmark with 51 common packages:

| Tool | Time | Notes |
|------|------|-------|
| pip | ~45-60s | Often fails to resolve complex dependencies |
| poetry | ~0.9s | Fast lock generation, but slow initial setup |
| uv | ~0.33s | 3x faster than poetry, 150x faster than pip |

uv's Rust implementation makes a massive difference. Poetry is actually quite fast at lock generation, but uv is still significantly faster.

### Lock File Size

For the same 51 packages:

| Tool | Size | Format |
|------|------|--------|
| pip (requirements.txt) | ~2.5KB | Plain text, not reproducible |
| poetry (poetry.lock) | ~636KB | Detailed metadata, reproducible |
| uv (uv.lock) | ~12KB | Compact TOML, reproducible |

uv's lock file is 50x smaller than poetry's while maintaining full reproducibility.

### Ease of Setup

**pip + venv**: Simplest to start, but manual and error-prone at scale.

**poetry + pyenv**: Most powerful, but requires understanding multiple tools and concepts.

**uv**: Sweet spot. One command to set up, one command to run code.

```bash
# uv: One tool, one command
uv init
uv add requests
uv run python main.py

# poetry + pyenv: Multiple steps, multiple tools
pyenv install 3.12.0
pyenv local 3.12.0
poetry new project
poetry add requests
poetry run python main.py
```

---

## Why I Chose uv

I've used all three approaches in production. Here's why uv is my default now:

**1. Transparent**: I understand exactly what's happening. No magic, no hidden complexity.

**2. Fast**: Dependency resolution is so fast it's almost instant. This matters when you're iterating.

**3. Simple**: One tool, one mental model. No context switching between pyenv and poetry.

**4. Good defaults**: Works well for new projects without configuration overhead.

**5. Compatible**: Uses standard `pyproject.toml`, plays nicely with existing Python tooling.

**6. Pragmatic**: It's opinionated but not dogmatic. You can still use pip if you want.

---

## When to Use What

**Use pip + venv if**:
- You're writing a simple script.
- You're in an environment where external tools aren't allowed.
- You're just learning Python.

**Use poetry + pyenv if**:
- You have a large, stable project with complex dependencies.
- You're publishing packages to PyPI.
- Your team is already invested in the poetry ecosystem.
- You need advanced features like dependency groups or custom repositories.

**Use uv if**:
- You're starting a new project.
- You want simplicity without sacrificing reproducibility.
- You value speed and transparency.
- You're tired of environment management friction.

---

## The Migration Question

If you have a stable poetry + pyenv setup that's working well, don't migrate just for the sake of it. The pain of migration usually outweighs the benefits.

But if you're starting something new or your current setup feels cumbersome, try uv. The onboarding is smooth, and you'll appreciate the simplicity.

```bash
# If you want to try uv with an existing project
uv init
uv add -r requirements.txt  # or migrate from poetry.lock
```

---

## Final Thoughts

Python's dependency management has come a long way. We're past the days of `pip freeze` and hope. Tools like poetry and uv have made reproducible environments the default, not the exception.

The best tool is the one your team understands and maintains consistently. But if you're choosing fresh, uv is worth a serious look. It's fast, simple, and just works.

Now go forth and never debug "it works on my machine" again.
