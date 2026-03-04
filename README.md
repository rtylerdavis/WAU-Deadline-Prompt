# WAU Deadline Prompt

An unofficial fork of [Winget-AutoUpdate](https://github.com/Romanitho/Winget-AutoUpdate) by Romanitho that adds update deadline enforcement with a user-facing prompt dialog. In managed enterprise environments, admins need users to be notified before updates install and given the option to defer -- but with a hard deadline to ensure compliance. This fork adds that capability while remaining fully backward compatible with the upstream WAU behavior when not configured.

![Update Prompt Dialog](https://github.com/user-attachments/assets/32176e88-403d-44c1-86d1-4a7af432e13b)

When enabled, WAU tracks the first time it detects a pending update and calculates a deadline. Instead of silently installing, it presents a WPF dialog to the logged-in user listing all pending updates with their deadlines. Users can click **Update Now** to install immediately, or **Remind Me** to snooze. Once the deadline passes, updates install silently in the background with no prompt.

## Configuration

All settings are configurable via ADMX Group Policy or direct registry entries under:

```
HKLM:\SOFTWARE\Policies\Romanitho\Winget-AutoUpdate   (GPO)
HKLM:\SOFTWARE\Romanitho\Winget-AutoUpdate             (direct)
```

| Setting | Type | Default | Description |
|---|---|---|---|
| `WAU_UpdateDeadlineDays` | REG_SZ | `0` (disabled) | Number of days from first detection until the update is forced. Setting this to `0` or leaving it unconfigured preserves the default silent update behavior. |
| `WAU_ReminderIntervalDays` | REG_SZ | `2` | Number of days to wait before showing the prompt again after the user dismisses it. Only active when `WAU_UpdateDeadlineDays` is greater than 0. |
| `WAU_CompanyName` | REG_SZ | *(empty)* | Custom company name displayed in the prompt header. For example, setting this to `Contoso` changes the header to "Contoso requires the following updates to be installed." If not set, defaults to "Your organization." |

ADMX/ADML policy definition files are included under `Sources/Policies/ADMX/` for deployment via Group Policy or Intune Administrative Templates.

## Future State

- Intune/Managed Deployment testing needed with ADMX integration
- Potentially moving upstream to Romanitho's WAU - integrating a solution for auto update in the interim
- Adding logic for per-app updates, rather than all or nothing
