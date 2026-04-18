<!--
id: acb3597a
title: Test-TcpPortAvailability
language: powershell
tags: tcp, function, port
description: Tests whether a TCP port is available on the local machine
created: 2026-04-18T12:49:06.708Z
updated: 2026-04-18T14:49:37
-->

# Test-TcpPortAvailability

> Tests whether a TCP port is available on the local machine

<p><code>tcp</code> <code>function</code> <code>port</code></p>

```powershell
function Test-TcpPortAvailability {
    <#
        .SYNOPSIS
            Tests whether a TCP port is available on the local machine.

        .DESCRIPTION
            Returns $true if the port is free, $false if it is already in use.

        .PARAMETER Port
            The TCP port number to check.

        .EXAMPLE
            Test-TcpPortAvailability -Port 8081

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2026-04-13

            Version history:
            1.0.0 - (2026-04-13) Function created
    #>
    [CmdletBinding()]
    [OutputType([bool])]
    param (
        [Parameter(Mandatory)]
        [ValidateRange(1, 65535)]
        [int]$Port
    )
    $Properties  = [System.Net.NetworkInformation.IPGlobalProperties]::GetIPGlobalProperties()
    $Connections = $Properties.GetActiveTcpListeners()

    if ($Connections.Port -contains $Port) {
        Write-Verbose "Port $Port is in use."
        return $false
    } else {
        Write-Verbose "Port $Port is available."
        return $true
    }
}
```