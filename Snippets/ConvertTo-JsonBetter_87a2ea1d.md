<!--
id: 87a2ea1d
title: ConvertTo-JsonBetter
language: powershell
tags: function, json
description: Converts objects to a prettified JSON string with improved formatting
created: 2026-04-18T12:46:41.659Z
updated: 2026-04-18T14:48:41
-->

# ConvertTo-JsonBetter

> Converts objects to a prettified JSON string with improved formatting

<p><code>function</code> <code>json</code></p>

```powershell
function ConvertTo-JsonBetter {
    <#
        .SYNOPSIS
            Converts objects to a prettified JSON string with improved formatting.

        .DESCRIPTION
            A wrapper around ConvertTo-Json providing consistent, pretty-printed output.
            - PowerShell 7+: uses the native -EscapeHandling parameter.
            - PowerShell 5.1: applies a regex-based indentation prettifier and unescapes
            characters that ConvertTo-Json escapes by default.

            Credits for PS5 prettifier logic:
            https://stackoverflow.com/questions/56322993/proper-formating-of-json-using-powershell

        .PARAMETER InputObject
            The object to convert to JSON.

        .PARAMETER Depth
            How many levels of nested objects to include. Default: 2.

        .PARAMETER Indentation
            Number of spaces per indentation level (1–10). Default: 4.
            Ignored on PowerShell 7+ (native formatting is used instead).

        .PARAMETER NewLineDelimiter
            Line ending style: 'CRLF' (Windows) or 'LF' (Unix).
            Auto-detected from the output if not specified.

        .EXAMPLE
            $MyObject | ConvertTo-JsonBetter -Depth 10

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2026-04-16

            Version history:
            1.0.0 - (2026-04-16) Function created
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true, ValueFromPipeline = $true)]
        $InputObject,

        [ValidateRange(1, 100)]
        [int]$Depth = 2,

        [ValidateRange(1, 10)]
        [int]$Indentation = 4,

        [ValidateSet('CRLF', 'LF')]
        [string]$NewLineDelimiter
    )

    begin {
        $DelimiterMap = @{
            CRLF = "`r`n"
            LF   = "`n"
        }

        # Returns the target line ending: user-specified or auto-detected from the string
        function Get-ActualDelimiter([string]$Text) {
            if ($PSBoundParameters.ContainsKey('NewLineDelimiter')) {
                return $DelimiterMap[$NewLineDelimiter]
            }
            if ($Text -match "`r`n") { return "`r`n" }
            return "`n"
        }
    }

    process {
        if ($PSVersionTable.PSVersion.Major -ge 7) {
            if ($PSBoundParameters.ContainsKey('Indentation')) {
                Write-Warning 'The -Indentation parameter is ignored in PowerShell 7+ (native formatting is used).'
            }

            $Json = $InputObject | ConvertTo-Json -Depth $Depth -EscapeHandling Unescape

            # Normalize line endings only when explicitly requested
            if ($PSBoundParameters.ContainsKey('NewLineDelimiter')) {
                $DetectedDelimiter = if ($Json -match "`r`n") { "`r`n" } else { "`n" }
                $TargetDelimiter   = $DelimiterMap[$NewLineDelimiter]

                if ($DetectedDelimiter -ne $TargetDelimiter) {
                    $Json = $Json -replace $DetectedDelimiter, $TargetDelimiter
                }
            }

            return $Json
        }

        # --- PowerShell 5.1 fallback ---
        $Json = $InputObject | ConvertTo-Json -Depth $Depth

        $IndentLevel        = 0
        $ActualDelimiter    = Get-ActualDelimiter $Json

        # Matches a pattern only when outside of quoted strings
        $RegexUnlessQuoted  = '(?=([^"]*"[^"]*")*[^"]*$)'

        $FormattedJson = ($Json.Split("`n") | ForEach-Object {
            $Line = $_.Trim()
            if ([string]::IsNullOrWhiteSpace($Line)) { return }

            if ($Line -match "[}\]]$RegexUnlessQuoted") {
                $IndentLevel = [Math]::Max($IndentLevel - $Indentation, 0)
            }

            $OutputLine = (' ' * $IndentLevel) + ($Line -replace ":\s+$RegexUnlessQuoted", ': ')

            if ($Line -match "[\{\[]$RegexUnlessQuoted") {
                $IndentLevel += $Indentation
            }

            $OutputLine
        }) -join $ActualDelimiter

        # Unescape sequences that ConvertTo-Json escapes by default in PS5
        $FormattedJson = $FormattedJson `
            -replace '\\r\\n', "`r`n" `
            -replace '\\n',    "`n"   `
            -replace '\\u003c','<'    `
            -replace '\\u003e','>'    `
            -replace '\\u0027',"'"    `
            -replace '\\u0026','&'

        return $FormattedJson
    }
}
```