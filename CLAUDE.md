# CLAUDE.md

*How to work in this repository. What the code is, the code says; this file says
what the code cannot — which command to trust, what "done" means, and which
plausible-looking action is the wrong one.*

*Last updated: 2026-08-28*

---

## Project overview

Builds one Docker image: a hardened OpenSSH **bastion** — a jump host, and only
a jump host. `sshd_config` sets `ForceCommand /usr/sbin/nologin`, `MaxSessions
0` and `PermitTTY no`, so the container forwards connections and refuses to be
a shell. Optional TOTP/MFA via `libpam-google-authenticator`, optional SSH
certificate-authority mode.

**The repository is `dennisdeh/bastion`** (`origin` →
`https://github.com/dennisdeh/bastion`). It was `docker-bastion`, and the old
name still survives in tracked files that nothing has swept: the
`org.opencontainers.image.source` LABEL in `Dockerfile`, and four README links
under `github.com/dennisdeh/docker-bastion/…`. **Two spellings are correct and
must survive any rename sweep**: `gnzsnz/docker-bastion`, the project this was
forked from, and `binlab/docker-bastion` in the README's reference list —
somebody else's repository in both cases.

**This project is a fork by credit only.** `README.md` line 2 credits
`gnzsnz/docker-bastion`; that is the whole relationship. There is no remote for
it, no sync, no schedule. Judge every question on this tree.

### This repository is not where the published image comes from

**`dennisdeh/ib-docker` vendors this project at `bastion/`, that copy is
*ahead* of this one, and it is the copy that gets published.** Verified by diff
on 2026-08-28: `ib-docker/bastion/` contains this repository's `master` content
(including HEAD `55e962a`'s `lslogins` guard and `util-linux`) **plus**
changes that have never landed here —

| in `ib-docker/bastion/` | not here |
|---|---|
| `sshd_config_d.sh` + `check_sshd_config_d()` | closes the drop-in gap below |
| `ARG IMAGE_VERSION=2604.01` declared | fixes the broken version LABEL below |
| `LABEL …image.source=…/ib-docker` | ghcr package linkage |
| `pull_policy: build`, `container_name`, `IMAGE_VERSION` build arg in compose | |
| `hadolint ignore=DL3025` on `HEALTHCHECK` | |
| README 584 lines | 445 here |

Two consequences, and neither is optional:

- **Read `ib-docker/bastion/` before writing a fix here.** A bug you find in
  this tree has a fair chance of already being fixed there, with the reasoning
  in a comment. Re-deriving it is wasted work and will diverge in the details.
- **A change merged here does not reach anything that runs.**
  `ghcr.io/dennisdeh/bastion` is built and tagged by `ib-docker`'s
  `publish.yml`, from `ib-docker/bastion/Dockerfile`'s own `ARG IMAGE_VERSION`.
  This repository has no CI (see *Deployment and CI*). Port the change, or say
  in the commit that it was not ported.

- **Primary language / runtime:** Bash + Dockerfile. No application code.
- **Entry point (one-time):** `provision.sh`, run *inside* the image against a
  bind-mounted `/data`. Creates users, host keys, and the checksum.
- **Entry point (runtime):** `entrypoint.sh`, which validates that checksum and
  then execs `sshd -D -e`.
- **Central concept:** the **provisioned `data/` directory** and the hash over
  it. Everything else is arrangement around those two.

---

## Environment

No language runtime, no virtualenv. Docker and the repo root:

```bash
cd /path/to/bastion
cp .env-dist .env          # required first; see below
docker compose config      # validates .env + compose wiring, starts nothing
```

- **Run compose from the repo root.** Every volume in `docker-compose.yml` is
  `$PWD/data/...`, read from the shell's working directory. From a
  subdirectory it silently mounts the wrong paths — no error, just a bastion
  with somebody else's `/etc/passwd`.
- **`docker compose config` "works" without `.env`.** It emits twelve *warning*
  lines and substitutes empty strings, then prints a valid-looking document
  with `BASE_VERSION: ""` and `published: ""` *(verified 2026-08-28)*. It is
  not a check unless `.env` exists — copy `.env-dist` first and read the
  warnings, not just the exit status.
- **The compose project name is `bastion`, derived from the directory**, since
  `docker-compose.yml` declares no `name:` *(verified 2026-08-28)*. So
  `docker compose up/down` here acts on a project named after wherever this is
  checked out. That is *not* the deployed bastion — the deployment
  `ib-docker` describes runs as `inv_bastion` under project `inv_ibkr`, from
  the vendored copy. Still check `docker compose ps` before any lifecycle
  command: the guarantee is about this file, not about what someone renamed a
  directory to.
- **Not installed, and not in this repository:** `pre-commit`, `shellcheck`,
  `shfmt`, `hadolint`, `bats`, `gh`. There is **no `.github/`** and **no
  `tests/`** — both were deleted in `671c271` (2026-06-14). Never plan a step
  around `gh` or around a CI run.
- `origin` is **HTTPS**, not SSH. A non-interactive push can prompt for a
  username and fail. `ib-docker` hit exactly this and moved to SSH on
  2026-08-25; this repository has not.

### What a fresh clone does not have

Everything below is gitignored or untracked and must be supplied out-of-band:

| path | what it is | how to get it |
|---|---|---|
| `.env` | every variable `docker-compose.yml` interpolates | `cp .env-dist .env`, then edit; `.env-dist` is the key list |
| `data/etc/{passwd,shadow,group}` | the bastion's users | written by `provision.sh` |
| `data/etc/ssh/` | `sshd_config`, host keys, and the provisioned hash | written by `provision.sh` |
| `data/home/<user>/.ssh/authorized_keys` | user access | **you** copy it in *before* provisioning, so permissions get set |
| `data/etc/ssh/user_ca.pub`, `…-cert.pub` | CA material, only when `CA_ENABLED=yes` | copied in by hand |

> **Create `data/` and the `authorized_keys` files before the first
> `docker compose up`.** The compose file bind-mounts four *files*
> (`data/etc/passwd`, `shadow`, `group`) and two directories. Docker creates a
> **directory** at the path of a missing file mount, and a directory at
> `/etc/passwd` breaks the container in a way whose error message points
> nowhere near the cause. `provision.sh` is what creates them; run it first.

> **The secrets guard here is thinner than it looks.** `.gitignore` covers
> `.env` and `/data/`, and that is the whole of it — this repository has
> **no** `detect-private-key` hook and **no** `no-real-env-files` hook, both of
> which `ib-docker` has. `data/` holds SSH **host private keys** and
> `/etc/shadow`. A `git add -f`, or a variant filename like `.env.bak` or
> `prod.env`, is not stopped by anything. **Stage files by name. Never
> `git add -A` or `git add .`.**

---

## Key conventions

- **`data/` is provisioned, not edited.** `provision.sh` `set_checksum()` writes
  `sha256sum` lines for `/etc/passwd`, `/etc/group`, `/etc/shadow`,
  `/etc/ssh/sshd_config` and every `ssh_host_*_key*` into
  `bastion_provisioned_hash`; `entrypoint.sh` `check_provision()` runs
  `sha256sum -c` over it at every start and **exits 1** on a mismatch. Editing
  a covered file by hand makes the container refuse to start. That is the
  feature. Re-run `provision.sh` instead.
- **`bastion_provisioned_hash.sum` is written and never read.** `set_checksum()`
  hashes the hash file into `.sum` beside it; nothing in `entrypoint.sh`
  verifies it *(grep-verified 2026-08-28 — `PROVISON` is the only path
  `check_provision()` touches)*. So the list of digests is itself unprotected
  at runtime. Do not describe `.sum` as a check; it is a record.
- **Per-file hashes cannot see a drop-in being added or removed.**
  `sshd_config` line 4 is `Include /etc/ssh/sshd_config.d/*.conf`, so a file
  dropped in there can set `AllowTcpForwarding`, `PermitRootLogin` or the
  cipher list — and every recorded digest still checks out, because the new
  file has no recorded line. **This is open in this repository.**
  `ib-docker/bastion/sshd_config_d.sh` fixes it by hashing the sorted
  *listing* as a file of its own. If you are asked to close it here, **port
  that file**; do not invent a second design.
- **`ARG IMAGE_VERSION` is never declared, so the version LABEL is broken.**
  `Dockerfile` ends with
  `LABEL org.opencontainers.image.version=${IMAGE_VERSION}-${BASE_VERSION}`,
  but only `BASE_VERSION` and `APT_PROXY` are declared as build args, and
  `docker-compose.yml` passes only those two — `IMAGE_VERSION` lives in
  `.env-dist` and reaches the build as nothing. Every image built from this
  tree reports version `-resolute` *(grep-verified 2026-08-28)*. Fixed in
  `ib-docker/bastion/Dockerfile` with `ARG IMAGE_VERSION=2604.01` plus the
  matching compose build arg.
- **`SSHD_OPT` elements are one argv entry each, containing a space, and that
  is deliberate.** `set_totp()` and `set_CA()` push strings like
  `'-o KbdInteractiveAuthentication=yes'`; `sshd` parses `-o<value>` with the
  value glued to the flag, and the leading space is harmless inside a config
  line. Splitting them into `-o` + value "to fix the quoting" changes nothing
  and risks the word-splitting bug the array exists to avoid.
- **`AllowGroups ssh-bastion` is the real access control.** The `Dockerfile`
  creates the group at **GID 59999**; `provision.sh` `create_users()` adds every
  `USERS` entry to it and sets `usermod -p '*'` to disable password login. A
  user created any other way authenticates against nothing — `sshd` will
  refuse it regardless of `authorized_keys`.
- **`AllowTcpForwarding yes` is required, not an oversight.** It is what makes
  `-J`/`ProxyJump` work. `PermitTTY no`, `X11Forwarding no`, `PermitTunnel no`,
  `AllowStreamLocalForwarding no` and `MaxSessions 0` are the hardening around
  it. Read `sshd_config` as a set before changing any one line.
- **TOTP contradicts the read-only `/home` mount.** `docker-compose.yml` mounts
  `$PWD/data/home:/home:ro`, but `pam_google_authenticator` needs to *write*
  the user's home directory. With `TOTP_ENABLED=yes` the `:ro` must be dropped —
  the README says so under *Setting MFA/TOTP*; the shipped compose file does
  not. `check_totp_users()` in `entrypoint.sh` refuses to start if any
  `ssh-bastion` member lacks a `.google_authenticator`, so this fails loudly at
  least.
- **`provision.sh` replaces `/home` with a symlink.** `create_data_dir()` does
  `mv /home /home-dist` then links `$DATA/home` in its place, and
  `set_sshd_config()` does the same for `/etc/ssh` → `/etc/ssh-dist`. The
  symlinks exist so `sha256sum` records `/etc/...` paths — the paths the
  *running* container will check. Anything that changes where those files live
  breaks the checksum silently.
- **`USER_SHELL` defaults to `/usr/sbin/nologin`** and should stay there.
  `provision.sh` applies the default when the variable is empty, which is what
  `.env-dist` ships.
- **`.env-dist` is the key list and stays tracked.** Add a variable there in the
  same commit that reads it, and keep it grouped by concern — it is read as
  documentation, and the README's variable table is generated by nobody.
- **`sntrup761.conf-dist` is orphaned.** Nothing copies, includes or mentions it
  *(grep-verified 2026-08-28)*; `sshd_config` already lists
  `sntrup761x25519-sha512@openssh.com` first in `KexAlgorithms`. It is a
  drop-in for a `sshd_config.d/` that the image does not populate. Do not wire
  it up as a "fix" without being asked — but do not assume it is live either.
- **`docker-compose.yml` is tracked *and* listed in `.gitignore`.** The ignore
  line is inert for a tracked file, so edits are staged normally — but delete
  and recreate the file and it vanishes from git's view. It became tracked in
  `2bd7487`, renamed from `template_docker-compose.yml`; the ignore line is a
  leftover. `ib-docker/bastion/.gitignore` has already dropped it.

---

## Git workflow

- Development happens on the branch the task names —
  currently `claude/claude-md-docs-lurk9v`, which at the time of writing was
  identical to `master`. Create it from `master` if it does not exist.
- **Stage by name, never `-A`/`.`** — see the secrets warning above. This
  repository's only guard is `.gitignore`.
- Push with `git push -u origin <branch>`. `origin` is HTTPS; if a push stalls
  on credentials that is the cause, not the network.
- Feature work in **worktrees** is the house preference
  (`git worktree add ../wt-<name> -b <name>`). In a worktree, edit **only** the
  worktree path. If a worktree command fails, run `git worktree list` and
  `git worktree prune` before retrying — do not retry blindly.
- **Nothing checks a push here.** There is no CI, no test job, no lint job. The
  verification below is the only verification that will ever happen, so it has
  to happen before the commit, not after.

---

## Testing

"Verified" means the checks below, run **in the foreground with a bounded
timeout**, and **reported with what actually ran**. "Lint passes" without
naming what was checked is not a report. There is no test suite here to hide
behind.

### 1. Lint — the canonical check

`.pre-commit-config.yaml` defines nine hooks. **This repository's hooks need
host binaries**, unlike `ib-docker`, which deliberately uses the `*-docker`
variants so Docker is the only prerequisite: here
`jumanjihouse/pre-commit-hooks` supplies `shellcheck` and `shfmt` as `script`
hooks, and `hadolint/hadolint` is pinned to the `hadolint` id, not
`hadolint-docker`. So `pre-commit run --all-files` needs `shellcheck`, `shfmt`
and `hadolint` on `PATH`.

```bash
python3 -m venv .venv && .venv/bin/pip install pre-commit   # once
.venv/bin/pre-commit run --all-files
```

If those binaries are unavailable, the equivalent Docker invocations are:

```bash
docker run --rm -v "$PWD:/mnt" -w /mnt koalaman/shellcheck:stable \
  entrypoint.sh provision.sh
docker run --rm -v "$PWD:/work" -w /work mvdan/shfmt:latest -d entrypoint.sh provision.sh
docker run --rm -i hadolint/hadolint:v2.12.0 hadolint - < Dockerfile
```

Known state, measured 2026-08-28 **without** running the hooks (no Docker
daemon and no linters in that environment — this is a file inspection, not a
hook run):

- `end-of-file-fixer` **will rewrite two files**: `Dockerfile` and
  `bastion_banner.txt` both lack a final newline. `ib-docker/bastion/` has
  already fixed both. Expect the first run to be red and to edit the tree.
- No trailing-whitespace matches in tracked files.
- Both scripts are `755` with `#!/usr/bin/env bash`, so
  `check-shebang-scripts-are-executable` and `check-executables-have-shebangs`
  are satisfied.

**Some hooks rewrite files.** Check `git status` after a run; a "Passed" second
run may only mean the first one already edited the tree.

**`--all-files` means all *tracked* files.** A file created but not yet
`git add`ed is invisible to every hook. Stage new files *before* the
verification run.

### 2. Compose configuration — fast, offline

```bash
cp .env-dist .env
docker compose config
```

Read the warnings, not just the exit status — see *Environment*. A missing
variable produces a warning and an empty string, never a failure.

### 3. Build — offline, slow

```bash
timeout 1800 docker build --progress=plain -t bastion:check .
```

`docker-compose.yml` declares `platforms: [linux/amd64, linux/arm64]`, so **a
local build proves one architecture only**. For the other, register the
emulator once (`docker run --privileged --rm tonistiigi/binfmt --install
arm64`) and:

```bash
timeout 2400 docker build --platform linux/arm64 -t bastion:arm64 .
```

Nothing else builds this image on a schedule. Report the outcome; a build that
was not run is not a build that passed.

### 4. Runtime — a throwaway, never a real `data/`

There is no `bats` suite here (`tests/` was deleted in `671c271`), so a runtime
check is by hand and must be isolated:

```bash
# provision a throwaway data dir
mkdir -p /tmp/bastion-check/data
docker run -it --rm --env-file .env -v /tmp/bastion-check/data:/data \
  bastion:check /provision.sh
```

- **Never point a check at a `data/` that is in use.** `provision.sh` rewrites
  `passwd`, `shadow`, `group` and the checksum in place, and re-runs
  `chown -R`/`chmod` over every home directory.
- Use `-p <throwaway-project>` with distinct container names and ports for any
  compose-based check, so nothing collides with a running bastion.
- The one check worth having: provision a throwaway `data/`, tamper with a
  covered file, and assert the container refuses to start. That is
  `bastion_hash.bats` in `ib-docker/tests/container/` — run it from there
  with `BASTION_IMAGE` pointing at the image built here, rather than writing a
  second one.

---

## Debugging

- **State the root cause with evidence — a log line, a reproducing command, a
  failing check — before editing anything.** A patch without a stated cause is
  a guess with a diff attached.
- **There is no harness to add a regression test to, so record the
  reproduction instead.** Demonstrate the failure against the unfixed code,
  paste the output, then show the same command clean. If the fix belongs to
  behaviour `ib-docker/tests/` already covers, add the case *there* — that is
  where a check can actually run.
- `docker logs <container>` first; it is read-only and restarts nothing.
  `entrypoint.sh` narrates every decision it makes with `> ` lines, including
  which of TOTP, CA and banner it enabled, so the startup log usually answers
  the question on its own.
- **A refusal to start is almost always the checksum.** `check_provision()` and
  `check_totp_users()` are the only two `exit 1` paths before `sshd` runs, and
  both print what failed. `sha256sum -c` names the file.
- **A scoped grep answers a scoped question.** Variable names, ports and image
  tags appear in `Dockerfile`, `docker-compose.yml`, `.env-dist`,
  `entrypoint.sh`, `provision.sh` and `README.md`. Run `git grep -n "<value>"`
  over the whole tree before calling anything fixed — and then check
  `ib-docker/bastion/` for the same string.
- Do not treat a prior session's notes as an exclusion list. They are
  point-in-time records. Judge every path on today's source.

---

## Deployment and CI

**There is none in this repository, and that is a deliberate state, not a gap
to fill on your own initiative.** `671c271` (2026-06-14) deleted
`.github/workflows/docker-base-image.yml`, `docker-build-n-test.yml`,
`docker-publish.yml`, `dpkg-updater.yml`, `dependabot.yml` and the whole
`test/` directory — 704 lines. Nothing has replaced them.

So:

- **No push to this repository builds, tests, lints or publishes anything.**
- `ghcr.io/dennisdeh/bastion` is published from `ib-docker`, by its
  `publish.yml`, from `ib-docker/bastion/Dockerfile` — tagged with that file's
  `ARG IMAGE_VERSION`, which carries no IB Gateway version because the bastion
  has none. **That is the one line to bump for a release, and it is not in
  this repository.**
- The version here (`IMAGE_VERSION=2604.01` in `.env-dist`) does not reach any
  image at all — see the LABEL entry under *Key conventions*.
- `docker-compose.yml` names `dennisdeh/bastion:local-resolute`, a **local
  build tag**. It is not a registry coordinate and pulling it will fail. That
  is correct for a repository whose compose file exists to build.
- Before proposing CI here, check whether the work belongs in `ib-docker`
  instead. Two publishing pipelines for one image is the failure mode.

---

## Documentation

- **`README.md` is hand-maintained here.** In `ib-docker` it is generated and
  editing it is a mistake; here it is the opposite, and the habit transfers
  wrongly in both directions. Edit `README.md` directly, and remember nothing
  regenerates or checks it.
- **It has drifted.** Verified 2026-08-28, all still present:

  | the README says | reality |
  |---|---|
  | `BASE_VERSION` default `jammy` (variable table) | `.env-dist` ships `resolute` |
  | links to `docker-compose.yml-dist` | deleted in `d9ef9b9` |
  | four links under `dennisdeh/docker-bastion` | repository is `dennisdeh/bastion` |
  | no row for `DNS` | `.env-dist` defines it; compose references it, commented |
  | `version: "3.6"` in the compose example | obsolete key, absent from the real file |

  Fix these when a task touches the surrounding text. Do not open a sweep of
  them as a side effect of unrelated work without saying so.
- **State a fact once.** If it belongs in two places, put it where the question
  is answered and link from the other — a fact stated twice goes stale once.
- **Anchor to symbol names, never line numbers** — `check_provision()` in
  `entrypoint.sh`, not `entrypoint.sh:71`. Line numbers move with the next
  edit; the name does not. This applies to code comments too.
- **Date every measurement and every version claim.** "as of 2026-08-28:
  `.env-dist` `BASE_VERSION=resolute`" stays checkable; a bare number does not.
- **Never write branch, worktree or merge state as present tense.** Write "at
  the time of writing, nothing was merged", or give the date.
- **Changing behaviour in `entrypoint.sh`, `provision.sh`, `sshd_config` or
  `Dockerfile` means updating `README.md` and `.env-dist` in the same commit**
  if either names the thing you changed.
- **Prose lives in documentation, not in code comments.** A comment says *what*
  and points; the docs say *why*, with the evidence. Keep the one-sentence trap
  at the line being edited. **Never delete such a comment — relocating is the
  only way to shorten it.**
- Do not number steps in comments and do not restate the next line.

---

## Rules vs. checks — how to grow this file

In order of value:

1. **A mechanical check** — a test, a lint rule, a CI gate. Every check should
   exist because a defect got through. **This repository has no place to put
   one**, which is why the list below is unusually long: with no CI and no
   suite, every rule above is carrying weight a check should carry. The
   realistic home for a new check is `ib-docker/tests/`.
2. **A rule in this file**, when a check is impossible or not yet worth
   writing.
3. **Nothing.** A rule nobody follows is worse than no rule: it trains the
   reader that this file is decoration.

Standing candidates for promotion, in rough order of what has already gone
wrong:

- `IMAGE_VERSION` reaching the image LABEL — currently silently empty.
- The `sshd_config.d/` listing being covered by the checksum — the gap
  `ib-docker/bastion/sshd_config_d.sh` closes.
- A `detect-private-key` hook and a `no-real-env-files` hook, both of which
  `ib-docker` has and this repository — which provisions SSH host private keys
  — does not.
- A diff of this tree against `ib-docker/bastion/`, so divergence is reported
  rather than discovered.

**Prune as well as add.** Delete a rule when its check exists, when the thing
it guards is gone, or when it has never once been the thing that went wrong.
