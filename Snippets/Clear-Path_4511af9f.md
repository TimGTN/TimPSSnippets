<!--
id: 4511af9f
title: Clear-Path
language: powershell
tags: function, path
description: Sanitizes a file system path by replacing invalid characters with underscores
created: 2026-04-17T08:04:20.014Z
updated: 2026-04-17T10:05:59
-->

# Clear-Path

> Sanitizes a file system path by replacing invalid characters with underscores

<p><code>function</code> <code>path</code></p>

```powershell
function Clear-Path {
<#
    .SYNOPSIS
        Sanitizes a file system path by replacing invalid characters with underscores.

    .DESCRIPTION
        Processes each segment of the given path individually, replacing characters that are
        invalid in file or directory names with underscores.

        The path separator is auto-detected from the input using [System.IO.Path] native methods,
        with the OS default ([System.IO.Path]::DirectorySeparatorChar) as fallback.
        It can also be overridden explicitly via -PathSeparator.

        Diacritical marks (e.g. é → e, ü → u) are optionally removed using Unicode
        normalization (FormD), which is more reliable than encoding-based approaches.
    
    .PARAMETER Path
        The full path to sanitize. Accepts pipeline input. Must not be null or empty.
    
    .PARAMETER RemoveDiacritics
        When specified, strips diacritical marks via Unicode normalization (FormD).
    
    .PARAMETER PathSeparator
        Overrides the output path separator ("/" or "\").
        If omitted, the separator is detected from the input, falling back to the OS default.
  
    .OUTPUTS
        System.String — The sanitized path.

    .EXAMPLE
        Clear-Path -Path "C:\Users\Te*st\Héllo Wörld" -RemoveDiacritics
        # Returns: C:\Users\Rene\Hello World
 
    .NOTES
        Author:      Tim GILLOTIN
        Contact:     @TimGTN
        Created:     2024-10-25

        Version history:
        1.0.0 - (2024-10-25) Function created
        2.0.0 - (2026-04-17) Auto-detect separator via .NET, Unicode normalization,
                             pipeline support, structured error handling
#>
    [OutputType([string])]
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [ValidateNotNullOrEmpty()]
        [string]$Path,

        [switch]$RemoveDiacritics,

        [ValidateSet('/', '\')]
        [string]$PathSeparator
    )

    begin {
        $InvalidChars = [System.IO.Path]::GetInvalidFileNameChars()
        $OsSeparator  = [System.IO.Path]::DirectorySeparatorChar
    }

    process {
        try {
            # --- Diacritics removal via Unicode normalization (FormD) ---
            # FormD decomposes characters into base letter + combining marks,
            # which can then be filtered out cleanly without lossy encoding tricks.
            if ($RemoveDiacritics) {
                $Path = -join (
                    $Path.Normalize([Text.NormalizationForm]::FormD).ToCharArray() |
                    Where-Object {
                        [Globalization.CharUnicodeInfo]::GetUnicodeCategory($_) -ne
                        [Globalization.UnicodeCategory]::NonSpacingMark
                    }
                )
            }

            # --- Auto-detect separator from the input, fallback to OS default ---
            if ($PSBoundParameters.ContainsKey('PathSeparator')) {
                $Separator = $PathSeparator
            } elseif ($Path.Contains('\')) {
                $Separator = '\'
            } elseif ($Path.Contains('/')) {
                $Separator = '/'
            } else {
                $Separator = $OsSeparator
            }

            # --- Sanitize each path segment individually ---
            $CleanSegments = $Path.Split($Separator) | ForEach-Object {
                if ([string]::IsNullOrEmpty($_)) {
                    # Preserve empty segments (e.g. leading "/" on Unix paths)
                    $_
                } elseif ($_ -match '^[a-zA-Z]:$') {
                    # Preserve Windows drive letters (e.g. "C:")
                    $_
                } else {
                    -join ($_.ToCharArray() | ForEach-Object {
                        if ($_ -in $InvalidChars) { '_' } else { $_ }
                    })
                }
            }

            return $CleanSegments -join $Separator
        }
        catch {
            $PSCmdlet.WriteError(
                [Management.Automation.ErrorRecord]::new(
                    [Exception]::new("Failed to sanitize path '$Path': $_"),
                    'ClearPathFailed',
                    [Management.Automation.ErrorCategory]::InvalidArgument,
                    $Path
                )
            )
        }
    }
}
```