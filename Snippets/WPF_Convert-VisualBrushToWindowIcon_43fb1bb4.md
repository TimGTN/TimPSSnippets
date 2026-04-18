<!--
id: 43fb1bb4
title: WPF Convert-VisualBrushToWindowIcon
language: powershell
tags: icon, wpf, function
description: Converts a WPF visual brush or a XAML string into a BitmapSource icon
created: 2026-04-18T12:37:23.123Z
updated: 2026-04-18T14:39:50
-->

# WPF Convert-VisualBrushToWindowIcon

> Converts a WPF visual brush or a XAML string into a BitmapSource icon

<p><code>icon</code> <code>wpf</code> <code>function</code></p>

```powershell
function Convert-VisualBrushToWindowIcon {
    <#
        .SYNOPSIS
            Converts a WPF visual brush or a XAML string into a BitmapSource icon.

        .DESCRIPTION
            Accepts either a VisualBrush object or a raw XAML string (parsed internally),
            renders it onto a WPF Canvas, encodes it as PNG into a MemoryStream,
            then converts it to a GDI Bitmap and finally to a WPF BitmapSource via HIcon.

        .PARAMETER Brush
            The VisualBrush to convert. Mutually exclusive with -Xaml.

        .PARAMETER Xaml
            A XAML string defining a VisualBrush. Parsed internally via XamlReader.
            Mutually exclusive with -Brush.

        .PARAMETER IconSize
            Size of the rendered icon in pixels. Must be between 1 and 32. Defaults to 32.

        .OUTPUTS
            System.Windows.Media.Imaging.BitmapSource

        .EXAMPLE
            $Icon = Convert-VisualBrushToWindowIcon -Brush $MyVisualBrush -IconSize 16
            Renders a VisualBrush object into a 16x16 BitmapSource icon.

        .EXAMPLE
            $Icon = Convert-VisualBrushToWindowIcon -Xaml $IconXaml
            Parses a XAML string and renders it into a 32x32 BitmapSource icon.

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2025-04-24

            Version history:
            1.0.0 - (2025-04-24) Function created
            1.1.0 - (2026-04-18) Added -Xaml
    #>
    param(
        [Parameter(Mandatory, ParameterSetName = 'FromBrush')]
        [ValidateNotNullOrEmpty()]
        $Brush,

        [Parameter(Mandatory, ParameterSetName = 'FromXaml')]
        [ValidateNotNullOrEmpty()]
        [string]$Xaml,

        [Parameter()]
        [ValidateRange(1, 32)]
        [int]$IconSize = 32
    )

    # Load required assemblies once — no-op if already loaded
    Add-Type -AssemblyName PresentationFramework, PresentationCore, System.Drawing

    # Import DestroyIcon to release the GDI handle allocated by GetHicon()
    if (-not ([System.Management.Automation.PSTypeName]'NativeMethods').Type) {
        Add-Type -TypeDefinition '
            using System;
            using System.Runtime.InteropServices;
            public class NativeMethods {
                [DllImport("user32.dll")]
                public static extern bool DestroyIcon(IntPtr hIcon);
            }
        '
    }

    try {
        # Parse XAML string into a VisualBrush if provided
        $ResolvedBrush = if ($PSCmdlet.ParameterSetName -eq 'FromXaml') {
            [System.Windows.Markup.XamlReader]::Parse($Xaml)
        } else { $Brush }

        # Render the brush onto a WPF Canvas
        $Canvas = [System.Windows.Controls.Canvas]::new()
        $Canvas.Width      = $IconSize
        $Canvas.Height     = $IconSize
        $Canvas.Background = $ResolvedBrush
        $Canvas.Measure([System.Windows.Size]::new($IconSize, $IconSize))
        $Canvas.Arrange([System.Windows.Rect]::new(0, 0, $IconSize, $IconSize))
        $Canvas.UpdateLayout()

        # Rasterize the canvas into a RenderTargetBitmap
        $RenderBitmap = [System.Windows.Media.Imaging.RenderTargetBitmap]::new(
            $IconSize, $IconSize, 96, 96,
            [System.Windows.Media.PixelFormats]::Pbgra32
        )
        $RenderBitmap.Render($Canvas)

        # Encode the bitmap to PNG in memory
        $Encoder = [System.Windows.Media.Imaging.PngBitmapEncoder]::new()
        $Encoder.Frames.Add([System.Windows.Media.Imaging.BitmapFrame]::Create($RenderBitmap))
        $MemoryStream = [System.IO.MemoryStream]::new()
        $Encoder.Save($MemoryStream)

        # Convert PNG stream to a GDI Bitmap, then to a WPF BitmapSource via HIcon
        $Bitmap = [System.Drawing.Bitmap]::new($MemoryStream)
        $MemoryStream.Dispose()

        $HIcon = $Bitmap.GetHicon()
        $Bitmap.Dispose()

        try {
            [System.Windows.Interop.Imaging]::CreateBitmapSourceFromHIcon(
                $HIcon,
                [System.Windows.Int32Rect]::Empty,
                [System.Windows.Media.Imaging.BitmapSizeOptions]::FromWidthAndHeight($IconSize, $IconSize)
            )
        }
        finally {
            # Release the GDI icon handle to prevent resource leak
            [void][NativeMethods]::DestroyIcon($HIcon)
        }
    
    } catch {
        $PSCmdlet.WriteError($_)
    }
}

# ===========
# TEST SAMPLE
# ===========
Add-Type -AssemblyName PresentationFramework

[xml]$Xaml = @'
<Window 
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    Width="300" Title="Test icon" SizeToContent="Height" Topmost="True">
    <Grid ><TextBlock Text="↑ Look at window icon" Margin="8"/></Grid>
</Window>
'@
$Interface = @{}
$Interface.Window = [Windows.Markup.XamlReader]::Load((New-Object -TypeName System.Xml.XmlNodeReader -ArgumentList $Xaml))
$Xaml.SelectNodes("//*[@*[contains(translate(name(.),'n','N'),'Name')]]") | ForEach-Object -Process {
    $Interface[$_.Name] = $Interface.Window.FindName($_.Name)
}

$IconXaml = @"
<VisualBrush xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
            xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml" Stretch="Uniform">
    <VisualBrush.Visual>
        <Canvas>
            <Path Fill="#26a6d1" Data="M117.547 266.156 0 249.141v-94.296h117.547z" />
            <Path Fill="#3db39e" Data="M291.346 136.51H136.31l.055-114.06L291.346.009z" />
            <Path Fill="#f4b459" Data="m291.346 291.337-155.091-22.459.182-114.015h154.909z" />
            <Path Fill="#e2574c" Data="M117.547 136.51H0V42.205l117.547-17.024z" />
        </Canvas>
    </VisualBrush.Visual>
</VisualBrush>
"@
$Interface.Window.Icon = Convert-VisualBrushToWindowIcon -Xaml $IconXaml

$Async = $Interface.Window.Dispatcher.InvokeAsync{
    $Interface.Window.ShowDialog() | Out-Null
}
$Async.Wait() | Out-Null

```