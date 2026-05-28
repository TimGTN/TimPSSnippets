<!--
id: ee0558fa
title: Invoke-SessionLogoff
language: powershell
tags: session, function, logoff
description: Logs off a user session on a remote computer
created: 2026-05-28T07:08:10.881Z
updated: 2026-05-28T09:08:41
-->

# Invoke-SessionLogoff

> Logs off a user session on a remote computer

<p><code>session</code> <code>function</code> <code>logoff</code></p>

```powershell
function Invoke-SessionLogoff {
    <#
        .SYNOPSIS
            Logs off a user session on a remote computer.

        .DESCRIPTION
            Runs the logoff command remotely via Invoke-Command and returns
            whether the operation succeeded based on the exit code.

        .PARAMETER ComputerName
            Name of the target computer.

        .PARAMETER SessionId
            Terminal Services session ID to log off.

        .OUTPUTS
            Boolean. $true if logoff succeeded, $false otherwise.

        .NOTES
            Author:  Tim GILLOTIN
            Contact: @TimGTN
            Created: 2025-05-28

            Version history:
            1.0.0 - (2025-05-28) Function created
    #>
    [CmdletBinding()]
    Param(
        [Parameter(Mandatory)]
        [string]$ComputerName,

        [Parameter(Mandatory)]
        [uint32]$SessionId
    )

    # Logoff
    $ExitCode = Invoke-Command -ComputerName $ComputerName -ScriptBlock {
        logoff $using:SessionId
        $LASTEXITCODE
    }
    return ($ExitCode -eq 0)
}
```