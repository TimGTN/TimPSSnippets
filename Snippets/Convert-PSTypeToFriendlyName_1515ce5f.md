<!--
id: 1515ce5f
title: Convert-PSTypeToFriendlyName
language: powershell
tags: type, function
description: Returns the friendliest PowerShell name for a .NET type
created: 2026-04-25T11:28:39.695Z
updated: 2026-04-25T15:16:10
-->

# Convert-PSTypeToFriendlyName

> Returns the friendliest PowerShell name for a .NET type

<p><code>type</code> <code>function</code></p>

```powershell
function Convert-PSTypeToFriendlyName {
    <#
    .SYNOPSIS
        Returns the friendliest PowerShell name for a .NET type.
        Priority: accelerator alias → array → generic → full CLR name.

    .NOTES
        Author:      Tim GILLOTIN
        Contact:     @TimGTN
        Created:     2026-04-25

        Version history:
        1.0.0 - (2026-04-25) Function created
    #>
    [CmdletBinding()]
    [OutputType([string])]
    param([Parameter(Mandatory)]$Type)

    $ResolvedType = $Type -as [type]
    if ($null -eq $ResolvedType) { throw "Failed to resolve type : $Type" }

    # Build accelerator table once per session instead of on every call
    if ($null -eq $Script:_PSAccelerators) {
        $Script:_PSAccelerators = [psobject].Assembly.GetType('System.Management.Automation.TypeAccelerators')::Get
    }

    # e.g : [bool] (System.Boolean) => bool
    if ($Alias = $Script:_PSAccelerators.GetEnumerator().Where({ $_.Value -eq $ResolvedType}, 'First')) {
        return $Alias[0].Key
    }

    # e.g : [int[]] (System.Int32[]) => int[]
    if ($ResolvedType.IsArray) {
        $ElementTypeName = Convert-PSTypeToFriendlyName -Type $ResolvedType.GetElementType()
        return "$ElementTypeName[]"
    }

    # e.g : [System.Collections.Generic.List[int]] (System.Collections.Generic.List`1[System.Int32])
    #          => System.Collections.Generic.List[int]
    if ($ResolvedType.IsGenericType) {
        $BaseName = $ResolvedType.Name -replace '`\d+$', ''
        $NameSpace = $ResolvedType.Namespace
        $GenericArgs = $ResolvedType.GetGenericArguments() | ForEach-Object { Convert-PSTypeToFriendlyName -Type $_ }
        return "$NameSpace.$BaseName[$($GenericArgs -join ',')]"
    }

    # Fallback
    return $ResolvedType.ToString()
}
```