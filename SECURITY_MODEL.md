# Security model: IBMi-topics-thanks-to-pub400

This threat model was written for Devin security scans. Scans use it to set the priority of their findings. Its current copy lives on the Devin scan profile "Banking migration + IBMi PUB400 threat model", and this file mirrors it so the next scan starts from it. Edit either one and keep the two in sync.

Status: a draft built from the code. The owner has not confirmed the assumptions below yet (see Open questions).

## 1. System overview and trust boundaries

This repo is a hobbyist's collection of IBM i 7.5 RPGLE/CLLE/SQL sources and how-tos. They are compiled and run under a normal user profile on PUB400, a free, shared, public IBM i instance. There is no service:
- Code runs as the user who installs it.
- The exit points would run inside system database server jobs, but only if an administrator registered them.

Trust boundaries:
- The IBM i user-profile boundary between PUB400 users.
- System jobs vs. user code.
- The workstation side of `envsav.ps1`.

There is no infrastructure repository, because PUB400 is someone else's machine.

## 2. Components

| Component | Code | Notes |
|---|---|---|
| **IBM i tools (PUB400)** | `Projects/`, `HowTo/` | `System/ExitPoints`: the DBSSNLMT exit programs for QIBM_QZDA_INIT / DDMACC. They build dynamic SQL from system environment variables and were never deployed, per their README. `System/SaveEnvironment`: the ENVSAV CL command plus the `envsav.ps1` workstation download. Also `Tools/JoblogList`, `Tools/UserSpaces`, `Tools/SampleDatabase`, `Utilities/CallStack` and the shared `Common/Includes`. |

## 3. Entry points and who can reach them

- Commands and programs: any PUB400 user who has been granted authority to them.
- Exit programs: if registered, system jobs would call them for every database connection.
- `envsav.ps1`: runs on the user's own workstation and downloads backups over SSH/SFTP.

## 4. Assets

- The installing user's libraries and IFS home, and their ENVSAV backup zips.
- The availability of the shared system's database servers (the exit point).

## 5. Attacker profiles (not confirmed)

- Another PUB400 user (local, low privilege) abusing objects or IFS files these tools create.
- A supply-chain attacker who changes the sources before someone installs them.

## 6. Existing controls

The Standard Error Handling Routine (SEHR) and the validity-checker conventions in `Projects/Common`.

## 7. Out of scope / accepted risk

The owner has not confirmed anything yet. Proposed: lower findings in exit-point code that was never deployed.

## 8. Open questions

1. How much do findings here matter: should other PUB400 users count as real attackers, or is this hobbyist sample code?
