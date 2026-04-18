<!--
id: e3051c26
title: WPF New-WindowLocation
language: powershell
tags: wpf, window, function, position
description: Positions a WPF window relative to the available work area (taskbar-aware)
created: 2026-04-17T07:47:11.757Z
updated: 2026-04-18T11:37:35
-->

# WPF New-WindowLocation

> Positions a WPF window relative to the available work area (taskbar-aware)

<p><code>wpf</code> <code>window</code> <code>function</code> <code>position</code></p>

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

# ===========
# TEST SAMPLE
# ===========
Add-Type -AssemblyName PresentationFramework

[xml]$Xaml = @'
<Window 
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    Height="100" Width="200" Title="Reposition sample">
    <Grid ><Button Name="BTN_Test" Width="100" Height="30"/></Grid>
</Window>
'@
$Interface = @{}
$Interface.Window = [Windows.Markup.XamlReader]::Load((New-Object -TypeName System.Xml.XmlNodeReader -ArgumentList $Xaml))
$Xaml.SelectNodes("//*[@*[contains(translate(name(.),'n','N'),'Name')]]") | ForEach-Object -Process {
    $Interface[$_.Name] = $Interface.Window.FindName($_.Name)
}

$v = (Get-Command "Set-WindowPosition").Parameters.Position.Attributes.ValidValues
$pMap = @{} ; 0..($v.Count - 1) | ForEach-Object { $pMap[$v[$_]] = $v[($_ + 1) % $v.Count] }
$Interface.BTN_Test.Content = "GO " + @($pMap.Keys)[0]
$Interface.BTN_Test.Add_Click{
    $Position = $_.Source.Content.Split(' ')[1]
    Set-WindowPosition -Window $Interface.Window -Position $Position
    $_.Source.Content = "GO " + $pMap[$Position]
}

$Async = $Interface.Window.Dispatcher.InvokeAsync{
    $Interface.Window.ShowDialog() | Out-Null
}
$Async.Wait() | Out-Null

```