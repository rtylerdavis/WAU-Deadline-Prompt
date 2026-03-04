## Feature Proposal: Update Deadline Enforcement with User Prompt

### The Problem

WAU updates apps silently, which is great for most scenarios. But in managed enterprise environments, admins often need to:

- Give users advance notice before updates install
- Let users defer to a convenient time (not mid-presentation)
- Enforce a hard deadline after which updates install automatically
- Meet organizational patching SLAs (e.g., "all apps updated within 7 days")

Tools like WSUS, Intune, and SCCM all offer deadline-based enforcement. WAU currently lacks this, which limits its usefulness as a primary patching tool in enterprises.

This also addresses the request in #1121 (User Notification: Allow Deferrals) -- the implementation provides deferral via a time-based snooze with a configurable reminder interval, plus a hard deadline after which updates are forced.

### The Solution

I've built a working implementation in my fork: https://github.com/rtylerdavis/WAU-Deadline-Prompt

When enabled via a new ADMX policy (`WAU_UpdateDeadlineDays`), WAU tracks when it first detects each pending update and calculates a deadline. Instead of silently installing, it shows a WPF dialog to the logged-in user:

![image](https://github.com/user-attachments/assets/placeholder)
*(WPF dialog listing pending updates with deadline dates, "Update Now" and "Remind Me" buttons, 5-minute auto-dismiss countdown)*

**User actions:**
- **Update Now** -- triggers a new scheduled task that runs updates immediately using WAU's existing `Update-App` function
- **Remind Me in X Days** -- snoozes the prompt (interval configurable via ADMX)
- **Close / Timeout** -- treated as snooze
- **Deadline passes** -- updates install silently on next WAU run, no dialog

**When disabled** (`WAU_UpdateDeadlineDays = 0` or not configured), WAU behaves identically to today -- fully backward compatible.

### New ADMX Settings

| Setting | Default | Description |
|---|---|---|
| `WAU_UpdateDeadlineDays` | `0` (disabled) | Days from first detection until forced update |
| `WAU_ReminderIntervalDays` | `2` | Days between reminder prompts |
| `WAU_CompanyName` | *(empty)* | Custom name in the prompt header (e.g., "Contoso requires...") |

### Implementation Details

- **5 new files**, **4 modified files** -- all changes are additive
- **2 new scheduled tasks**: `UpdatePrompt` (WPF dialog via ServiceUI.exe) and `UpdateNow` (runs updates)
- Reuses existing WAU patterns throughout: `Update-App`, `Confirm-Installation`, `Start-NotifTask`, `Write-ToLog`, `Get-WAUConfig`
- No changes needed to `Get-WAUConfig.ps1` (reads registry dynamically) or `WAU-Policies.ps1`
- Deadline tracking stored in registry under `HKLM:\...\Winget-AutoUpdate\UpdateDeadlines\[AppId]`
- Respects blacklist/whitelist filtering
- Self-healing: crashed prompt re-shown next run, crashed update preserves data for retry
- Tested on physical hardware with Windows 11 Pro, MSI install, PowerShell 5.1

A full technical write-up is in the fork's [UPSTREAM-PROPOSAL.md](https://github.com/rtylerdavis/WAU-Deadline-Prompt/blob/master/UPSTREAM-PROPOSAL.md).

### Interest?

I'd love to contribute this upstream if there's interest. Happy to:
- Open a PR against your repo
- Adapt the implementation to fit your preferred patterns or conventions
- Split it into smaller PRs if that's easier to review

Let me know what you think!
