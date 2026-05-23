# Repository Scaffolding — Reference

All templates use placeholders. Substitute before writing.

## Project Templates

### CLAUDE.md

```markdown
# Coding Style

## General

- No trailing whitespace
- Files end with a single newline
- UTF-8 encoding everywhere
- Lines max 120 chars (code), 120 chars (prose/markdown)

## Naming

- Files and directories: kebab-case
- Branches: `dev/<username>/<change>` with kebab-case change name
- Commits: Conventional Commits format, subject ≤50 chars, body lines ≤72 chars

## Shell (bash)

- `set -uo pipefail` (avoid `set -e` in scripts with arithmetic or grep)
- Use `[[` not `[` for conditionals
- Quote all variable expansions: `"$var"` not `$var`

## Git

- Conventional Commits: `feat`/`fix`/`refactor`/`docs`/`test`/`chore`/`ci`/`build`/`perf`/`style`
- Imperative mood in subject line, lowercase after prefix
- No trailing period on subject
- Body explains why, not what

## Tool Preferences

- **Containers**: podman, never docker. Use `Containerfile`, never `Dockerfile`
- **VCS**: git
```

Customize the Tool Preferences section based on the chosen language. Add language-specific sections as needed.

After writing CLAUDE.md, create the symlink:

```bash
ln -s CLAUDE.md AGENTS.md
```

### .editorconfig

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 4

[*.go]
indent_style = tab

[*.md]
trim_trailing_whitespace = false

[Makefile]
indent_style = tab
```

### .gitignore — Go

```gitignore
# Binaries
*.exe
*.exe~
*.dll
*.so
*.dylib
bin/
dist/

# Test
*.test
*.out
coverage.txt
coverage.html

# Debugger
__debug_bin*

# Dependencies
vendor/

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Environment
.env
.env.local
```

### .gitignore — Rust (Service/CLI)

```gitignore
# Build
/target/

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Environment
.env
.env.local
```

### .gitignore — Rust (Library)

Same as Service/CLI, plus:

```gitignore
# Build
/target/

# Lock file — consumers manage their own
Cargo.lock

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Environment
.env
.env.local
```

### .gitignore — Python (Service/CLI)

```gitignore
# Virtual environments
.venv/
venv/
__pycache__/
*.py[cod]
*$py.class
*.pyo

# Distribution
dist/
build/
*.egg-info/
*.egg

# Testing
.pytest_cache/
.coverage
htmlcov/
.mypy_cache/
.ruff_cache/

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Environment
.env
.env.local
```

### .gitignore — Python (Library)

Same as Service/CLI, plus `uv.lock`:

```gitignore
# Virtual environments
.venv/
venv/
__pycache__/
*.py[cod]
*$py.class
*.pyo

# Distribution
dist/
build/
*.egg-info/
*.egg

# Testing
.pytest_cache/
.coverage
htmlcov/
.mypy_cache/
.ruff_cache/

# Lock file — consumers manage their own
uv.lock

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Environment
.env
.env.local
```

### CODEOWNERS

For GitHub (org repo):

```text
* @<org>/<team>
```

For GitHub (personal repo):

```text
* @<username>
```

For GitLab, same syntax but uses GitLab groups/users. Ask user for the owner value.

### LICENSE — MIT

```text
MIT License

Copyright (c) <year> <author>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

### LICENSE — Apache-2.0

Use the full Apache 2.0 text from <https://www.apache.org/licenses/LICENSE-2.0.txt> — it is too long to inline here.
Fetch it at generation time:

```bash
curl -sL https://www.apache.org/licenses/LICENSE-2.0.txt > LICENSE
```

## CI Templates

### Resolving Action SHAs

All GitHub Actions templates use `<sha>` placeholders. At generation time, resolve each action to its latest tag SHA:

```bash
# Get SHA for a tagged release
gh api repos/{owner}/{action}/git/ref/tags/{tag} --jq '.object.sha'

# Examples
gh api repos/actions/checkout/git/ref/tags/v6.0.2 --jq '.object.sha'
gh api repos/actions/setup-go/git/ref/tags/v6.4.0 --jq '.object.sha'
gh api repos/golangci/golangci-lint-action/git/ref/tags/v9.2.1 --jq '.object.sha'
gh api repos/astral-sh/setup-uv/git/ref/tags/v8.1.0 --jq '.object.sha'
gh api repos/actions/cache/git/ref/tags/v5.0.5 --jq '.object.sha'
gh api repos/actions/dependency-review-action/git/ref/tags/v5.0.0 --jq '.object.sha'
```

Always look up the latest release first with `gh api repos/{owner}/{action}/releases/latest --jq '.tag_name'`,
then resolve its SHA. Never copy SHAs from these templates without verifying — they go stale.

### GitHub Actions — Go

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@<sha> # <version>

      - name: Set up Go
        uses: actions/setup-go@<sha> # <version>
        with:
          go-version-file: go.mod

      - name: Run tests
        run: go test -race -coverprofile=coverage.txt ./...

  lint:
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@<sha> # <version>

      - name: Set up Go
        uses: actions/setup-go@<sha> # <version>
        with:
          go-version-file: go.mod

      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@<sha> # <version>
        with:
          version: latest
```

### GitHub Actions — Rust

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@<sha> # <version>

      - name: Set up Rust toolchain
        run: rustup toolchain install stable --profile minimal --component clippy rustfmt

      - name: Cache cargo registry and build
        uses: actions/cache@<sha> # <version>
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

      - name: Run tests
        run: cargo test --workspace

  lint:
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@<sha> # <version>

      - name: Set up Rust toolchain
        run: rustup toolchain install stable --profile minimal --component clippy rustfmt

      - name: Cache cargo registry and build
        uses: actions/cache@<sha> # <version>
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

      - name: Run clippy
        run: cargo clippy --workspace -- -D warnings

      - name: Check formatting
        run: cargo fmt --all -- --check
```

### GitHub Actions — Python

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@<sha> # <version>

      - name: Install uv
        uses: astral-sh/setup-uv@<sha> # <version>

      - name: Set up Python
        run: uv python install

      - name: Install dependencies
        run: uv sync --all-extras --dev

      - name: Run tests
        run: uv run pytest

  lint:
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@<sha> # <version>

      - name: Install uv
        uses: astral-sh/setup-uv@<sha> # <version>

      - name: Set up Python
        run: uv python install

      - name: Install dependencies
        run: uv sync --all-extras --dev

      - name: Run ruff check
        run: uv run ruff check .

      - name: Run ruff format check
        run: uv run ruff format --check .

      - name: Run mypy
        run: uv run mypy src/
```

### GitHub Actions — Dependency Review

```yaml
name: Dependency Review

on:
  pull_request:

permissions:
  contents: read

jobs:
  dependency-review:
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@<sha> # <version>

      - name: Dependency review
        uses: actions/dependency-review-action@<sha> # <version>
```

### GitLab CI — Go

```yaml
stages:
  - test
  - lint

variables:
  GOPATH: ${CI_PROJECT_DIR}/.go

cache:
  paths:
    - .go/pkg/mod/

test:
  stage: test
  image: golang:1.26
  script:
    - go test -race -coverprofile=coverage.txt ./...

lint:
  stage: lint
  image: golangci/golangci-lint:latest
  script:
    - golangci-lint run ./...
```

### GitLab CI — Rust

```yaml
stages:
  - test
  - lint

variables:
  CARGO_HOME: ${CI_PROJECT_DIR}/.cargo

cache:
  paths:
    - .cargo/registry/
    - .cargo/git/
    - target/

test:
  stage: test
  image: rust:1.85
  script:
    - cargo test --workspace

lint:
  stage: lint
  image: rust:1.85
  before_script:
    - rustup component add clippy rustfmt
  script:
    - cargo clippy --workspace -- -D warnings
    - cargo fmt --all -- --check
```

### GitLab CI — Python

```yaml
stages:
  - test
  - lint

variables:
  UV_CACHE_DIR: ${CI_PROJECT_DIR}/.uv-cache

cache:
  paths:
    - .uv-cache/

test:
  stage: test
  image: ghcr.io/astral-sh/uv:python3.12-bookworm
  script:
    - uv sync --all-extras --dev
    - uv run pytest

lint:
  stage: lint
  image: ghcr.io/astral-sh/uv:python3.12-bookworm
  script:
    - uv sync --all-extras --dev
    - uv run ruff check .
    - uv run ruff format --check .
    - uv run mypy src/
```

## Renovate

JSONC format in `.json` file. Place in `.github/renovate.json` (GitHub) or `.gitlab/renovate.json` (GitLab).

```jsonc
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  // Renovate parses JSONC natively from .json files
  "extends": ["config:best-practices"]
}
```

## Security Templates

### .gitleaks.toml

```toml
[extend]
useDefault = true

[allowlist]
description = "Global allowlist"
paths = [
    '''go\.sum''',
    '''vendor/''',
    '''\.git/'''
]
```

### .pre-commit-config.yaml

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.24.3
    hooks:
      - id: gitleaks
```

## Language Templates

### Go

#### go.mod

```go
module <module_path>

go 1.26
```

#### .golangci.yml

```yaml
run:
  timeout: 5m

linters:
  enable:
    - gofmt
    - govet
    - staticcheck
    - errcheck
    - ineffassign
    - unused
    - gosec
    - revive

linters-settings:
  gofmt:
    simplify: true
  revive:
    rules:
      - name: exported
        arguments:
          - checkPrivateReceivers

issues:
  exclude-use-default: false
```

#### Makefile

```makefile
.PHONY: build test lint fmt

build:
	go build -o bin/<project_name> ./cmd/<project_name>

test:
	go test -race ./...

lint:
	golangci-lint run ./...

fmt:
	gofmt -w .
```

For library type, remove the `build` target and its `.PHONY` entry.

#### Service/CLI Layout

```text
cmd/
  <project_name>/
    main.go
internal/
```

##### cmd/\<project_name\>/main.go

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	if err := run(); err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}
}

func run() error {
	return nil
}
```

#### Library Layout

No `cmd/` directory. Package at root or under a descriptive package name.

### Rust

#### Cargo.toml — Service/CLI

```toml
[package]
name = "<project_name>"
version = "0.1.0"
edition = "2024"
resolver = "3"
description = "<description>"
license = "MIT"
repository = "https://<module_path>"

[lints.clippy]
pedantic = { level = "warn", priority = -1 }
module_name_repetitions = "allow"
must_use_candidate = "allow"
dbg_macro = "warn"
unwrap_used = "warn"
undocumented_unsafe_blocks = "warn"
```

#### Cargo.toml — Library

Same as Service/CLI, plus:

```toml
[lib]
name = "<project_name>"
```

Use the crate name with hyphens converted to underscores for the `[lib] name` field.

#### src/main.rs (Service/CLI)

```rust
fn main() {
    if let Err(e) = run() {
        eprintln!("error: {e}");
        std::process::exit(1);
    }
}

fn run() -> Result<(), Box<dyn std::error::Error>> {
    Ok(())
}
```

#### src/lib.rs (Library)

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```

### Python

All Python templates use `<package_name>` (underscored form of `<project_name>`) for directory and import paths.

#### pyproject.toml

```toml
[project]
name = "<project_name>"
version = "0.1.0"
description = "<description>"
requires-python = ">=3.12"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/<package_name>"]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "ruff>=0.11",
    "mypy>=1.15",
]

[tool.ruff]
target-version = "py312"

[tool.ruff.lint]
select = ["E", "W", "F", "I", "UP", "B", "S"]

[tool.mypy]
python_version = "3.12"
strict = true
```

#### Python Service/CLI Layout

```text
src/
  <package_name>/
    __init__.py
    __main__.py
tests/
  conftest.py
  test_<package_name>.py
```

##### src/\<package_name\>/\_\_init\_\_.py

```python
```

Empty file — user fills in.

##### src/\<package_name\>/\_\_main\_\_.py

```python
import sys


def main() -> int:
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

##### tests/conftest.py

```python
```

Empty file — user adds fixtures.

##### tests/test\_\<package_name\>.py

```python
def test_placeholder() -> None:
    assert True
```

#### Python Library Layout

Same as Python Service/CLI but without `__main__.py`.
