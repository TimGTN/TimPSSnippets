<!--
id: dd50001b
title: Get-UserNotificationState
language: powershell
tags: function, notification, user
description: Returns the current Windows user notification state via SHQueryUserNotificationState
created: 2026-04-17T21:40:07.650Z
updated: 2026-04-17T23:40:35
-->

# Get-UserNotificationState

> Returns the current Windows user notification state via SHQueryUserNotificationState

<p><code>function</code> <code>notification</code> <code>user</code></p>

```powershell
function Get-UserNotificationState {
    <#
        .SYNOPSIS
            Returns the current Windows user notification state via SHQueryUserNotificationState.

        .PARAMETER ListStates
            Returns the full mapping of state codes to their QUNS_* names.

        .PARAMETER Resolve
            Returns the QUNS_* name instead of the numeric state code.

        .EXAMPLE
            Get-UserNotificationState

        .EXAMPLE
            Get-UserNotificationState -Resolve

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2026-04-17

            Version history:
            1.0.0 - (2026-04-17) Function created
    #>
    [CmdletBinding()]
    param(
        [switch]$ListStates,
        [switch]$Resolve
    )

    $NotificationStates = [ordered]@{
        1 = "QUNS_NOT_PRESENT"
        2 = "QUNS_BUSY"
        3 = "QUNS_RUNNING_D3D_FULL_SCREEN"
        4 = "QUNS_PRESENTATION_MODE"
        5 = "QUNS_ACCEPTS_NOTIFICATIONS"
        6 = "QUNS_QUIET_TIME"
        7 = "QUNS_APP"
    }

    if ($ListStates) { return $NotificationStates }

    $Source = @"
        using System;
        using System.Runtime.InteropServices;
        public static class UserNotificationHelper
        {
            [DllImport("shell32.dll")]
            private static extern int SHQueryUserNotificationState(out int state);
            public static int QueryUserNotificationState()
            {
                int state;
                int result = SHQueryUserNotificationState(out state);
                if (result == 0)
                {
                    return state;
                }
                else
                {
                    throw new InvalidOperationException("Failed to query user notification state. Error code: " + result);
                }
            }
        }
"@

    if (-not ('UserNotificationHelper' -as [type])) {
        Add-Type -TypeDefinition $Source -Language CSharp
    }

    try {
        [int]$Return = [UserNotificationHelper]::QueryUserNotificationState()
        if ($Resolve) { return $NotificationStates[$Return] }
        return $Return
    
    } catch {
        $PSCmdlet.WriteError($_)
    }
}
```