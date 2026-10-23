---
name: dockerfile-generator
description: Generate production-ready Dockerfiles, .dockerignore files, and optional compose.yaml for 15 languages. Use when containerizing an app, refreshing a Dockerfile, or adding a monorepo build.
license: MIT
compatibility: Requires bash 4.4+, python3 3.10+, and the Python packages click, PyYAML, and Jinja2. Docker, hadolint, and shellcheck are optional.
metadata:
  version: "1.0.0"
  author: "Mohamed Rafaa TOUNEKTI"
  upstream: "https://github.com/rafaabou/docker-file-generator"
  upstream-commit: "2c0fbaf"
---

# Dockerfile Generator

Generate Dockerfiles from the project itself, not from a blank template. The
bundled runtime detects the language, framework, package manager, port, and
health endpoint, then writes a multi-stage `Dockerfile-1` and, where supported,
a release-asset `Dockerfile-2`, plus the matching `.dockerignore` files.

## When to use

Use this skill when the user asks to:

- containerize an application or add a Dockerfile
- create or refresh a production multi-stage build
- generate `compose.yaml` for an app and its backing services
- add Dockerfiles to a monorepo
- check whether an existing `Dockerfile-1` still matches the project
- work with Python, Go, Node.js/SPA, Rust, Java, Kotlin, Ruby, C#, PHP,
  Swift, Scala, Elixir, Clojure, or Haskell

Run the generator before hand-writing a Dockerfile. Inspect its output first,
then make deliberate project-specific edits only when the detected result needs
them.

## Quick start

Let `SKILL_DIR` be the directory containing this `SKILL.md`:

```bash
SKILL_DIR=/path/to/dockerfile-generator

# Preview, write nothing
bash "$SKILL_DIR/scripts/generate.sh" --project /path/to/app --dry-run

# Generate Dockerfile-1 only, skipping Docker/hadolint verification
bash "$SKILL_DIR/scripts/generate.sh" --project /path/to/app \
  --no-dockerfile-2 --fast

# Full default run (may also write Dockerfile-2)
bash "$SKILL_DIR/scripts/generate.sh" --project /path/to/app
```

- `--no-dockerfile-2` writes only Dockerfile-1 and its `.dockerignore`.
- `--fast` skips Docker build and hadolint checks. Use it when Docker is
  unavailable or the user only wants files.
- Without `--fast`, the generator may run Docker/hadolint verification. That is
  slower and requires those tools.
- Without `--no-dockerfile-2`, the generator may also write Dockerfile-2 and
  `.dockerignore-2`.

The wrapper prints the resolved project path and the runtime it is using, then
propagates the generator's exit code. A nonzero exit means generation or
verification failed; read stderr before retrying.

## Common commands

| Goal | Command (after `--project /path/to/app`) |
| --- | --- |
| Preview without writing | `--dry-run` |
| Dockerfile-1 only | `--no-dockerfile-2 --fast` |
| Full generated output | no extra flags |
| Also write `compose.yaml` | `--compose` |
| Monorepo, one app at a time | `--all` |
| Verify Dockerfile-1 matches detection | `--check` |
| Create a starter `dockerfile.conf` | `--init` |
| Use a specific config file | `--config dockerfile.yaml` |
| Ignore all config files | `--no-config` |
| Validate configuration only | `--validate-config` |
| Print resolved config as JSON | `--print-config` |
| Explain one setting | `--explain defaults.port` |
| Show the upstream full help | `--upstream-help` |

`--all` cannot be combined with `--compose`; run `--compose` separately in each
application directory. `--dry-run` and `--check` cannot be combined.

## Configuration

The generator reads, in increasing precedence:

1. built-in registry defaults
2. the bundled `vendor/config/defaults.yaml`
3. `dockerfile.conf` (legacy, only when no `dockerfile.yaml` exists)
4. `dockerfile.yaml` in the project
5. environment variables
6. CLI flags

Common environment overrides:

```bash
PORT=8080 HEALTH=/ready STOPSIGNAL=SIGTERM COMPOSE_BIND=127.0.0.1 \
  bash "$SKILL_DIR/scripts/generate.sh" --project /path/to/app --fast
```

Other supported variables include `EXTRA_COPY`, `EXTRA_RUN`,
`EXTRA_DIRECTIVES`, `D2_ASSET_URL`, and `D2_ASSET_SHA_URL`. See
`references/FLAGS.md` for details.

## Output and verification

A successful run normally writes:

- `Dockerfile-1` - builds from the project source
- `Dockerfile-2` - installs from a release/package registry, when supported
- `.dockerignore` or `.dockerignore-1` / `.dockerignore-2`
- `compose.yaml` only when `--compose` is used

After generation:

1. Inspect the files and confirm the base image, build stages, exposed port,
   health check, and runtime user are appropriate.
2. Check that no existing user files were overwritten unexpectedly.
3. Run `hadolint Dockerfile-1` if hadolint is available.
4. Run the project's build or tests when the environment allows it.
5. Use `--check` to confirm Dockerfile-1 still matches current detection.

`--check` verifies detection output; it does not lint arbitrary hand edits.
Use hadolint and the project's own tests for that.

## Safety rules for the agent

- Run `git status` and `--dry-run` before generating in a repository that may
  already contain `Dockerfile-1`, `Dockerfile-2`, `.dockerignore*`, or
  `compose.yaml`. The generator can overwrite generated paths.
- Do not silently overwrite a user's hand-written Dockerfile. If one exists,
  show the diff or ask before replacing it.
- Do not modify `vendor/`. It is a pinned runtime snapshot covered by this
  package's MIT license; update it only through a deliberate snapshot refresh.
- Do not add secrets, private registry credentials, or host-specific paths to
  generated files.
- Use `--fast` when Docker is unavailable, but tell the user that build and
  lint verification were skipped.
- Generated Compose files bind to `127.0.0.1` by default. Set
  `COMPOSE_BIND=0.0.0.0` only when remote access is intentional.

## Supported agents

The same `SKILL.md` works with the Agent Skills layout used by Claude Code,
OpenAI Codex, Cursor, GitHub Copilot, Google Antigravity, Devin Desktop,
OpenCode, Cline, and Kiro. Use `scripts/install.sh` to place the skill in the
right directory, or copy the whole folder manually. See `README.md` for the
full path matrix.

For tools that do not load `SKILL.md`, use `adapters/AGENTS.md` (generic),
`adapters/copilot-instructions.md` (older Copilot), `adapters/cline-rule.md`
(older Cline), or `adapters/kiro-steering.md` (older Kiro).

## Troubleshooting

- **`python3` missing** - install Python 3.10 or newer.
- **Missing Python packages** - run
  `python3 -m pip install -r "$SKILL_DIR/requirements.txt"`.
- **Docker/hadolint unavailable** - use `--fast`; verification is skipped.
- **Wrong language detected** - add the expected marker file (for example
  `go.mod` or `requirements.txt`) or use a `dockerfile.yaml` override; run
  `--dry-run` to inspect detection before writing.
- **Dockerfile-2 missing** - not every language/target supports it; run
  `--no-dockerfile-2` if Dockerfile-1 is all that is needed.
- **Path with spaces** - quote the path; the wrapper handles spaces.
- **`--check` reports drift** - the existing Dockerfile-1 no longer matches
  detection. Review the diff and regenerate, or keep the hand-written file and
  document why.

More detail: `references/FLAGS.md`, `references/LANGUAGES.md`,
`references/TROUBLESHOOTING.md`, and `references/UPSTREAM.md`.
