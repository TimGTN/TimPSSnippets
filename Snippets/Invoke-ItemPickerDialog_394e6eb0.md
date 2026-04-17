<!--
id: 394e6eb0
title: Invoke-ItemPickerDialog
language: powershell
tags: function, dialog, file, folder
description: Displays a modern Windows file or folder picker dialog
created: 2026-04-17T20:23:39.135Z
updated: 2026-04-17T22:26:43
-->

# Invoke-ItemPickerDialog

> Displays a modern Windows file or folder picker dialog

<p><code>function</code> <code>dialog</code> <code>file</code> <code>folder</code></p>

```powershell
function Invoke-ItemPickerDialog {
    <#
        .SYNOPSIS
            Displays a modern Windows file or folder picker dialog.

        .DESCRIPTION
            Wraps the Vista-style IFileDialog via reflection to provide a single picker
            for both files and folders. Use -Folder to switch to folder selection mode.

        .PARAMETER Title
            Text displayed in the dialog title bar.

        .PARAMETER InitialDirectory
            Starting directory when the dialog opens. Defaults to the user's desktop.

        .PARAMETER Filter
            File type filter string in WinForms format (e.g. "Executables (*.exe)|*.exe|All files (*.*)|*.*").
            Ignored when -Folder is specified.

        .PARAMETER Folder
            Opens the dialog in folder selection mode instead of file selection mode.

        .PARAMETER MultiSelect
            Allows selecting multiple items. Returns an array of paths.

        .EXAMPLE
            Invoke-ItemPickerDialog -Title "Select a folder" -Folder

        .EXAMPLE
            Invoke-ItemPickerDialog -Title "Select setup file" -InitialDirectory "C:\Apps" `
                -Filter "Setup files (*.exe;*.msi)|*.exe;*.msi|All files (*.*)|*.*"

        .OUTPUTS
            System.String[]
            Returns an array of selected paths, or $null if the dialog was cancelled.

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2026-04-17

            Version history:
            1.0.0 - (2026-04-17) Function created
    #>
    [CmdletBinding()]
    param(
        [Parameter()]
        [string]$Title = "Select",

        [string]$InitialDirectory = [Environment]::GetFolderPath("Desktop"),

        [string]$Filter = "All files (*.*)|*.*",

        [switch]$Folder,

        [switch]$MultiSelect
    )

    try {
        # Load WinForms — required for IFileDialog reflection
        try {
            Add-Type -AssemblyName System.Windows.Forms -ErrorAction Stop
        } catch {
            throw [System.PlatformNotSupportedException]::new(
                "Failed to load System.Windows.Forms. This function requires a Windows environment with .NET WinForms support.")
        }

        # Validate InitialDirectory
        if (-not (Test-Path -Path $InitialDirectory -PathType Container)) {
            throw [System.IO.DirectoryNotFoundException]::new(
                "Initial directory not found: '$InitialDirectory'")
        }

        # Reflection binding flags to access internal members
        [Reflection.BindingFlags]$BindingFlags = [Reflection.BindingFlags]::Instance -bor
                                                 [Reflection.BindingFlags]::Public -bor
                                                 [Reflection.BindingFlags]::NonPublic

        $FileDialogAssembly = [System.Windows.Forms.FileDialog].Assembly

        # Resolve internal types
        $IFileDialogType        = $FileDialogAssembly.GetType("System.Windows.Forms.FileDialogNative+IFileDialog")
        $FOSType                = $FileDialogAssembly.GetType("System.Windows.Forms.FileDialogNative+FOS")
        $DialogEventsType       = $FileDialogAssembly.GetType("System.Windows.Forms.FileDialog+VistaDialogEvents")

        if (-not $IFileDialogType -or -not $FOSType -or -not $DialogEventsType) {
            throw [System.InvalidOperationException]::new(
                "Unable to resolve required internal WinForms types. This may indicate an unsupported .NET version.")
        }

        # Resolve method infos
        $GetOptionsMethodInfo = [System.Windows.Forms.FileDialog].GetMethod("GetOptions", $BindingFlags)
        $SetOptionsMethodInfo = $IFileDialogType.GetMethod("SetOptions", $BindingFlags)
        $AdviseMethodInfo     = $IFileDialogType.GetMethod("Advise")
        $UnadviseMethodInfo   = $IFileDialogType.GetMethod("Unadvise")
        $ShowMethodInfo       = $IFileDialogType.GetMethod("Show")
        $DialogMethodInfo     = [System.Windows.Forms.OpenFileDialog].GetMethod("CreateVistaDialog", $BindingFlags)
        $OnBeforeMethodInfo   = [System.Windows.Forms.OpenFileDialog].GetMethod("OnBeforeVistaDialog", $BindingFlags)

        [System.Reflection.ConstructorInfo]$DialogEventsConstructorInfo = $DialogEventsType.GetConstructor(
            $BindingFlags, $null, [System.Windows.Forms.FileDialog], $null)

        if (-not $GetOptionsMethodInfo -or -not $SetOptionsMethodInfo -or -not $ShowMethodInfo -or
            -not $DialogMethodInfo    -or -not $OnBeforeMethodInfo   -or -not $DialogEventsConstructorInfo) {
            throw [System.InvalidOperationException]::new(
                "Unable to resolve one or more required reflection methods.")
        }

        # Configure OpenFileDialog as the base object for IFileDialog
        $OpenFileDialog = New-Object System.Windows.Forms.OpenFileDialog -ErrorAction Stop
        $OpenFileDialog.AddExtension     = $false
        $OpenFileDialog.CheckFileExists  = $false
        $OpenFileDialog.DereferenceLinks = $true
        $OpenFileDialog.Title            = $Title
        $OpenFileDialog.Multiselect      = $MultiSelect.IsPresent
        $OpenFileDialog.InitialDirectory = $InitialDirectory

        # Filter is irrelevant in folder mode
        $OpenFileDialog.Filter = if ($Folder) { "Folders|" } else { $Filter }

        # Instantiate the Vista IFileDialog
        $IFileDialog = $DialogMethodInfo.Invoke($OpenFileDialog, @())
        [void]$OnBeforeMethodInfo.Invoke($OpenFileDialog, @($IFileDialog))

        # Append FOS_PICKFOLDERS flag when in folder mode
        $options = [uint32]$GetOptionsMethodInfo.Invoke($OpenFileDialog, @())
        if ($Folder) {
            [uint32]$FOSPickFolders = $FOSType.GetField("FOS_PICKFOLDERS").GetValue($null)
            $options = $options -bor $FOSPickFolders
        }
        [void]$SetOptionsMethodInfo.Invoke($IFileDialog, @($options))

        # Register dialog events
        $AdviseParams = @($DialogEventsConstructorInfo.Invoke($OpenFileDialog), [uint32]0)
        [void]$AdviseMethodInfo.Invoke($IFileDialog, $AdviseParams)

        $return = $ShowMethodInfo.Invoke($IFileDialog, [System.IntPtr]::Zero)

        # Unregister dialog events
        [void]$UnadviseMethodInfo.Invoke($IFileDialog, @($AdviseParams[1]))

        # Return selected paths, or $null if cancelled
        if ($return -eq 0) { return $OpenFileDialog.FileNames }

    } catch {
        $PSCmdlet.WriteError($_)
    }
}
```