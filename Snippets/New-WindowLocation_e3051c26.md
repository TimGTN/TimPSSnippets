<!--
id: e3051c26
title: New-WindowLocation
language: powershell
tags: wpf, location, window
description: Positions a WPF window relative to the available work area (taskbar-aware)
created: 2026-04-17T07:47:11.757Z
updated: 2026-04-18T09:44:51
-->

# New-WindowLocation

> Positions a WPF window relative to the available work area (taskbar-aware)

<p><code>wpf</code> <code>location</code> <code>window</code></p>

```powershell
function Set-WindowPosition {
    <#
        .SYNOPSIS
            Positions a WPF window relative to the available work area (taskbar-aware).

        .PARAMETER Window
            The WPF window to position.

        .PARAMETER Position
            Corner or center of the work area where the window will be placed. Default: BottomRight.

        .PARAMETER Margin
            Pixel offset from the target edge. Default: 0.

        .EXAMPLE
            Set-WindowPosition -Window $MyWindow -Position BottomRight

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2026-04-18

            Version history:
            1.0.0 - (2026-04-18) Function created
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [System.Windows.Window]$Window,

        [ValidateSet('TopLeft', 'TopRight', 'BottomLeft', 'BottomRight', 'Center')]
        [string]$Position = 'BottomRight',

        [int]$Margin = 0
    )
    $WorkArea = [System.Windows.SystemParameters]::WorkArea

    [int]$Left = switch ($Position) {
        { $_ -in 'TopLeft',    'BottomLeft'  } { $WorkArea.Left  + $Margin }
        { $_ -in 'TopRight',   'BottomRight' } { $WorkArea.Right - $Window.ActualWidth  - $Margin }
        'Center'                               { $WorkArea.Left  + ($WorkArea.Width  - $Window.ActualWidth)  / 2 }
    }

    [int]$Top = switch ($Position) {
        { $_ -in 'TopLeft',    'TopRight'    } { $WorkArea.Top    + $Margin }
        { $_ -in 'BottomLeft', 'BottomRight' } { $WorkArea.Bottom - $Window.ActualHeight - $Margin }
        'Center'                               { $WorkArea.Top    + ($WorkArea.Height - $Window.ActualHeight) / 2 }
    }

    $Window.Left = $Left
    $Window.Top  = $Top
}

```