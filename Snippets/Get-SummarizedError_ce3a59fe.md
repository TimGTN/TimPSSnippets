<!--
id: ce3a59fe
title: Get-SummarizedError
language: powershell
tags: function, error, exception
description: Extracts and summarizes key information from a PowerShell ErrorRecord
created: 2026-04-18T11:00:54.864Z
updated: 2026-04-18T13:02:49
-->

# Get-SummarizedError

> Extracts and summarizes key information from a PowerShell ErrorRecord

<p><code>function</code> <code>error</code> <code>exception</code></p>

```powershell
function Get-SummarizedError {
    <#
        .SYNOPSIS
            Extracts and summarizes key information from a PowerShell ErrorRecord.
            
        .DESCRIPTION
            Takes an ErrorRecord and returns a clean PSCustomObject containing the exception
            message and type, the innermost exception (often the true root cause), the
            invocation context (command, line, line number), and the first frame of the
            script stack trace.
            Supports pipeline input and multiple ErrorRecords.

        .PARAMETER ErrorRecord
            The ErrorRecord to summarize. Accepts pipeline input.
            
        .EXAMPLE
            $Error[0] | Get-SummarizedError

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2024-10-03

            Version history:
            1.0.0 - (2024-10-03) Function created
    #>
    param(
        [Parameter(ValueFromPipeline)]
        [ValidateNotNullOrEmpty()]
        [System.Management.Automation.ErrorRecord]$ErrorRecord
    )
    process {
        # Walk down the inner exception chain to find the root cause
        $Inner = $ErrorRecord.Exception.InnerException
        while ($Inner.InnerException) {
            $Inner = $Inner.InnerException
        }

        [PsCustomObject]@{
            # Top-level exception
            Message      = $ErrorRecord.Exception.Message
            Type         = $ErrorRecord.Exception.GetType().Name
            # Innermost exception — often the true cause
            InnerMessage = if ($Inner) { $Inner.Message } else { $null }
            InnerType    = if ($Inner) { $Inner.GetType().Name } else { $null }
            # Invocation context
            Command      = $ErrorRecord.InvocationInfo.MyCommand.Name
            Line         = $ErrorRecord.InvocationInfo.Line.Trim()
            LineNumber   = $ErrorRecord.InvocationInfo.ScriptLineNumber
            # First frame of the script stack trace
            StackTrace   = ($ErrorRecord.ScriptStackTrace -split "`n")[0].Trim()
        }
    }
}
```