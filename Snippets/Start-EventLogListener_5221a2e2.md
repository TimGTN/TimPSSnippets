<!--
id: 5221a2e2
title: Start-EventLogListener
language: powershell
tags: function, eventlog
description: Registers a real-time, push-based listener on a Windows Event Log source
created: 2026-04-17T06:26:45.549Z
updated: 2026-04-17T08:39:37
-->

# Start-EventLogListener

> Registers a real-time, push-based listener on a Windows Event Log source

<p><code>function</code> <code>eventlog</code></p>

```powershell
function Start-EventLogListener {
<#
    .SYNOPSIS
        Registers a real-time, push-based listener on a Windows Event Log source.

    .DESCRIPTION
        Subscribes to the EntryWritten .NET delegate on a given EventLog source,
        avoiding any polling. Only events written after the listener starts are processed.

        Two execution paths are available depending on the calling context:

        - Default (no -UseDispatcher): a standard Register-ObjectEvent -Action block
            is used. The callback runs in an isolated PS job runspace, driven by the
            PowerShell event pump. Suitable for console scripts or while/Wait-Event loops.

        - UI (-UseDispatcher): no -Action block is registered. Instead, a
            DispatcherTimer drains the PowerShell event queue on the WPF UI thread via
            Get-Event, allowing OnMessage to safely access WPF controls. Required when
            ShowDialog() or Application.Run() blocks the PowerShell event pump.
            Must be called from the UI thread before ShowDialog().

        Filtering is applied in order, from cheapest to most expensive:
            1. Source name          → ignores noise from other applications
            2. Timestamp            → ignores historical entries present at startup
            3. Category exclusion   → drops entries whose Category matches ExcludeCategories
            4. Message inclusion    → drops entries whose message matches none of IncludeMessagePattern
            5. Message exclusion    → drops entries whose message matches any of ExcludeMessagePattern

        The OnMessage callback receives the raw System.Diagnostics.EventLogEntry.
        All further parsing (JSON deserialization, field extraction, etc.) is the caller's responsibility.

    .PARAMETER Source
        Event Log source name to listen on.

    .PARAMETER LogName
        Target event log. Defaults to "Application".

    .PARAMETER ExcludeCategories
        Optional list of Category values (Int16) to silently drop.
        Evaluated before reading the message body.

    .PARAMETER IncludeMessagePattern
        Optional list of regex patterns. An entry is kept only if its message
        matches at least one of these patterns.
        Useful to restrict processing to structured entries (e.g. "^\s*\{" for JSON).

    .PARAMETER ExcludeMessagePattern
        Optional list of regex patterns. An entry is dropped if its message
        matches any of these patterns.

    .PARAMETER OnMessage
        ScriptBlock invoked for each entry that passes all filters.
        Signature: param([System.Diagnostics.EventLogEntry] $Entry)

    .PARAMETER UseDispatcher
        When specified, bypasses the PowerShell event pump and uses a DispatcherTimer
        instead. Required when the calling thread is blocked by a Win32 message loop
        (e.g. ShowDialog(), Application.Run()) that prevents -Action blocks from firing.
        The DispatcherTimer is bound to the current thread's dispatcher at call time —
        this switch must therefore be used before ShowDialog() is called, from the UI thread.
        OnMessage runs on the WPF UI thread and has direct, thread-safe access to controls.

    .PARAMETER DispatcherTimerInterval
        Polling interval in milliseconds for the DispatcherTimer. Defaults to 100 ms.
        Only relevant when -UseDispatcher is specified.

    .EXAMPLE
        # Console path — filter to JSON entries, ignore own writes
        $listener = Start-EventLogListener `
            -Source                "CustomSource" `
            -IncludeMessagePattern "^\s*\{" `
            -ExcludeMessagePattern "`"Component`"\s*:\s*`"$([regex]::Escape($Component))`"" `
            -OnMessage {
                param($Entry)
                $payload = $Entry.Message | ConvertFrom-Json
                Write-Host "[$($payload.Component)] $($payload.Data._bus.verb)"
            }

        $listener.Stop()

    .EXAMPLE
        # UI path — OnMessage can interact with WPF controls
        $listener = Start-EventLogListener `
            -Source                "CustomSource" `
            -ExcludeCategories     2 `
            -IncludeMessagePattern "^\s*\{" `
            -UseDispatcher `
            -OnMessage {
                param($Entry)
                $Interface.Border.Background = "Red"
            }

        $Interface.ShowDialog() # Blocks — DispatcherTimer keeps firing during ShowDialog()
        $listener.Stop()

    .OUTPUTS
        PSCustomObject with properties : Log, SourceIdentifier, DispatcherTimer, Source, LogName
        PSCustomObject with method     : Stop() — unregisters the subscription, stops the timer if any, and disposes the EventLog instance

    .NOTES
        Author:      Tim GILLOTIN
        Contact:     @TimGTN
        Created:     2026-04-09

        Version history:
        1.0.0 - (2026-04-09) Function created
#>
    [CmdletBinding()]
    [OutputType([PSCustomObject])]
    param (
        [Parameter(Mandatory)]
        [string]$Source,

        [ValidateNotNullOrEmpty()]
        [string]$LogName = "Application",

        [int[]]$ExcludeCategories,

        [string[]]$IncludeMessagePattern,

        [string[]]$ExcludeMessagePattern,

        [Parameter(Mandatory)]
        [scriptblock]$OnMessage,

        [switch]$UseDispatcher,

        [Parameter(DontShow)]
        [int]$DispatcherTimerInterval = 100
    )

    $Log              = [System.Diagnostics.EventLog]::new($LogName)
    $StartTime        = [datetime]::Now  # Captured before EnableRaisingEvents — no historical event will pass the timestamp filter
    $SourceIdentifier = "EventBusListener_$([guid]::NewGuid().Guid)"
    $Log.EnableRaisingEvents = $true

    $MessageData = @{
        Source                = $Source
        StartTime             = $StartTime
        ExcludeCategories     = $ExcludeCategories
        IncludeMessagePattern = $IncludeMessagePattern
        ExcludeMessagePattern = $ExcludeMessagePattern
    }

    # Shared filter logic — runs in both paths to keep behavior consistent
    $FilterAndInvoke = {
        param($Entry, $MD, $Callback)

        # Ordered from cheapest to most expensive check
        if ($Entry.Source -ne $MD.Source)                                                       { return $false }
        if ($Entry.TimeGenerated -lt $MD.StartTime)                                             { return $false }
        if ($MD.ExcludeCategories -and ([int]$Entry.CategoryNumber) -in $MD.ExcludeCategories) { return $false }

        if ($MD.IncludeMessagePattern) {
            $included = $false
            foreach ($p in $MD.IncludeMessagePattern) { if ($Entry.Message -match $p) { $included = $true; break } }
            if (-not $included) { return $false }
        }

        if ($MD.ExcludeMessagePattern) {
            foreach ($p in $MD.ExcludeMessagePattern) { if ($Entry.Message -match $p) { return $false } }
        }

        & $Callback $Entry
        return $true
    }

    $DispatcherTimer = $null

    if ($UseDispatcher) {
        # UI path — no -Action block: events queue natively in the PS event queue
        # and are drained by the DispatcherTimer on the UI thread.
        # New-Object binds the timer to the current thread's dispatcher automatically —
        # this must be called before ShowDialog() from the UI thread.
        Register-ObjectEvent `
            -InputObject      $Log `
            -EventName        "EntryWritten" `
            -SourceIdentifier $SourceIdentifier | Out-Null

        $DispatcherTimer = New-Object System.Windows.Threading.DispatcherTimer
        $DispatcherTimer.Interval = [TimeSpan]::FromMilliseconds($DispatcherTimerInterval)
        $DispatcherTimer.Tag = @{
            SourceIdentifier = $SourceIdentifier
            MessageData      = $MessageData
            OnMessage        = $OnMessage
            FilterAndInvoke  = $FilterAndInvoke
        }
        $DispatcherTimer.Add_Tick({
            foreach ($PSEvent in (Get-Event -SourceIdentifier $this.Tag.SourceIdentifier -ErrorAction SilentlyContinue)) {
                Remove-Event -EventIdentifier $PSEvent.EventIdentifier  # Prevent unbounded queue growth
                & $this.Tag.FilterAndInvoke $PSEvent.SourceEventArgs.Entry $this.Tag.MessageData $this.Tag.OnMessage
            }
        })
        $DispatcherTimer.Start()

    } else {
        # Console path — standard -Action block driven by the PowerShell event pump
        $MessageData.OnMessage      = $OnMessage
        $MessageData.FilterAndInvoke = $FilterAndInvoke

        Register-ObjectEvent `
            -InputObject      $Log `
            -EventName        "EntryWritten" `
            -SourceIdentifier $SourceIdentifier `
            -MessageData      $MessageData `
            -Action {
                & $Event.MessageData.FilterAndInvoke `
                    $Event.SourceEventArgs.Entry `
                    $Event.MessageData `
                    $Event.MessageData.OnMessage
            } | Out-Null
    }

    $Listener = [PSCustomObject]@{
        Log              = $Log
        SourceIdentifier = $SourceIdentifier
        DispatcherTimer  = $DispatcherTimer
        Source           = $Source
        LogName          = $LogName
    }

    $Listener | Add-Member -MemberType ScriptMethod -Name Stop -Value {
        Unregister-Event -SourceIdentifier $this.SourceIdentifier -ErrorAction SilentlyContinue
        Get-Job | Where-Object Name -eq $this.SourceIdentifier | Remove-Job -ErrorAction SilentlyContinue
        if ($this.DispatcherTimer) { $this.DispatcherTimer.Stop() }
        $this.Log.EnableRaisingEvents = $false
        $this.Log.Dispose()
    }

    return $Listener
}
```