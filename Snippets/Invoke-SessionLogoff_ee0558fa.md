<!--
id: ee0558fa
title: Invoke-SessionLogoff
language: powershell
tags: session, function, logoff
description: Logs off a user session on a remote computer
created: 2026-05-28T07:08:10.881Z
updated: 2026-05-28T09:14:35
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
            Runs the logoff command remotely via Invoke-Command and throws
            an exception if the operation fails.

        .PARAMETER ComputerName
            Name of the target computer.

        .PARAMETER SessionId
            Terminal Services session ID to log off.

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
    if ($ExitCode -ne 0) {
        throw "Session $SessionId logoff failed on '$ComputerName' (exitcode: $exitCode)"    
    }
}
```