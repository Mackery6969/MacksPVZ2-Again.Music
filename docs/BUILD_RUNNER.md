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
4. if `BANK_INSTALL_DIR` is set, replaces the `.bank` files there with the new
   ones - point it at the game checkout's `audio/banks`;
5. uploads the banks as the `fmod-banks-Desktop` artifact, kept for 14 days,
   for any other machine.

Banks are never committed: the Music bank alone is over 100 MB, and committing
it on every build would use up GitHub's free LFS quota within a few builds.

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

### 2. Set the repository variables

**Settings > Secrets and variables > Actions > Variables > New repository
variable**:

| Name | Example | |
|---|---|---|
| `FMOD_STUDIO_CL` | `D:\Programs\FMOD Studio 2.03.14\fmodstudiocl.exe` | required |
| `BANK_INSTALL_DIR` | `D:\Programs\GitHub\MacksPVZ2-Again\audio\banks` | optional |

Use a **2.03** Studio. The project is saved in the 2.03 format, and the FMOD
Engine the game loads has to be the same minor version as the Studio that
built the banks.

### 3. Try it

**Actions > Build FMOD banks > Run workflow**. A green run shows the bank
sizes in the **Build banks** step.

## Troubleshooting

- **Job stays queued:** the runner is offline, or missing the `fmod` label
  (Settings > Actions > Runners shows both).
- **"FMOD_STUDIO_CL is not set":** the repository variable is missing or
  points at the wrong path.
- **Copy step fails with "being used by another process":** the game or the
  Godot editor has the banks loaded. Close it and re-run the job.
- **Clips missing from the build:** check that `git lfs version` works for the
  runner's account; without LFS the checkout holds pointer files, not audio.

## Security

The repository is public, so the workflow must never run on `pull_request`:
that would let anyone who opens a pull request run code on the runner's PC. It
only runs on pushes to `master` and manual dispatch.
