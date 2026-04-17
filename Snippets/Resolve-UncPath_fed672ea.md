<!--
id: fed672ea
title: Resolve-UncPath
language: powershell
tags: function, unc, path
description: Resolves a mapped drive path to its UNC equivalent
created: 2026-04-17T21:10:02.562Z
updated: 2026-04-17T23:10:40
-->

# Resolve-UncPath

> Resolves a mapped drive path to its UNC equivalent

<p><code>function</code> <code>unc</code> <code>path</code></p>

```powershell
function Resolve-UncPath {
    <#
        .SYNOPSIS
            Resolves a mapped drive path to its UNC equivalent.

        .DESCRIPTION
            Gets a mapped drive root from Get-PsDrive to convert the path to a UNC path.
            ex. (\\domain.lan\share$) X:\Example → \\domain.lan\share$\Example
            Local drive paths (e.g. C:\) are returned as-is.

        .PARAMETER Path
            Path to resolve. Must be accessible.

        .OUTPUTS
            System.String

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2025-04-24

            Version history:
            1.0.0 - (2025-04-24) Function created
    #>
    [CmdletBinding()]
    param(
        [Parameter()]
        [ValidateScript({ if (Test-Path $_) { return $true } else { throw "Can't access specified path: $_" } })]
        [string]$Path
    )

    try {
        # UNC paths need no conversion
        if ($Path -like "\\*") {
            Write-Verbose "Path is already a UNC path, skipping conversion: $Path"
            return $Path
        }

        $Qualifier = Split-Path -Qualifier $Path
        Write-Verbose "Resolving mapped drive '$Qualifier' to UNC root..."

        $Drive = Get-PSDrive -PSProvider FileSystem | Where-Object { $_.Name -eq $Qualifier.TrimEnd(":") }

        if (-not $Drive) {
            Write-Warning "No PSDrive found matching qualifier '$Qualifier'. Returning original path."
            return $Path
        }

        # Local drive — return path as-is
        if ([string]::IsNullOrEmpty($Drive.DisplayRoot)) {
            Write-Verbose "Drive '$Qualifier' is a local drive, no UNC conversion needed."
            return $Path
        }

        Write-Verbose "Mapped drive root resolved: '$Qualifier' → '$($Drive.DisplayRoot)'"

        $Resolved = $Path.Replace($Qualifier, $Drive.DisplayRoot)
        Write-Verbose "Resolved path: $Resolved"

        if (-not (Test-Path $Resolved -ErrorAction Stop)) {
            throw "Can't access resolved path: $Resolved"
        }

        return $Resolved
    
    } catch {
        $PSCmdlet.WriteError($_)
    }
}
```