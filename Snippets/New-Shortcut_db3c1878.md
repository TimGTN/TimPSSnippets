<!--
id: db3c1878
title: New-Shortcut
language: powershell
tags: shortcut, function
description: Creates a Windows shortcut (.lnk) file with optional icon, arguments, working directory and window style
created: 2026-04-17T12:04:51.189Z
updated: 2026-04-17T14:05:36
-->

# New-Shortcut

> Creates a Windows shortcut (.lnk) file with optional icon, arguments, working directory and window style

<p><code>shortcut</code> <code>function</code></p>

```powershell
function New-Shortcut {
<#
    .SYNOPSIS
        Creates a Windows shortcut (.lnk) file with optional icon, arguments, working directory and window style.
   
    .DESCRIPTION
        Creates a Windows shortcut file at the specified path using the WScript.Shell COM object.
        The destination directory is created automatically if it does not exist.
        TargetPath is resolved automatically via WorkingDirectory merge or PATH lookup unless -SkipValidation is specified.

   
    .PARAMETER Path
        The directory where the shortcut file will be created. Created automatically if it does not exist.
  
    .PARAMETER Name
        The name of the shortcut file. The .lnk extension is added automatically if omitted.

    .PARAMETER TargetPath
        The full or relative path to the target executable or file.
        If WorkingDirectory is provided, it will be merged with TargetPath to resolve the full path.
        If WorkingDirectory is omitted, resolution is attempted via Get-Command (PATH), then as an absolute path.

    .PARAMETER ArgumentList
        One or more arguments to pass to the target executable. Multiple values are joined with a space.

    .PARAMETER IcoFilePath
        The full path to the .ico file to use as the shortcut icon.

    .PARAMETER WorkingDirectory
        The working directory for the target process. Must exist if specified.
        If provided alongside TargetPath, both are merged to resolve and validate the target.

    .PARAMETER WindowStyle
        The window style for the target process.
        1 = Normal, 3 = Maximized, 7 = Minimized. Defaults to 1.

    .PARAMETER SkipValidation
        Skips TargetPath resolution and validation. Useful when the target may not exist at packaging time.

    .EXAMPLE
        New-Shortcut -Path "C:\temp" -Name "myShortcut" -TargetPath "powershell.exe" `
            -ArgumentList "-NoProfile", "-File `"C:\script.ps1`""

    .EXAMPLE
        New-Shortcut -Path "C:\temp" -Name "myShortcut" -TargetPath "MyApp.exe" `
            -WorkingDirectory "C:\temp\MyApp" -IcoFilePath "C:\temp\MyApp\icon.ico"

    .OUTPUTS
        System.String. Returns the full path to the created shortcut file.

    .NOTES
        Author:      Tim GILLOTIN
        Contact:     @TimGTN
        Created:     2026-04-17

        Version history:
        1.0.0 - (2026-04-17) Function created
#>
    [CmdletBinding()]
    [OutputType([string])]
    param (
        [Parameter(Mandatory)]
        [string]$Path,

        [Parameter(Mandatory)]
        [string]$Name,

        [Parameter(Mandatory)]
        [string]$TargetPath,

        [string[]]$ArgumentList = @(),

        [ValidateScript({ Test-Path $_ -PathType Leaf })]
        [string]$IcoFilePath,

        [ValidateScript({ -not $_ -or (Test-Path $_ -PathType Container) })]
        [string]$WorkingDirectory,

        [ValidateSet(1, 3, 7)]
        [int]$WindowStyle = 1,

        [switch]$SkipValidation
    )

    try {
        # Resolve and validate TargetPath
        if (-not $SkipValidation) {
            if ($WorkingDirectory) {
                $resolvedTarget = Join-Path -Path $WorkingDirectory -ChildPath $TargetPath
                if (-not (Test-Path $resolvedTarget -PathType Leaf)) {
                    Write-Error "Target '$resolvedTarget' does not exist. Use -SkipValidation to bypass."
                    return
                }
            }
            else {
                $cmd            = Get-Command $TargetPath -ErrorAction SilentlyContinue
                $resolvedTarget = if ($cmd) { $cmd.Source } else { $TargetPath }
                if (-not (Test-Path $resolvedTarget -PathType Leaf)) {
                    Write-Error "Target '$resolvedTarget' could not be resolved. Use -SkipValidation to bypass."
                    return
                }
            }
            $TargetPath = $resolvedTarget
        }

        # Ensure the destination directory exists
        if (-not (Test-Path -Path $Path -PathType Container)) {
            New-Item -Path $Path -ItemType Directory -Force | Out-Null
            Write-Verbose "Created directory '$Path'"
        }

        # Ensure the .lnk extension is present
        if (-not $Name.EndsWith(".lnk", [System.StringComparison]::OrdinalIgnoreCase)) {
            $Name = "$Name.lnk"
        }

        $lnkPath = Join-Path -Path $Path -ChildPath $Name

        $shell            = New-Object -ComObject WScript.Shell
        $link             = $shell.CreateShortcut($lnkPath)
        $link.TargetPath  = $TargetPath
        $link.WindowStyle = $WindowStyle
        if ($ArgumentList.Count -gt 0) { $link.Arguments = $ArgumentList -join " " }
        if ($WorkingDirectory) { $link.WorkingDirectory = $WorkingDirectory }
        if ($IcoFilePath) { $link.IconLocation = $IcoFilePath }

        $link.Save()

        Write-Verbose "Shortcut created at '$lnkPath'"
        return $lnkPath
    
    } catch {
        Write-Error "Failed to create shortcut '$Name' at '$Path'. $_"
    }
}
```