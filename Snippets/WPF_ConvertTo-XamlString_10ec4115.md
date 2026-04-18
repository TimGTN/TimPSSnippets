<!--
id: 10ec4115
title: WPF ConvertTo-XamlString
language: powershell
tags: function, wpf, xaml, convert
description: Serializes a WPF UIElement (or any XAML-compatible object) to an XAML string
created: 2026-04-17T21:32:43.347Z
updated: 2026-04-18T11:44:57
-->

# WPF ConvertTo-XamlString

> Serializes a WPF UIElement (or any XAML-compatible object) to an XAML string

<p><code>function</code> <code>wpf</code> <code>xaml</code> <code>convert</code></p>

```powershell
function ConvertTo-XamlString {
    <#
        .SYNOPSIS
            Serializes a WPF UIElement (or any XAML-compatible object) to an XAML string.

        .PARAMETER InputObject
            The WPF / XAML object to serialize. Accepts pipeline input.

        .PARAMETER Indent
            Indents the output XML for readability.

        .PARAMETER RemoveNamespaces
            Strips all XML namespace declarations and prefixed attributes from the output.

        .EXAMPLE
            $MyButton | ConvertTo-XamlString -Indent -RemoveNamespaces

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2026-04-17

            Version history:
            1.0.0 - (2026-04-17) Function created
    #>
    [OutputType([string])]
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [ValidateNotNull()]
        [object]$InputObject,

        [switch]$Indent,

        [ValidateRange(1, 8)]
        [int]$IndentSize = 4,

        [switch]$RemoveNamespaces
    )

    begin {
        foreach ($Assembly in 'PresentationFramework', 'System.Xml.Linq') {
            if (-not ($Assembly -as [type])) {
                Add-Type -AssemblyName $Assembly
            }
        }
    }

    process {
        try {
            # Indented without namespace stripping requires XmlWriter (exposes XmlWriterSettings)
            # All other cases go through StringWriter ; XElement handles formatting later if needed
            [string]$RawXaml = if ($Indent -and -not $RemoveNamespaces) {

                $Stream         = [System.IO.MemoryStream]::new()
                $WriterSettings = [System.Xml.XmlWriterSettings]@{
                    Indent             = $true
                    IndentChars        = ' ' * $IndentSize
                    ConformanceLevel   = [System.Xml.ConformanceLevel]::Fragment
                    OmitXmlDeclaration = $true
                    Encoding           = [System.Text.Encoding]::UTF8
                }

                $Writer = [System.Xml.XmlWriter]::Create($Stream, $WriterSettings)
                try {
                    [System.Windows.Markup.XamlWriter]::Save($InputObject, $Writer)
                    $Writer.Flush()
                }
                finally { $Writer.Dispose() }

                $Stream.Position = 0
                $Reader = [System.IO.StreamReader]::new($Stream, [System.Text.Encoding]::UTF8)
                try   { $Reader.ReadToEnd() }
                finally {
                    $Reader.Dispose()
                    $Stream.Dispose()
                }

            } else {

                $StringWriter = [System.IO.StringWriter]::new()
                try   { [System.Windows.Markup.XamlWriter]::Save($InputObject, $StringWriter) ; $StringWriter.ToString() }
                finally { $StringWriter.Dispose() }
            }

            if ($RemoveNamespaces) {
                $XElement = [System.Xml.Linq.XElement]::Parse($RawXaml)

                $XElement.DescendantNodesAndSelf() | ForEach-Object {
                    if ($_.NodeType -ne 'Element') { return }

                    $_.Name = [System.Xml.Linq.XName]::Get($_.Name.LocalName)

                    $_.Attributes() |
                        Where-Object { $_.IsNamespaceDeclaration -or
                                       $_.Name.Namespace -ne [System.Xml.Linq.XNamespace]::None } |
                        ForEach-Object { $_.Remove() }
                }

                if ($Indent) {
                    $StringBuilder  = [System.Text.StringBuilder]::new()
                    $WriterSettings = [System.Xml.XmlWriterSettings]@{
                        Indent             = $true
                        IndentChars        = ' ' * $IndentSize
                        ConformanceLevel   = [System.Xml.ConformanceLevel]::Fragment
                        OmitXmlDeclaration = $true
                    }
                    $Writer = [System.Xml.XmlWriter]::Create($StringBuilder, $WriterSettings)
                    try   { $XElement.WriteTo($Writer) ; $Writer.Flush() }
                    finally { $Writer.Dispose() }

                    return ($StringBuilder.ToString() -replace '\s+xmlns(?::\w+)?="[^"]*"', '')
                }

                return ($XElement.ToString([System.Xml.Linq.SaveOptions]::DisableFormatting) -replace '\s+xmlns(?::\w+)?="[^"]*"', '')
            }

            return $RawXaml
        
        } catch {
            $PSCmdlet.WriteError($_)
        }
    }
}
```