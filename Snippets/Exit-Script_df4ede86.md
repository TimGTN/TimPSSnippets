<!--
id: df4ede86
title: Exit-Script
language: powershell
tags: exit, ise, function
description: Stops script execution cleanly in both normal and ISE/selection-run contexts
created: 2026-04-18T10:55:59.466Z
updated: 2026-04-18T12:57:41
-->

# Exit-Script

> Stops script execution cleanly in both normal and ISE/selection-run contexts

<p><code>exit</code> <code>ise</code> <code>function</code></p>

```powershell
function Exit-Script {
    <#
        .SYNOPSIS
            Stops script execution cleanly in both normal and ISE/selection-run contexts.

        .DESCRIPTION
            In ISE or when running a selected block, uses 'break _script_' to halt execution
            without closing the host. Since the label '_script_' never exists, PowerShell bubbles
            up the entire call stack and stops all execution cleanly.
            In a normal full-script run, exits the process with the provided exit code.

        .PARAMETER ExitCode
            The exit code to return when running as a full script. Defaults to 0.

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2024-10-03

            Version history:
            1.0.0 - (2024-10-03) Function created
    #>
    param([int]$ExitCode = 0)

    if ($psISE -or (Get-PSCallStack)[-1].Command -eq '<ScriptBlock>') {
        Write-Verbose "Exit-Script : Script would have stopped with exit code: $ExitCode"
        break __Exit-Script__
    }

    [System.Environment]::Exit($ExitCode)
}

```