<!--
id: 5fea2063
title: Measure-CommandAverage
language: powershell
tags: function, optimize
description: Measures the average execution time of a scriptblock over multiple runs
created: 2026-04-18T12:09:03.980Z
updated: 2026-04-18T14:10:03
-->

# Measure-CommandAverage

> Measures the average execution time of a scriptblock over multiple runs

<p><code>function</code> <code>optimize</code></p>

```powershell
function Measure-CommandAverage {
    <#
        .SYNOPSIS
            Measures the average execution time of a scriptblock over multiple runs.

        .DESCRIPTION
            Executes the given scriptblock a specified number of times using a high-resolution
            Stopwatch, then returns the average elapsed time as a TimeSpan.
            Uses InvokeReturnAsIs() instead of Invoke() to minimize measurement overhead.

        .PARAMETER ScriptBlock
            The scriptblock to benchmark.

        .PARAMETER Count
            Number of iterations. Must be greater than 0. Defaults to 1.

        .EXAMPLE
            Measure-CommandAverage -ScriptBlock { Get-Date } -Count 1000
            Measure-CommandAverage -ScriptBlock { [datetime]::Now } -Count 1000

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2024-10-03

            Version history:
            1.0.0 - (2024-10-03) Function created
    #>
    param(
        [Parameter(Mandatory)]
        [scriptblock]$ScriptBlock,

        [Parameter()]
        [ValidateScript({ $_ -gt 0 })]
        [int]$Count = 1
    )

    [long]$TotalTicks = 0
    $Stopwatch = [System.Diagnostics.Stopwatch]::new()

    for ($i = 1; $i -le $Count; $i++) {
        $Stopwatch.Restart()

        # InvokeReturnAsIs avoids wrapping the result in a Collection<PSObject>,
        # reducing overhead and keeping the measurement as accurate as possible
        $null = $ScriptBlock.InvokeReturnAsIs()

        $Stopwatch.Stop()
        $TotalTicks += $Stopwatch.Elapsed.Ticks
    }

    # Return average elapsed time as a TimeSpan
    [timespan]::new([long]($TotalTicks / $Count))
}
```