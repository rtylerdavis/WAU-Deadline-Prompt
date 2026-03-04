# Update Deadline Enforcement for Winget-AutoUpdate

**Fork:** https://github.com/rtylerdavis/WAU-Deadline-Prompt

## Problem

WAU currently updates apps silently in the background with no user visibility or control over timing. In managed enterprise environments, admins often need to:

- Give users advance notice that updates are coming
- Allow users to defer updates to a convenient time (e.g., not during a presentation)
- Enforce a hard deadline after which updates install automatically
- Comply with organizational patching SLAs that require updates within N days

Other enterprise update tools (WSUS, Intune, SCCM) all provide deadline-based enforcement with user prompts. WAU lacks this capability, limiting its usefulness as a primary patching tool in managed environments.

## Solution

This fork adds an **update deadline enforcement system** controlled by two new ADMX-configurable registry settings. When enabled, WAU tracks when it first detects each pending update and calculates a deadline. Instead of silently installing, it presents a WPF dialog to the logged-in user showing pending updates with their deadlines. Users can choose to update immediately or snooze. Once the deadline passes, updates install silently with no prompt.

### User-Facing Experience

When updates are detected with deadline mode enabled:

1. A WPF dialog appears listing all pending updates with columns: Application, Available Version, Required By (date), and Days Remaining
2. Apps within 3 days of their deadline are highlighted in amber
3. User can click **"Update Now"** to install all updates immediately
4. User can click **"Remind Me in X Days"** to snooze (configurable interval)
5. Closing the window or letting the 5-minute timeout expire is treated as a snooze
6. Once a deadline passes, the app is updated silently on the next WAU run with no dialog

### Configuration

Two new ADMX policy settings (with full ADMX/ADML definitions included):

| Setting | Type | Default | Description |
|---|---|---|---|
| `WAU_UpdateDeadlineDays` | REG_SZ (integer) | `0` (disabled) | Days from first detection until forced update. `0` = current silent behavior. |
| `WAU_ReminderIntervalDays` | REG_SZ (integer) | `2` | Days between reminder dialogs when user snoozes. |
| `WAU_CompanyName` | REG_SZ (string) | *(empty)* | Custom company name for the prompt header (e.g., "Contoso requires the following updates..."). Falls back to "Your organization" if not set. |

### How It Works

**Modified file: `Winget-Upgrade.ps1`**

The deadline system wraps the existing main update loop in an if/else branch. When `WAU_UpdateDeadlineDays > 0`:

- The existing `foreach ($app in $outdated) { Update-App ... }` loop is **skipped entirely**
- Deadline logic takes over: syncs registry entries, splits apps into overdue vs. pending, handles each group appropriately
- When deadline mode is disabled (`= 0`), all deadline registry data is purged and the existing silent behavior runs unchanged
- Blacklist/whitelist filtering is respected -- only eligible apps enter the deadline flow

**Registry structure for deadline tracking:**

```
HKLM:\SOFTWARE\Romanitho\Winget-AutoUpdate\UpdateDeadlines\
  [AppID]                          (subkey per app)
    FirstDetected    = "2026-02-27"
    Deadline         = "2026-03-02"
    AvailableVersion = "130.0.1"
```

- New app detected: creates entry with `Deadline = today + DeadlineDays`
- Known app, new version available: updates `AvailableVersion` only (deadline does NOT reset)
- App no longer outdated (user self-updated): purges entry automatically
- Deadline mode disabled: all entries purged on next run (prevents stale deadlines from causing instant forced updates if re-enabled later)

**Snooze tracking:**

```
HKLM:\SOFTWARE\Romanitho\Winget-AutoUpdate\
  NextPromptTime = "2026-03-01T10:00:00.0000000+00:00"
```

Machine-wide (HKLM) -- on shared machines, one user's snooze applies to all users. This is documented in the ADMX policy explanation text.

### New Files

| File | Purpose |
|---|---|
| `functions/Get-UpdateDeadlines.ps1` | Reads deadline registry entries, purges stale ones (apps no longer outdated) |
| `functions/Set-UpdateDeadline.ps1` | Creates or updates a deadline entry for an app |
| `functions/Start-UpdatePromptTask.ps1` | Writes `pending-updates.json`, triggers the UpdatePrompt scheduled task |
| `WAU-UpdatePrompt.ps1` | WPF dialog script -- runs as SYSTEM via ServiceUI.exe in user's desktop session |
| `WAU-UpdateNow.ps1` | Triggered by "Update Now" button -- loads WAU function library, calls `Update-App` for each pending app, purges completed entries |

### New Scheduled Tasks

Both registered in `WAU-MSI_Actions.ps1` under TaskPath `WAU`:

**Winget-AutoUpdate-UpdatePrompt** -- Displays the WPF dialog
- Runs as SYSTEM (S-1-5-18), RunLevel Highest
- Uses ServiceUI.exe to display in the logged-in user's desktop session
- Uses `-EncodedCommand` to avoid path quoting issues with ServiceUI's `CreateProcessAsUser`
- On-demand only (triggered by the main WAU task via `Start-UpdatePromptTask`)
- MultipleInstances: IgnoreNew

**Winget-AutoUpdate-UpdateNow** -- Executes the updates
- Runs as SYSTEM (S-1-5-18), RunLevel Highest
- On-demand only (triggered by the UpdatePrompt dialog's "Update Now" button)
- Loads WAU's full function library and calls the existing `Update-App` function (preserves logging, notifications, mods support)
- MultipleInstances: IgnoreNew

Both task names are added to the uninstall array in `Uninstall-WingetAutoUpdate` for clean removal.

### Modified Files

| File | Change |
|---|---|
| `Winget-Upgrade.ps1` | Deadline-mode if/else branch wrapping main update loop |
| `WAU-MSI_Actions.ps1` | Register two new scheduled tasks, add to uninstall array |
| `WAU.admx` | Three new policy definitions |
| `en-US/WAU.adml` | Display strings and presentation entries for new policies |

### Design Decisions

- **No changes to `Get-WAUConfig.ps1`** -- it reads all registry properties dynamically, so new `WAU_` keys appear automatically
- **No changes to `WAU-Policies.ps1`** -- it only manages schedule triggers, which are unaffected
- **Uses existing WAU patterns throughout**: `Start-NotifTask` for overdue toasts, `Update-App` + `Confirm-Installation` for update execution, `Get-CimInstance` for logged-in user detection, `Write-ToLog` for logging
- **Self-healing**: crashed prompt is re-shown next run, crashed UpdateNow preserves JSON for retry, corrupt dates are purged and recreated
- **User context task**: when deadline mode is active, the UserContext task logs a skip message instead of running updates (prevents bypassing the deadline system)

### Data File

`config/pending-updates.json` -- written by the SYSTEM task, read by both UpdatePrompt and UpdateNow, deleted by UpdateNow on completion:

```json
{
  "Config": {
    "ReminderIntervalDays": 2,
    "CompanyName": "Contoso"
  },
  "Apps": [
    {
      "Name": "Google Chrome",
      "Id": "Google.Chrome",
      "Version": "129.0.0",
      "AvailableVersion": "130.0.1",
      "Deadline": "2026-03-14"
    }
  ]
}
```

## Testing

Tested on physical hardware with:
- Windows 11 Pro
- WAU installed via MSI to `C:\Program Files\Winget-AutoUpdate\`
- Intentionally outdated apps (RustDesk, Chrome)
- Deadline enforcement with blacklist filtering
- Full prompt lifecycle: dialog display, Remind Me (snooze), Update Now (immediate install)
- Custom company name and window icon
- Registry entry creation, deadline calculation, and automatic purge on successful update

## Compatibility

- Fully backward compatible -- when `WAU_UpdateDeadlineDays` is `0` or not configured, WAU behaves identically to upstream
- No changes to existing function signatures or return values
- All new files are additive
- ADMX changes are additive (new policies only, no modifications to existing ones)
- Tested with PowerShell 5.1 (Windows PowerShell) -- all files use ASCII-only characters to avoid BOM/encoding issues
