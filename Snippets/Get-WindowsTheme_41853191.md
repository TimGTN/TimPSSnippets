<!--
id: 41853191
title: Get-WindowsTheme
language: powershell
tags: ui, color, function
description: Returns the current Windows system theme and accent color
created: 2026-04-18T07:15:52.734Z
updated: 2026-04-18T09:16:41
-->

# Get-WindowsTheme

> Returns the current Windows system theme and accent color

<p><code>ui</code> <code>color</code> <code>function</code></p>

```powershell
function Get-WindowsTheme {
    <#
        .SYNOPSIS
            Returns the current Windows system theme and accent color.

        .EXAMPLE
            Get-WindowsTheme

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2026-04-18

            Version history:
            1.0.0 - (2026-04-18) Function created
    #>
    [OutputType([PSCustomObject])]
    [CmdletBinding()]
    param()

    $PersonalizeKey = Get-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize' -ErrorAction SilentlyContinue
    $DwmKey         = Get-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\DWM' -ErrorAction SilentlyContinue

    # AccentColor is stored as ABGR — reorder to RGB
    $Abgr        = $DwmKey.AccentColor
    $AccentColor = '#{0:X2}{1:X2}{2:X2}' -f ($Abgr -band 0xFF), (($Abgr -shr 8) -band 0xFF), (($Abgr -shr 16) -band 0xFF)

    [PSCustomObject]@{
        Theme       = if ($PersonalizeKey.SystemUsesLightTheme -eq 1) { 'Light' } else { 'Dark' }
        AccentColor = $AccentColor
    }
}
```