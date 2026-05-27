<!--
id: a42edc04
title: Get-ActiveSession
language: powershell
tags: session, function
description: Returns all active interactive sessions on a remote computer
created: 2026-05-27T19:38:12.054Z
updated: 2026-05-27T21:38:51
-->

# Get-ActiveSession

> Returns all active interactive sessions on a remote computer

<p><code>session</code> <code>function</code></p>

```powershell
function Get-ActiveSession {
    <#
        .SYNOPSIS
            Returns all active interactive sessions on a remote computer.

        .DESCRIPTION
            Queries Win32_Process for explorer.exe instances and retrieves the owner
            of each process, giving both the username and the Terminal Services session ID.
            Returns one object per active session.

        .PARAMETER ComputerName
            Name of the target computer. Defaults to the local machine.

        .OUTPUTS
            PSCustomObject with UserName and SessionId properties.

        .NOTES
            Author:  Tim GILLOTIN
            Contact: @TimGTN
            Created: 2025-05-27

            Version history:
            1.0.0 - (2025-05-27) Function created
    #>
    [CmdletBinding()]
    Param(
        [Parameter()]
        [string]$ComputerName = "."
    )
    try {
        Get-WmiObject -ComputerName $ComputerName -Class Win32_Process `
                      -Filter "Name='explorer.exe'" -ErrorAction Stop |
        ForEach-Object {
            [PSCustomObject]@{
                UserName  = $_.GetOwner().User
                SessionId = $_.SessionId
            }
        }
    } catch {
        Write-Error $_
    }
}
```