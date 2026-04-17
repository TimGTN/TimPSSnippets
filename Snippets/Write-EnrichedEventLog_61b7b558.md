<!--
id: 61b7b558
title: Write-EnrichedEventLog
language: powershell
tags: function, eventlog
description: Writes a structured, JSON-enriched entry to the Windows Event Log
created: 2026-04-17T05:56:57.809Z
updated: 2026-04-17T08:24:29
-->

# Write-EnrichedEventLog

> Writes a structured, JSON-enriched entry to the Windows Event Log

<p><code>function</code> <code>eventlog</code></p>

```powershell
function Write-EnrichedEventLog {
    <#
        .SYNOPSIS
            Writes a structured, JSON-enriched entry to the Windows Event Log.

        .DESCRIPTION
            Ensures the specified event source exists in the target log (creating it if needed),
            then writes a structured event entry whose message is a JSON payload.

            The payload wraps the provided data under a "Data" key alongside a "Component" field,
            making entries consistent, machine-parseable, and easy to correlate across scripts.

            If the source already exists but is registered under a different log, an error is raised
            to prevent silent misrouting of events.

        .PARAMETER EventID
            The event identifier. Must be a positive integer meaningful to the application.

        .PARAMETER Component
            Name of the script, module, or process writing the event. Used as a correlation key
            when multiple components share the same source.

        .PARAMETER EntryType
            Severity of the event: Information, Warning, Error, SuccessAudit, or FailureAudit.

        .PARAMETER LogName
            Target event log. Defaults to "Application".

        .PARAMETER Source
            Event source name. Created automatically if absent. Raises an error if it exists
            under a different log than LogName.

        .PARAMETER MessageData
            Arbitrary object serialized as the JSON body of the event under the "Data" key.
            Accepts any depth of nested objects or arrays.

        .PARAMETER Category
            Arbitrary value to simplify filtering events.

        .EXAMPLE
            Write-EnrichedEventLog `
                -EventID     1000 `
                -Component   "Provisioning.ps1" `
                -EntryType   Information `
                -Source      "CustomSource" `
                -MessageData @{ Step = "Init"; User = $env:USERNAME }

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2026-04-09

            Version history:
            1.0.0 - (2026-04-09) Function created
    #>
    [CmdletBinding()]
    param (
        [Parameter(Mandatory)]
        [int]$EventID,

        [Parameter(Mandatory)]
        [string]$Component,

        [ValidateNotNullOrEmpty()]
        [System.Diagnostics.EventLogEntryType]$EntryType = "Information",

        [ValidateNotNullOrEmpty()]
        [string]$LogName = "Application",

        [Parameter(Mandatory)]
        [string]$Source,

        [Parameter(Mandatory)]
        [object]$MessageData,
    
        [ValidateNotNullOrEmpty()]
        [int]$Category
    )
    try {
        # Ensure source exists and is registered to the expected log
        if ([System.Diagnostics.EventLog]::SourceExists($Source)) {
            $registeredLog = [System.Diagnostics.EventLog]::LogNameFromSourceName($Source, ".")
            if ($registeredLog -ne $LogName) {
                throw "Source '$Source' is registered under '$registeredLog', not '$LogName'."
            }
        } else {
            New-EventLog -LogName $LogName -Source $Source -ErrorAction Stop
        }

        # Build structured payload
        $Payload = [ordered]@{
            Component = $Component
            Data      = $MessageData
        }
        $Json = ($Payload | ConvertTo-Json -Depth 10 -Compress).Replace('\u0027',"'")

        # Write event
        $Param = @{
            LogName = $LogName
            Source = $Source
            EventID = $EventID
            EntryType = $EntryType
            Message = $Json
        }
        if ($PSBoundParameters.ContainsKey('Category')) { $Param.Category = $Category }
        Write-EventLog @Param
    
    } catch {
        $PSCmdlet.WriteError($_)
    }
}
```