<!--
id: 376fa54c
title: ConvertFrom-JwtToken
language: powershell
tags: jwt, auth, token
description: Decodes a JSON Web Token (JWT) and returns its Header and Payload as a PSCustomObject
created: 2026-04-17T06:45:00.045Z
updated: 2026-04-17T08:46:36
-->

# ConvertFrom-JwtToken

> Decodes a JSON Web Token (JWT) and returns its Header and Payload as a PSCustomObject

<p><code>jwt</code> <code>auth</code> <code>token</code></p>

```powershell
function ConvertFrom-JwtToken {
<#
    .SYNOPSIS
        Decodes a JSON Web Token (JWT) and returns its Header and Payload as a PSCustomObject.

    .DESCRIPTION
        Splits a JWT string into its three parts (Header, Payload, Signature), Base64Url-decodes
        the first two segments, and deserializes them from JSON.
        The Signature segment is intentionally ignored as it requires asymmetric key verification.

    .PARAMETER Jwt
        A valid JWT string in the format: Base64Url(Header).Base64Url(Payload).Signature
        Must start with 'eyJ' and contain exactly three dot-separated segments.

    .OUTPUTS
        PSCustomObject with two properties:
            - Header  : decoded JWT header claims
            - Payload : decoded JWT payload claims

    .EXAMPLE
        $token = ConvertFrom-JwtToken -Jwt $bearerToken
        $token.Payload.exp   # expiry Unix timestamp
        $token.Payload.sub   # subject claim
#>
    [OutputType([PSCustomObject])]
    [CmdletBinding()]
    param (
        [Parameter(Mandatory)]
        [ValidateScript({
            # Must start with 'eyJ' and have exactly three dot-separated segments
            if ($_.StartsWith('eyJ') -and ($_.Split('.').Count -eq 3)) { $true }
            else { throw "Invalid JWT string: must start with 'eyJ' and contain exactly 3 segments." }
        })]
        [string] $Jwt
    )

    # Helper: converts a Base64Url segment → UTF-8 string → PSObject
    $DecodeSegment = {
        param([string] $Segment)

        # Base64Url → Base64 (replace URL-safe chars, then pad to a multiple of 4)
        $base64 = $Segment.Replace('-', '+').Replace('_', '/')
        $base64 = $base64.PadRight($base64.Length + (4 - $base64.Length % 4) % 4, '=')

        [System.Text.Encoding]::UTF8.GetString(
            [Convert]::FromBase64String($base64)
        ) | ConvertFrom-Json
    }

    # Split once and reuse — avoids calling Split() repeatedly
    $segments = $Jwt.Split('.')

    return [PSCustomObject]@{
        Header  = & $DecodeSegment $segments[0]
        Payload = & $DecodeSegment $segments[1]
        # Segment[2] is the cryptographic signature — skipped (cannot be verified without the key)
    }
}
```