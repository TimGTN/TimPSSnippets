<!--
id: dd1658e5
title: Remove-XmlNamespaces
language: powershell
tags: function, xml, namespace
description: Removes all namespace declarations and prefixes from an XML string
created: 2026-04-17T08:09:41.284Z
updated: 2026-04-17T10:12:34
-->

# Remove-XmlNamespaces

> Removes all namespace declarations and prefixes from an XML string

<p><code>function</code> <code>xml</code> <code>namespace</code></p>

```powershell
function Remove-XmlNamespaces {
<#
    .SYNOPSIS
        Removes all namespace declarations and prefixes from an XML string.

    .DESCRIPTION
        Parses the input as an XElement (System.Xml.Linq), then strips namespace
        declarations and prefixes from every element and attribute in the tree.
        Returns the cleaned XML as a string, optionally without indentation.

    .PARAMETER XmlString
        The XML string to process. Accepts pipeline input.

    .PARAMETER NoFormatting
        When specified, returns the XML on a single line without indentation.

    .OUTPUTS
        System.String — The namespace-free XML string.

    .EXAMPLE
        Remove-XmlNamespaces -XmlString $rawXml

    .EXAMPLE
        $rawXml | Remove-XmlNamespaces -NoFormatting

    .NOTES
        Author:      Tim GILLOTIN
        Contact:     @TimGTN
        Created:     2024-09-29

        Version history:
        1.0.0 - (2024-09-29) Function created
        2.0.0 - (2026-04-17) Direct attribute removal, 
                             pipeline support
#>
    [OutputType([string])]
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [ValidateNotNullOrEmpty()]
        [string]$XmlString,

        [switch]$NoFormatting
    )

    begin {
        if (-not ('System.Xml.Linq.XElement' -as [type])) {
            Add-Type -AssemblyName "System.Xml.Linq"
        }
    }

    process {
        try {
            $XElement = [System.Xml.Linq.XElement]::Parse($XmlString)

            $XElement.DescendantNodesAndSelf() | ForEach-Object {
                if ($_.NodeType -ne 'Element') { return }

                # Replace the qualified name with its local name only
                $_.Name = [System.Xml.Linq.XName]::Get($_.Name.LocalName)

                # Remove namespace declarations and namespace-prefixed attributes
                $_.Attributes() |
                    Where-Object { $_.IsNamespaceDeclaration -or $_.Name.Namespace -ne [System.Xml.Linq.XNamespace]::None } |
                    ForEach-Object { $_.Remove() }  # $_ is already the XAttribute — no re-fetch needed
            }

            $SaveOptions = if ($NoFormatting) {
                [System.Xml.Linq.SaveOptions]::DisableFormatting
            } else {
                [System.Xml.Linq.SaveOptions]::None
            }

            return $XElement.ToString($SaveOptions)
        }
        catch {
            $PSCmdlet.WriteError($_)
        }
    }
}
```