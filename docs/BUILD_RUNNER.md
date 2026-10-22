# Automated bank builds

`.github/workflows/build-banks.yml` builds the banks whenever `master` changes
the Studio project (`PVZGE.fspro`, `Metadata/` or `Assets/`), and on demand
from the Actions tab (**Build FMOD banks > Run workflow**).

GitHub's hosted runners have no FMOD Studio, and downloading it needs an
fmod.com login, so the job runs on a **self-hosted runner** on a Windows PC
that already has Studio installed. Each build:

1. checks out the project with its LFS clips (only changed clips are fetched,
   since the runner keeps its workspace between runs);
2. runs `fmodstudiocl -build -platforms Desktop -ignore-warnings PVZGE.fspro`
   (about 20 seconds);
3. fails if `Master.bank` or `Master.strings.bank` is missing;
4. uploads the banks as the `fmod-banks-Desktop` artifact, kept for 14 days;
5. on `master` only, starts the game repo's **Update FMOD banks** workflow,
   which downloads that artifact, commits the banks to the game repo's
   `audio/banks/` (through Git LFS) together with a `Music` submodule bump to
   the commit they were built from, and pushes straight to its `master`.

The build is deterministic - the same project gives byte-identical banks - so
a change that does not affect a bank does not commit a new copy of it, and LFS
storage only grows when a bank really changes. The Music bank is over 100 MB,
so every change to the music does cost that much of the LFS quota.

## One-time setup

### 1. Register the runner

On the PC with FMOD Studio:

1. Open this repository on GitHub, then **Settings > Actions > Runners > New
   self-hosted runner**, and pick **Windows / x64**.
2. Run the commands GitHub shows, in a PowerShell window. Unpack the runner
   into a short path such as `D:\actions-runner`: the job's checkout lives
   under it, and Windows still trips over very long paths.
3. When `config.cmd` asks for **additional labels**, enter `fmod`. The
   workflow only runs on a runner labelled `self-hosted`, `windows` and `fmod`.
4. When it asks whether to **run as a service**, either:
   - answer **N** and start it with `run.cmd` whenever you want builds; or
   - answer **Y** and give your own Windows account when it asks which user
     to run as. The default, `NETWORK SERVICE`, cannot write to your game
     checkout and may not find Git LFS.

Git and Git LFS must be on that account's `PATH` (`git lfs version` should
work in a fresh PowerShell window).

### 2. Make a token for the two workflows

The Music workflow has to start a workflow in the game repo, and the game
repo's workflow has to download this repo's artifact. A workflow's own token
cannot reach another repository, so they share a personal token:

1. GitHub > **Settings > Developer settings > Personal access tokens >
   Fine-grained tokens > Generate new token**.
2. **Repository access: Only select repositories**, and pick both
   `MacksPVZ2-Again` and `MacksPVZ2-Again.Music`.
3. **Permissions > Repository permissions > Actions: Read and write.** Nothing
   else is needed. Pick an expiry you will remember to renew.
4. Add it as a repository **secret** named `BANKS_TOKEN` in **both**
   repositories (**Settings > Secrets and variables > Actions > Secrets > New
   repository secret**).

### 3. Set the repository variable

In this repo, **Settings > Secrets and variables > Actions > Variables > New
repository variable**: `FMOD_STUDIO_CL` =
`D:\Programs\FMOD Studio 2.03.14\fmodstudiocl.exe` (wherever yours is).

Use a **2.03** Studio. The project is saved in the 2.03 format, and the FMOD
Engine the game loads has to be the same minor version as the Studio that
built the banks.

### 4. Try it

**Actions > Build FMOD banks > Run workflow** on `master`. A green run shows
the bank sizes in the **Build banks** step, and a moment later the game repo's
**Update FMOD banks** run commits them (or reports they were already up to
date). Pull the game repo to get them.

## Troubleshooting

- **Job stays queued:** the runner is offline, or missing the `fmod` label
  (Settings > Actions > Runners shows both).
- **"FMOD_STUDIO_CL is not set":** the repository variable is missing or
  points at the wrong path.
- **"BANKS_TOKEN secret is not set", or a 401/403/404 from either workflow:**
  the token is missing from one of the repos, lacks Actions read and write on
  both, or has expired.
- **Pull fails on `audio/banks`:** an older setup copied banks in by hand.
  Delete the `.bank` files there and pull again.
- **Clips missing from the build:** check that `git lfs version` works for the
  runner's account; without LFS the checkout holds pointer files, not audio.

## Security

The repository is public, so the workflow must never run on `pull_request`:
that would let anyone who opens a pull request run code on the runner's PC. It
only runs on pushes to `master` and manual dispatch.
