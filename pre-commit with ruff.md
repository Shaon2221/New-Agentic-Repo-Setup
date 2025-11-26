To enforce **Ruff** for both linting and formatting in your repository using `pre-commit`, you need to configure the `.pre-commit-config.yaml` file to use the official Ruff hooks.

Here is the step-by-step implementation guide.

### 1\. Install pre-commit

If you haven't already, install the `pre-commit` package in your local environment.

```bash
pip install pre-commit
```

### 2\. Create or Update `.pre-commit-config.yaml`

Create a file named `.pre-commit-config.yaml` in the root of your repository (or append to it if it exists). Add the following configuration to use both the **linter** and the **formatter**.

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.6
    hooks:
      - id: ruff
        args: [ --fix ]
      - id: ruff-format

  - repo: https://github.com/astral-sh/uv-pre-commit
    rev: 0.9.12
    hooks:
      - id: uv-lock

  - repo: local
    hooks:
      - id: pyright
        name: pyright
        entry: uv run pyright
        language: system
        types: [python]
        pass_filenames: false
        always_run: true

  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: uv run pytest
        language: system
        pass_filenames: false
        always_run: true
```

  * **`id: ruff`**: This runs the linter (replaces Flake8, isort, etc.). The `args: [ --fix ]` line tells it to automatically fix simple issues (like unused imports) instead of just complaining about them.
  * **`id: ruff-format`**: This runs the formatter (replaces Black). It rewrites your code to match the standard style.

### 3\. Install the Git Hook

Run this command to "activate" the config. This installs a git hook script into your `.git/hooks/` directory.

```bash
pre-commit install
```

Now, every time you run `git commit`, Ruff will automatically run on the staged files.

### 4\. How to Use & Verify

#### Automatic Usage (On Commit)

When you commit changes, `pre-commit` will intercept the process.

  * If Ruff fixes a file (e.g., reformats it), the commit will **fail** so you can review the changes.
  * You simply need to `git add` the changes Ruff made and run `git commit` again.

#### Manual Usage (Run on All Files)

To check your entire repository immediately without committing (great for first-time setup), run:

```bash
pre-commit run --all-files
```

### Summary of Commands

| Goal | Command |
| :--- | :--- |
| **Install** | `pip install pre-commit` |
| **Activate** | `pre-commit install` |
| **Run Manually** | `pre-commit run --all-files` |
| **Update Versions** | `pre-commit autoupdate` |

