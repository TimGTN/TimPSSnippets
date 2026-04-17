<!--
id: b6fcc735
title: Join-Collection
language: powershell
tags: concat, function, join
description: Concatenates a collection of objects into a delimited, quoted string
created: 2026-04-17T21:56:32.642Z
updated: 2026-04-17T23:57:06
-->

# Join-Collection

> Concatenates a collection of objects into a delimited, quoted string

<p><code>concat</code> <code>function</code> <code>join</code></p>

```powershell
function Join-Collection {
    <#
        .SYNOPSIS
            Concatenates a collection of objects into a delimited, quoted string.

        .PARAMETER InputObject
            Collection of objects or hashtable to join. Accepts pipeline input.
            For hashtables, values are used. For objects, use -Property to specify which member to extract.

        .PARAMETER Property
            Name of the property to extract from each object. Ignored for hashtables and scalar values.

        .PARAMETER Delimiter
            Separator between items. Accepts ',' or ';'. Default: ','.

        .PARAMETER Quote
            Quote character wrapping each item. Accepts ' or ". Default: ".

        .EXAMPLE
            $Users | Join-Collection -Property SamAccountName -Delimiter ';' -Quote '"'

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
        $InputObject,

        [string]$Property,

        [ValidateSet(',', ';')]
        [string]$Delimiter = ',',

        [ValidateSet("'", '"')]
        [string]$Quote = '"'
    )

    begin {
        $Items = [System.Collections.Generic.List[string]]::new()
        $PropertyValidated = $false
    }

    process {
        if ($InputObject -is [System.Collections.IDictionary]) {
            foreach ($Value in $InputObject.Values) {
                $Items.Add($Quote + $Value.ToString() + $Quote)
            }
        } elseif ($Property) {
            if (-not $PropertyValidated) {
                if (-not ($InputObject.PSObject.Properties[$Property])) {
                    $PSCmdlet.ThrowTerminatingError(
                        [System.Management.Automation.ErrorRecord]::new(
                            [System.ArgumentException]::new("Property '$Property' does not exist on object of type '$($InputObject.GetType().Name)'."),
                            'PropertyNotFound',
                            [System.Management.Automation.ErrorCategory]::InvalidArgument,
                            $InputObject
                        )
                    )
                }
                $PropertyValidated = $true
            }
            $Items.Add($Quote + $InputObject.$Property.ToString() + $Quote)
        } else {
            $Items.Add($Quote + $InputObject.ToString() + $Quote)
        }
    }

    end {
        return [string]::Join($Delimiter, $Items)
    }
}
```