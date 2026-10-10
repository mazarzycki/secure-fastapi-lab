# Change plan: secure-fastapi-lab docs, plan revision 3.2

## Instructions for the assistant applying this plan

You are editing the repository **mazarzycki/secure-fastapi-lab** (branch `main`). Apply the changes below to documentation files only.

Rules:

1. Apply the steps **in order**. Step 1 is a scripted rename; every later "Find" block is written against the text **after** that rename.
2. Each **Find** block is an exact excerpt of the current file and occurs **exactly once**. Replace it with the **Replace with** block exactly, keeping surrounding text untouched.
3. If a Find block doesn't match exactly once, **stop and report** which step failed. Don't guess.
4. Don't reformat, re-wrap or re-pad anything you weren't asked to change. Don't touch code, scripts, workflows or Compose files.
5. Step 7 depends on a decision only the repo owner can make. **Ask before applying it.**
6. Finish with the checks in step 11 and report their output.

Inputs you need besides the repository:

- The setup guide file supplied by the user (currently named `README.md`, titled `# HP t740 Home Server: Ubuntu Server 26.04 LTS from Scratch`). It becomes `docs/lab-host-setup.md` in step 10.

### Why these changes

- The v1.0 target moves from **30 November 2026** to **30 December 2026**.
- The plan stops scheduling individual blocks by date. Each version keeps **one target date only**: v1.0 is 30 December 2026; v1.1 gets its date when v1.0 is tagged.
- "Week N" blocks become "Block N", because they are an order of work, not calendar weeks.
- The lab host (an HP t740 running Ubuntu Server 26.04) is now built, and its setup guide moves into the repo.
- `README.md` already uses 30 December 2026 and needs no date changes.

---

## Files touched

| File | Change |
| --- | --- |
| `Project_description/homelab_networking_plan.md` | Steps 1–8 |
| `Project_description/Simple_version/homelab.md` | Step 9: mark as superseded |
| `README.md` | Step 10 |
| `docs/lab-host-setup.md` | Step 10: new file (the supplied guide) |
| `docs/.gitkeep` | Step 10: delete |

---

## Step 1: rename "Week N" to "Block N" in the plan

Run from the repository root:

```bash
sed -i -E 's/\bWeek ([1-6])/Block \1/g' Project_description/homelab_networking_plan.md
```

This changes 28 references (including headings such as `## Week 1`, phrases such as "from Week 4 on", and "Week 1–2"). Historical revision notes are included on purpose so the whole file uses one vocabulary.

"Block" is one character longer than "Week", so a few space-padded tables lose their column alignment in the raw file. That's cosmetic: GitHub renders them correctly. Don't re-pad them.

Check:

```bash
grep -nE '\bWeek [0-9]' Project_description/homelab_networking_plan.md   # expected: no output
```

---

## Step 2: header and reading guide

File: `Project_description/homelab_networking_plan.md`

**Find**

````text
**Revision:** 3.1 — 7 October 2026
**v1.0 target:** tagged by **30 November 2026**
````

**Replace with**

````text
**Revision:** 3.2 — 10 October 2026
**v1.0 target:** tagged by **30 December 2026**
````

**Find**

````text
- Every milestone maps to exactly one schedule block (section 23).
- Every week has a **core** and a **stretch** part. Stretch is the first thing cut when time runs short.
````

**Replace with**

````text
- Milestones are done in the order given in section 23. Blocks have no dates; only the version target does.
- Every block has a **core** and a **stretch** part. Stretch is the first thing cut when time runs short.
````

**Find**

````text
so no later week has to migrate the network layout.
````

**Replace with**

````text
so no later block has to migrate the network layout.
````

---

## Step 3: section 23, schedule becomes order of work

File: `Project_description/homelab_networking_plan.md`

**Find**

````text
# 23. Schedule to 30 November 2026

Calendar: Wednesday 7 October to Monday 30 November. Six build blocks after the setup block, plus a protected buffer.
````

**Replace with**

````text
# 23. Order of Work

v1.0 has one date: **30 December 2026**. Blocks are done in order and have no dates of their own: Setup, Blocks 1–6, then a protected buffer before the tag.
````

Then replace the **whole table** that follows (it starts with the line `| Block  | Dates        | Milestone ...` and ends with the `| Buffer | ...` row, 10 lines in total) with this table. It is the same table with the **Dates** column removed and the padding dropped; no cell text changes:

````text
| Block | Milestone | Core | Stretch (cut first) | Exit check |
| --- | --- | --- | --- | --- |
| Setup | M0 + M1 part 1 | lab-host decision; host baseline **before** the first `compose up`; repo and `.gitignore`; secrets script; `/health`, `/db-check`; settings loader; Compose with Nginx, API, PostgreSQL and all four networks; one pytest; CI with pytest and Gitleaks | Alembic baseline migration | `/health` and `/db-check` return 200 through `127.0.0.1:8080` from a fresh `up` |
| Block 1 | M1 part 2 | Redis; `/cache-check`; `/limited` (pipeline, 503 fail-closed); `/debug/request`; `/slow`; JWT login, `/protected` and an authorization test; dependency timeouts; healthchecks; persistence check; Alembic if not done | Hadolint, Semgrep, Trivy report-only | Block 1 exit commands (section 14) all 200 |
| Block 2 | M2 | sections 15.1–15.6; `matrix.sh` with every row explained; DNS capture; port publishing; clean-clone smoke test on the final host (if different from the lab host); report-only scanners if not done | LAN-client comparison (15.6) | `docker-networking.md` with matrix output committed |
| Block 3 | M3 | veth, bridge-in-`ns-sw` and router labs saved as scripts; ARP, ICMP, TCP, HTTP and RESP captures | Wireshark screenshots (text excerpts are enough) | `linux-network-primitives.md` and `packet-captures.md` part 1 committed |
| Block 4 | M4 | 17.1–17.3 (A, B, C); certs script; TLS block; TLS 1.3 capture; timeout table | subnet-membership bonus; TLS 1.2 comparison capture | `proxy-client-ip.md` and the Experiment C write-up committed |
| Block 5 | M5 | eight drills | optional router drill | `troubleshooting.md` with eight entries committed |
| Block 6 | CI + docs | scanner triage, main switches to blocking; `vuln/sql-injection` write-up; one-page threat model; four diagrams; README | diagram polish | CI green and blocking on main; all DoD docs exist |
| Buffer | ship | final clean clone on the final host; DoD review; fix broken instructions; remove dead experiments; tag | — | `v1.0` tag pushed |
````

**Find**

````text
## Weekly gate

Every Sunday:

- commit everything
- the block's deliverable doc exists, even if rough
- tick the finished DoD boxes

If a block's exit check is still open on Tuesday of the following week, apply the cut order immediately rather than borrowing from the buffer.
````

**Replace with**

````text
## Gates

- Don't start a block until the previous block's exit check passes, or its open items have been cut.
- Every Sunday: commit everything, make sure the current block's deliverable doc exists (even if rough), and tick the finished DoD boxes.
- Halfway to the tag date (mid-November), Block 3 should be done. If it isn't, apply the cut order immediately rather than borrowing from the buffer.
````

**Find**

````text
Cut from the top until back on schedule:
````

**Replace with**

````text
Cut from the top until back on track:
````

**Find**

````text
If a DoD item is still open on 27 November, move it to a "v1.0.1" list in the README and tag v1.0 on time anyway.
````

**Replace with**

````text
If a DoD item is still open three days before the tag date, move it to a "v1.0.1" list in the README and tag v1.0 on time anyway.
````

---

## Step 4: remaining November dates

File: `Project_description/homelab_networking_plan.md`

**Find**

````text
- [ ] `v1.0` tagged by 30 November 2026
````

**Replace with**

````text
- [ ] `v1.0` tagged by 30 December 2026
````

**Find**

````text
No Azure resources before 30 November.
````

**Replace with**

````text
No Azure resources before v1.0 ships.
````

**Find**

````text
The objective is not to become a network engineer by 30 November.
````

**Replace with**

````text
The objective is not to become a network engineer by v1.0.
````

**Find**

````text
# 27. v1.1 — Network Track (after v1.0 ships)

Order follows dependency, not novelty.
````

**Replace with**

````text
# 27. v1.1 — Network Track (after v1.0 ships)

**Target date:** set when v1.0 is tagged.

Order follows dependency, not novelty.
````

Leave revision note 29 in the "rev 2 -> rev 3" appendix ("so the 30 November tag doesn't depend...") unchanged. It records what revision 3 said.

---

## Step 5: safety rule for a headless lab host

File: `Project_description/homelab_networking_plan.md`, section 3.

**Find**

````text
- publish Nginx only on host loopback (`127.0.0.1`)
````

**Replace with**

````text
- publish Nginx only on host loopback (`127.0.0.1`)
- reach the stack from another machine only through an SSH tunnel to the host's loopback, never by publishing on a LAN address
````

---

## Step 6: lab host facts (always apply)

File: `Project_description/homelab_networking_plan.md`, section 4.

**Find**

````text
The t740 is likely headless. Capture there with `tcpdump -w`, copy the file to the laptop, and open it in Wireshark.
````

**Replace with**

````text
The t740 is headless and managed over SSH and Tailscale. Capture there with `tcpdump -w`, copy the file to the laptop, and open it in Wireshark.

How the t740 was built (Ubuntu Server 26.04 LTS, SSH keys, ufw, Tailscale, Docker Engine): [docs/lab-host-setup.md](../docs/lab-host-setup.md).
````

---

## Step 7: lab host decision (ASK THE OWNER FIRST)

Ask: **"Is the t740 the lab host (Option 1) or the laptop (Option 2)?"** Apply only the matching block.

The decision line goes **after** the Option 2 line, so the two options stay one list.

**If Option 1**, make two replacements (text after the step 1 rename).

**Find**

````text
- **Option 1:** the t740 is the lab host from Block 2 onward. All captures and host-specific findings come from it.
````

**Replace with**

````text
- **Option 1:** the t740 is the lab host from the setup block onward. All captures and host-specific findings come from it.
````

**Find**

````text
- **Option 2:** the laptop is the lab host. Then run a clean-clone smoke test on the t740 in Block 2, and repeat the host-specific checks of section 15.6 there before writing the final docs.
````

**Replace with**

````text
- **Option 2:** the laptop is the lab host. Then run a clean-clone smoke test on the t740 in Block 2, and repeat the host-specific checks of section 15.6 there before writing the final docs.

**Decision (10 October 2026): Option 1.** The host baseline (Milestone 0) is taken on the t740.
````

**If Option 2**, make one replacement.

**Find**

````text
- **Option 2:** the laptop is the lab host. Then run a clean-clone smoke test on the t740 in Block 2, and repeat the host-specific checks of section 15.6 there before writing the final docs.
````

**Replace with**

````text
- **Option 2:** the laptop is the lab host. Then run a clean-clone smoke test on the t740 in Block 2, and repeat the host-specific checks of section 15.6 there before writing the final docs.

**Decision (10 October 2026): Option 2.** The laptop is the lab host; the t740 runs the clean-clone smoke test.
````

---

## Step 8: host baseline additions and revision notes

File: `Project_description/homelab_networking_plan.md`, section 13.

**Find**

````text
cat /etc/docker/daemon.json 2>/dev/null || echo "no daemon.json"
````

**Replace with**

````text
cat /etc/docker/daemon.json 2>/dev/null || echo "no daemon.json"
sudo ufw status verbose
tailscale status
````

**Find**

````text
Save as `docs/host-network-baseline.md`.
````

**Replace with**

````text
Save as `docs/host-network-baseline.md`.

On the t740, ufw and Tailscale were set up before this baseline (see `docs/lab-host-setup.md`). Expect a `tailscale0` interface and ufw/Tailscale chains in the firewall output, and record them as pre-existing. Take the baseline after removing any test Compose projects (`docker compose down -v`), so their networks don't show up in the subnet check.
````

Append the new revision notes at the very end of the file.

**Find**

````text
7. **"No API port mapping" stated as a design decision**, not as a consequence of Docker's handling of internal networks.
````

**Replace with**

````text
7. **"No API port mapping" stated as a design decision**, not as a consequence of Docker's handling of internal networks.



## Revision 3.2

1. **Dated schedule removed.** Each version has one target date: v1.0 is 30 December 2026 (was 30 November); v1.1 gets its date when v1.0 is tagged.
2. **"Week N" renamed "Block N"** throughout. Blocks are an order of work, not calendar weeks.
3. **Weekly Tuesday rule replaced** by block gates and one halfway check (section 23).
4. **DoD cut-off is relative:** three days before the tag date.
5. **Lab host provisioned** and documented in `docs/lab-host-setup.md`; baseline now records ufw and Tailscale (sections 4, 13).
6. **SSH tunnel rule added** for reaching the loopback-only stack from another machine (section 3).
````

---

## Step 9: mark the simple plan as superseded

File: `Project_description/Simple_version/homelab.md`. Don't change its dates; it is kept only for history.

**Find**

````text
# Secure FastAPI Homelab

**Target:** v1.0 complete by **30 November 2026**
````

**Replace with**

````text
# Secure FastAPI Homelab

> [!WARNING]
> **Superseded** by [homelab_networking_plan.md](../homelab_networking_plan.md) (revision 3.2).
> Kept for history. Its dates, four-week plan and version names (v1.1 to v2.0 here) are not current.

**Target:** v1.0 complete by **30 November 2026**
````

---

## Step 10: README and the setup guide

### 10.1 Add the guide

1. Save the user-supplied guide as `docs/lab-host-setup.md`.
2. In that file, find:

   ````text
   # HP t740 Home Server: Ubuntu Server 26.04 LTS from Scratch
   ````

   Replace with:

   ````text
   # Lab host setup: HP t740 + Ubuntu Server 26.04 LTS

   Part of [secure-fastapi-lab](../README.md). This is the host the lab's evidence comes from.
   ````

3. Delete `docs/.gitkeep` (the folder is no longer empty).

### 10.2 README edits

File: `README.md`

**Find**

````text
git clone <repo-url> secure-fastapi-lab && cd secure-fastapi-lab
````

**Replace with**

````text
git clone https://github.com/mazarzycki/secure-fastapi-lab.git && cd secure-fastapi-lab
````

**Find**

````text
| A Linux host or VM | Bare metal or a VM. Docker Desktop on macOS or Windows hides the bridges inside its own VM, so the capture chapters won't match |
````

**Replace with**

````text
| A Linux host or VM | Bare metal or a VM. Docker Desktop on macOS or Windows hides the bridges inside its own VM, so the capture chapters won't match. How I built mine: [docs/lab-host-setup.md](docs/lab-host-setup.md) |
````

**Find**

````text
curl -s http://127.0.0.1:8080/health
```
````

**Replace with**

````text
curl -s http://127.0.0.1:8080/health
```

On a headless lab host, run the stack there and reach it from your laptop through an SSH tunnel. The ports stay on the host's loopback:

```bash
ssh -L 8080:127.0.0.1:8080 -L 8443:127.0.0.1:8443 <user>@<lab-host>
```

With the tunnel open, the same `curl` works from the laptop.
````

**Find**

````text
├── docs/                 one write-up per chapter
````

**Replace with**

````text
├── docs/                 one write-up per chapter
│   └── lab-host-setup.md how the lab host (HP t740, Ubuntu Server 26.04) was built
````

---

## Step 11: verify and report

Run from the repository root and report the output:

```bash
P=Project_description/homelab_networking_plan.md

grep -nE '\bWeek [0-9]' "$P"                      # expected: no output
grep -nE '[0-9]+(–[0-9]+)? (Oct|Nov)\b' "$P"       # expected: no output (no block dates left)
grep -n 'November' "$P"                            # expected: 3 lines: Gates halfway check, rev-3 note 29, rev-3.2 note 1
grep -n '30 December 2026' "$P"                    # expected: header, section 23, DoD, rev 3.2 note
grep -rn 'lab-host-setup.md' README.md Project_description/   # expected: README (2), plan (2+)
test -f docs/lab-host-setup.md && echo "guide present"
test ! -e docs/.gitkeep && echo ".gitkeep removed"
```

Also confirm that the table in section 23 renders as a table on GitHub (header row plus separator row, 5 columns in every row).

Suggested commit message:

```text
docs: plan rev 3.2, single v1.0 date and lab host guide

- v1.0 target moved to 30 December 2026; per-block dates removed
- "Week N" blocks renamed "Block N"; gates and halfway check replace the weekly rule
- add docs/lab-host-setup.md (HP t740, Ubuntu Server 26.04) and link it
- baseline records ufw and Tailscale; SSH tunnel rule for the headless host
- mark Simple_version/homelab.md as superseded
```
