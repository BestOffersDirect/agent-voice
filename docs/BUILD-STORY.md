# How agent-voice was built

Sources for this page: the repository's two commits, its CHANGELOG, the
INSTALL guide, the test suite and the GitHub Actions run history. No issue
tracker was used for this project.

## The problem and the first version (2026-08-27)

The starting point was a Claude Code session that should speak its replies in
a character voice without a browser in the loop. The first decision was to
keep two channels apart: the screen keeps the full technical reply and the
speaker gets one short line. A whole reply read aloud is slow and unpleasant,
and most of it (code, paths, tables) is not speakable.

The first commit, `dcd1bf4` at 23:00 on 2026-08-27, shipped version 0.1.0 as
27 files and 1,275 lines:

- the spoken-line cascade: an explicit `<say>...</say>` tag, then a writer
  model with the voice's persona, then the text itself with markup stripped;
- an OmniVoice `/synthesize` client that speaks the same contract as scorbo2's
  `server_omnivoice.py` and TalkWithMe, so an existing server works unchanged;
- three entry points (`speak`, a stdin `pipe`, and a standard-library HTTP
  server with `POST /say`);
- cross-platform playback and a voice library kept as plain directories;
- the Claude Code hooks, moved to `integrations/claude-code/` as one example
  host rather than a dependency;
- 17 tests across tags, writer, speaker and server.

## Key design decisions

- **Tag first, model second.** An agent that knows what is worth saying can
  write the `<say>` line itself, which costs nothing and is deterministic. The
  writer model is the fallback for agents that cannot be prompted. Tags quoted
  inside code spans are ignored so an agent can talk about the tag safely.
- **Never cut a sentence.** The writer's line is capped at
  `AGENT_VOICE_MAX_WORDS` by dropping whole sentences, so the voice never stops
  mid-word.
- **One dependency.** `httpx` is the only runtime dependency; the HTTP server
  is the standard library. The tool is meant to be dropped next to any agent
  without pulling in a framework.
- **Serialize utterances.** A lock in `Speaker` makes two agents take turns
  instead of talking over each other through one server.
- **Reuse an existing TTS contract.** Rather than define its own synthesis API,
  agent-voice speaks the `/health` + `/synthesize` JSON that existing OmniVoice
  servers already expose.

## Phase two: making it installable (same evening)

Sixteen minutes after the first commit, pull request #1 (`16ae5a5`) merged a
public-release pass: a rewritten README, a step-by-step `INSTALL.md` with a
troubleshooting table, `install.sh` (finds Python, uses uv when present, writes
the env file, can build a demo voice), an `agent-voice.env` file loader, and
`agent-voice doctor`, which checks every dependency and prints the fix for each
failure. The default TTS port moved from 8000 to 8181 to match the server most
people run.

The PR's own commits show what review caught before merge:

- `ruff format` was rewriting code blocks inside Markdown, so CI excluded
  `*.md` from formatting.
- `doctor` treated a missing audio player as fatal even when playback was off,
  and its tests depended on which players the CI host had installed. The fix
  made the player check conditional on `AGENT_VOICE_PLAY` and patched the
  lookup in tests.

## How it is tested

22 pytest tests (`tests/`) cover tag extraction, the writer's prompt and
sentence cap, the speaker cascade and config loading (including the env-file
precedence rule), the HTTP server's status codes, and `doctor`. `ruff`,
`ruff format --check` and `mypy --strict` run on every push. The GitHub
Actions matrix runs all of them on Ubuntu and macOS for Python 3.11, 3.12 and
3.13, plus a separate job that runs `install.sh`. The PR run and the merge run
on 2026-08-28 both passed.

## How AI was used

The tool exists to give an AI coding agent a voice, and it was built in the
same setting: the project grew out of a Claude Code session, and the Claude
Code hooks under `integrations/claude-code/` are the integration it was first
used with. The history does not record a separate multi-model review for this
project.

## Current state

Version 0.1.0, feature-complete for its scope. The CHANGELOG has an empty
Unreleased section. Open directions are more integrations and playback
backends, as listed in CONTRIBUTING.md.
