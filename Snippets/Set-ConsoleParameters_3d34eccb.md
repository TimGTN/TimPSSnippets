<!--
id: 3d34eccb
title: Set-ConsoleParameters
language: powershell
tags: console, function
description: Change console window parameters
created: 2026-04-17T08:27:00.686Z
updated: 2026-04-17T10:53:00
-->

# Set-ConsoleParameters

> Change console window parameters

<p><code>console</code> <code>function</code></p>

```powershell
function Set-ConsoleParameters {
    <#
    .SYNOPSIS
        Change console window parameters.

    .DESCRIPTION
        Change console window visibility and/or QuickEdit mode.

    .PARAMETER WindowState
        Specify the desired window state.

    .PARAMETER DisableQuickEdit
        Disable QuickEdit mode (prevents console from pausing on click).

    .PARAMETER EnableQuickEdit
        Enable QuickEdit mode.

    .LINK
        https://stackoverflow.com/questions/30872345/script-commands-to-disable-quick-edit-mode
        https://stackoverflow.com/questions/40617800/opening-powershell-script-and-hide-command-prompt-but-not-the-gui

    .NOTES
        Author:      Tim GILLOTIN
        Contact:     @TimGTN
        Created:     2025-02-25

        Version history:
        1.0.0 - (2025-02-25) Function created
    #>
    [CmdletBinding()]
    param(
        [ValidateSet("Hide","ShowNormal","ShowMinimized","ShowMaximized","Maximize","ShowNormalNoActivate","Show","Minimize","ShowMinNoActivate","ShowNoActivate","Restore","ShowDefault","ForceMinimized")]
        [Parameter()]
        [string]$WindowState,

        [Parameter()]
        [switch]$DisableQuickEdit,

        [Parameter()]
        [switch]$EnableQuickEdit
    )
    # Load native methods
    Add-Type -TypeDefinition '
        using System;
        using System.Runtime.InteropServices;
        public class ConsoleHelper {
            [DllImport("kernel32.dll")] public static extern IntPtr GetConsoleWindow();
            [DllImport("kernel32.dll")] public static extern IntPtr GetStdHandle(int nStdHandle);
            [DllImport("kernel32.dll")] public static extern bool GetConsoleMode(IntPtr hConsoleHandle, out uint lpMode);
            [DllImport("kernel32.dll")] public static extern bool SetConsoleMode(IntPtr hConsoleHandle, uint dwMode);
            [DllImport("user32.dll")]   public static extern bool ShowWindow(IntPtr hWnd, Int32 nCmdShow);
        }
    ' -ErrorAction SilentlyContinue

    # Window state
    if ($WindowState) {
        $StatesMapping = [ordered]@{
            Hide = 0
            ShowNormal = 1
            ShowMinimized = 2
            ShowMaximized = 3
            Maximize = 3
            ShowNormalNoActivate = 4
            Show = 5
            Minimize = 6
            ShowMinNoActivate = 7
            ShowNoActivate = 8
            Restore = 9
            ShowDefault = 10
            ForceMinimized = 11
        }

        $ConsoleHandle = [ConsoleHelper]::GetConsoleWindow()
        [ConsoleHelper]::ShowWindow($ConsoleHandle, $StatesMapping[$WindowState]) | Out-Null
    }

    # QuickEdit mode
    if ($DisableQuickEdit -or $EnableQuickEdit) {
        $ENABLE_QUICK_EDIT    = 0x0040
        $ENABLE_EXTENDED_FLAGS = 0x0080
        $ENABLE_MOUSE_INPUT = 0x0010

        $Handle = [ConsoleHelper]::GetStdHandle(-10) # STD_INPUT_HANDLE
        $Mode = 0
        [ConsoleHelper]::GetConsoleMode($Handle, [ref]$Mode) | Out-Null

        if ($DisableQuickEdit) {
            $Mode = $Mode -band (-bnot $ENABLE_QUICK_EDIT)
            $Mode = $Mode -bor $ENABLE_MOUSE_INPUT
        } elseif ($EnableQuickEdit) {
            $Mode = $Mode -bor $ENABLE_QUICK_EDIT
        }

        $Mode = $Mode -bor $ENABLE_EXTENDED_FLAGS
        [ConsoleHelper]::SetConsoleMode($Handle, $Mode) | Out-Null
    }
}
```