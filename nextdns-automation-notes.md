# NextDNS Denylist Automation — Setup Notes

## Current status

Setup is functionally complete and live. `update-nextdns.yml` is
hardcoded to `profiles="111a83"` (confirmed against the actual file
content pasted from the fork). The most recent run pushed ~14,228
domains — largely `googlevideo.com` CDN hosts from `youtubelist.txt` —
into `111a83`'s denylist.

**Next action (pending on your end): test YouTube playback** on the
device(s)/profile using `111a83`. See "Testing YouTube after this
change" below for what to check and how to revert if needed.

## Overview
Automated syncing of a custom domain denylist to NextDNS via GitHub Actions,
replacing manual entry through ReNXEnhanced / the NextDNS dashboard.

- **Tool used:** [ph00lt0/custom-blocklists-nextdns-updater](https://github.com/ph00lt0/custom-blocklists-nextdns-updater)
- **Your fork:** https://github.com/vishalrajvanshi-collab/custom-blocklists-nextdns-updater
- **Underlying CLI:** `nextdnsctl` (community NextDNS API wrapper)

---

## Key URLs

| Purpose | URL |
|---|---|
| Your custom blocklist (hosted) | `https://gitlab.com/bangobang/ytblk/-/raw/main/NextDNS-Denylist.txt?ref_type=heads` |
| ph00lt0 default blocklist (NOT currently used — removed) | `https://raw.githubusercontent.com/ph00lt0/blocklist/master/domains.txt` |
| kboghdady "curated" YouTube list — **AVOID**, see warning below | `https://raw.githubusercontent.com/kboghdady/youTube_ads_4_pi-hole/master/youtubelist.txt` |
| kboghdady raw/unfiltered list — **AVOID** | `.../master/black.list` |

---

## ⚠️ Critical gotchas / lessons learned

1. **`blocklists.txt` = full source of truth.** Any domain in your NextDNS
   denylist that isn't covered by a URL listed in `blocklists.txt` gets
   deleted as "stale" on every run. Never add/remove domains by hand in the
   NextDNS dashboard or via ReNXEnhanced once this automation is live —
   **edit the GitLab file instead.**

2. **`youtubelist.txt` is NOT actually playback-safe**, despite the repo's
   README claiming it excludes risky domains. Verified directly: 15,854 of
   its 15,866 lines are `googlevideo.com` CDN hosts (YouTube's actual video
   servers, not just ad servers). Only 12 lines are genuine ad domains.
   Blocking these can break YouTube playback (buffering, looping, errors).
   **Do not add this URL to `blocklists.txt`.**

3. **One API key = your entire NextDNS account, not one profile.** The
   script auto-discovers every profile under the account tied to
   `NEXTDNS_ACCOUNT_1_API_KEY` and syncs ALL of them to the same
   `blocklists.txt` content, unless the workflow file is manually edited.
   Discovered profiles on this account: `cd136e` and `111a83`.

4. **To scope to specific profile(s):** edit
   `.github/workflows/update-nextdns.yml`. The original line was:
   ```
   profiles=$(nextdnsctl profile-list 2>/dev/null | grep -oP '^[a-z0-9]+(?=:)' | grep -v '^$')
   ```
   **This line has already been deleted** and replaced with a hardcoded
   value (see "Profile scoping — RESOLVED" below for current state). To
   list multiple profiles, use a space-separated string, e.g.:
   ```
   profiles="cd136e 111a83"
   ```
   All listed profiles get the **same** combined blocklist — this script
   can't give different profiles different denylists.

5. **NextDNS API rate limits are aggressive** for large domain counts —
   expect repeated "Rate limit hit... pausing for 60s" retries. A few
   hundred domains: fast (~1–2 min). Tens of thousands: can take a very
   long time. This is a reason to keep the source list small and curated
   rather than importing huge upstream lists.

6. **Forked repos have GitHub Actions disabled by default** — must
   manually enable from the Actions tab after forking.

7. **Uncommitted edits don't save.** Always re-open a file after editing
   to confirm the change actually persisted before assuming it worked.

8. **60-day inactivity auto-disables scheduled workflows** on a fork (both
   the sync and cleanup workflows). A commit or manual trigger resets the
   clock.

---

## Current `blocklists.txt` (as of last confirmed state)

```
# One blocklist URL per line
# Lines starting with # are ignored
https://gitlab.com/bangobang/ytblk/-/raw/main/NextDNS-Denylist.txt?ref_type=heads
https://raw.githubusercontent.com/kboghdady/youTube_ads_4_pi-hole/refs/heads/master/youtubelist.txt
```

⚠️ Deliberately kept despite the googlevideo.com warning above — decision
made to test real-world YouTube playback impact directly rather than avoid
it preemptively. ~14,228 additional domains (mostly googlevideo.com CDN
hosts) added to `111a83`'s denylist as a result. **If playback issues
appear, remove the kboghdady line and re-run/wait for next sync to
auto-prune them back out.**

(ph00lt0 removed. You already use OISD via NextDNS's built-in
Privacy/Blocklists tab, which is unaffected by any of this and requires no
automation.)

---

## Ongoing maintenance workflow

**To add a domain to the denylist:**
1. Edit `NextDNS-Denylist.txt` in your GitLab repo (`bangobang/ytblk`)
2. Add the domain — bare domain only, no `*.` wildcard prefix, no
   `googlevideo.com` hosts
3. Commit
4. Either wait for the daily scheduled run, or trigger manually:
   Actions tab → **Update NextDNS Denylist** → Run workflow

**To remove a domain:** same steps, delete the line instead — the next
sync will prune it from NextDNS automatically.

**Do not** add/remove domains directly in the NextDNS dashboard or via
ReNXEnhanced for any profile covered by this automation — it will be
reverted on the next sync.

---

## Testing YouTube after this change

**What to check:**
- Play several different videos on the device(s)/app using the `111a83`
  profile
- Watch for: indefinite buffering, playback looping/restarting,
  resolution constantly downgrading, "an error occurred" messages
- Try scrubbing the timeline mid-video — this is often where a blocked
  CDN host shows up first, since scrubbing requests a fresh segment that
  may come from a different (possibly now-blocked) server
- Test more than one video, since the specific CDN servers assigned can
  vary between videos and sessions
- If nothing seems affected but you tested immediately after the run
  finished, note that DNS caching (on NextDNS's side and/or the local
  device/router) can delay how quickly a denylist change is actually
  reflected — worth flushing DNS cache or waiting a few minutes before
  concluding either way

**If playback breaks, revert like this:**
1. Remove the kboghdady `youtubelist.txt` line from `blocklists.txt` in
   your fork (keep only your GitLab URL)
2. Commit the change
3. Either wait for the next scheduled 02:00 UTC run, or trigger manually:
   Actions tab → **Update NextDNS Denylist** → Run workflow
4. The next run will see those ~14,228 domains are no longer in any
   source and prune them back out of `111a83` automatically (subject to
   the same rate-limiting delays discussed above)

## Schedules

| Workflow | Cron (UTC) | Purpose |
|---|---|---|
| Update NextDNS Denylist | `0 2 * * *` (02:00 UTC daily) | Syncs blocklists.txt sources → NextDNS denylist for listed profiles |
| Cleanup old NextDNS Denylist runs | `0 3 * * *` (03:00 UTC daily) | Deletes old Actions run history, keeps only the newest run per workflow |

---

## Profile scoping — RESOLVED

The original auto-discovery line in `update-nextdns.yml`:
```bash
profiles=$(nextdnsctl profile-list 2>/dev/null | grep -oP '^[a-z0-9]+(?=:)' | grep -v '^$')
```
**has been deleted** and replaced with:
```bash
profiles="111a83"
```

**Final decision: automation manages `111a83` only.**

`cd136e` — the profile all prior domain curation (315 original + 3 from
black_list.txt = 318 total) was originally based on — is now **completely
untouched** by this automation. It will not be pruned, added to, or
otherwise affected by any future scheduled or manual runs of this
workflow. Its denylist remains whatever was last set on it manually
(before this automation existed), frozen in place unless edited by hand
again in the NextDNS dashboard or via ReNXEnhanced.

If you ever want `cd136e` brought into this automation too, change the
line to:
```bash
profiles="111a83 cd136e"
```
— but note `cd136e` would then need its own domains represented somewhere
in `blocklists.txt`'s sources first, or its existing denylist content
would be stale-pruned on the first run (same risk as originally discussed
for this profile at the start of this process).
