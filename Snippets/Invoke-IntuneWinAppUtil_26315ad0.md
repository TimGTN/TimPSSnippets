<!--
id: 26315ad0
title: Invoke-IntuneWinAppUtil
language: powershell
tags: function, intune, app, #394e6eb0
description: Packages an application into the .intunewin format required by Microsoft Intune
created: 2026-04-17T21:00:20.258Z
updated: 2026-04-18T00:33:41
-->

# Invoke-IntuneWinAppUtil

> Packages an application into the .intunewin format required by Microsoft Intune

<p><code>function</code> <code>intune</code> <code>app</code></p>

```powershell
function Invoke-IntuneWinAppUtil {
    <#
        .SYNOPSIS
            Packages an application into the .intunewin format required by Microsoft Intune.

        .DESCRIPTION
            Wraps the Microsoft Win32 Content Prep Tool (IntuneWinAppUtil.exe).
            If no custom path is provided, the tool is automatically downloaded to the user's
            temp folder and removed after execution.
            Use -Interactive to select the source folder and setup file via picker dialogs.

        .PARAMETER SourceFolder
            Path to the folder containing the application source files.
            Mandatory when -Interactive is not specified.

        .PARAMETER SetupFile
            Name or full path of the setup file (e.g. setup.exe, install.msi).
            If only a filename is provided, it is resolved relative to SourceFolder.
            Mandatory when -Interactive is not specified.

        .PARAMETER OutputFolder
            Path where the .intunewin package will be saved.
            Defaults to SourceFolder if not specified. Created automatically if missing.

        .PARAMETER AppUtilPath
            Full path to IntuneWinAppUtil.exe.
            Defaults to a temporary path — the tool is downloaded automatically
            and deleted after execution.

        .PARAMETER Interactive
            Opens a folder picker to select SourceFolder, then a file picker to select SetupFile.
            When specified, SourceFolder and SetupFile are not required.

        .EXAMPLE
            Invoke-IntuneWinAppUtil -SourceFolder "C:\Apps\MyApp" -SetupFile "setup.exe"

            Packages setup.exe from C:\Apps\MyApp and saves the .intunewin to the same folder.

        .EXAMPLE
            Invoke-IntuneWinAppUtil -Interactive

            Opens picker dialogs to select the source folder and setup file.

        .OUTPUTS
            System.IO.FileInfo
            Returns the generated .intunewin file on success.

        .LINK
            Invoke-ItemPickerDialog

        .LINK
            https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2026-04-17

            Version history:
            1.0.0 - (2026-04-17) Function created
    #>
    [CmdletBinding(SupportsShouldProcess = $true, DefaultParameterSetName = "Manual")]
    param(
        [Parameter(Mandatory, ParameterSetName = "Manual")]
        [ValidateScript({ Test-Path $_ -PathType Container })]
        [string]$SourceFolder,

        [Parameter(Mandatory, ParameterSetName = "Manual")]
        [string]$SetupFile,

        [Parameter(ParameterSetName = "Manual")]
        [Parameter(ParameterSetName = "Interactive")]
        [string]$OutputFolder,

        [Parameter(Mandatory, ParameterSetName = "Interactive")]
        [switch]$Interactive,

        [ValidateNotNullOrEmpty()]
        [string]$AppUtilPath = (Join-Path -Path "C:\temp" -ChildPath "IntuneWinAppUtil.exe")
    )

    try {
        # Detect whether AppUtilPath was explicitly provided or is using the default value
        $isDefaultAppUtilPath = -not $PSBoundParameters.ContainsKey('AppUtilPath')

        if ($Interactive) {
            # Step 1 — Select SourceFolder via modern folder picker
            $SourceFolder = Invoke-ItemPickerDialog -Title "Select the application source folder" -Folder
            if (-not $SourceFolder) {
                Write-Warning "No folder selected. Aborting."
                return
            }

            # Step 2 — Select SetupFile via file picker, rooted in SourceFolder
            $SetupFile = Invoke-ItemPickerDialog -Title "Select the setup file" `
                -InitialDirectory $SourceFolder `
                -Filter "Setup files (*.exe;*.msi;*.msp;*.bat;*.cmd;*.ps1)|*.exe;*.msi;*.msp;*.bat;*.cmd;*.ps1|All files (*.*)|*.*"

            if (-not $SetupFile) {
                Write-Warning "No file selected. Aborting."
                return
            }

            # Warn if the selected file is outside SourceFolder — IntuneWinAppUtil requires -s to be inside -c
            if (-not $SetupFile.StartsWith($SourceFolder, [System.StringComparison]::OrdinalIgnoreCase)) {
                Write-Warning "The selected file is outside the source folder. IntuneWinAppUtil may not package it correctly."
            }
        } else {
            # Resolve SourceFolder to an absolute path
            $SourceFolder = (Resolve-Path -Path $SourceFolder).Path

            # If SetupFile is a relative path or filename, resolve it from SourceFolder
            if (-not [System.IO.Path]::IsPathRooted($SetupFile)) {
                $SetupFile = Join-Path -Path $SourceFolder -ChildPath $SetupFile
            }

            if (-not (Test-Path -Path $SetupFile -PathType Leaf)) {
                throw [System.IO.FileNotFoundException]::new(
                    "Setup file '$SetupFile' could not be found.")
            }
        }

        # Default OutputFolder to SourceFolder if not specified
        if (-not $OutputFolder) {
            $OutputFolder = $SourceFolder
        } elseif (-not [System.IO.Path]::IsPathRooted($OutputFolder)) {
            $OutputFolder = (Resolve-Path -Path $OutputFolder).Path
        }

        # Create OutputFolder if it does not exist
        if (-not (Test-Path -Path $OutputFolder)) {
            New-Item -ItemType Directory -Path $OutputFolder -Force | Out-Null
            Write-Verbose "Output folder created: $OutputFolder"
        }

        # Download IntuneWinAppUtil.exe if using the default path and not already present
        if ($isDefaultAppUtilPath -and -not (Test-Path -Path $AppUtilPath)) {
            Write-Verbose "Downloading IntuneWinAppUtil.exe to '$AppUtilPath'..."
            Invoke-WebRequest -Uri "https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool/raw/master/IntuneWinAppUtil.exe" `
                -OutFile $AppUtilPath -UseBasicParsing
        } elseif (-not (Test-Path -Path $AppUtilPath)) {
            throw [System.IO.FileNotFoundException]::new(
                "IntuneWinAppUtil.exe could not be found at the specified path: '$AppUtilPath'")
        }

        try {
            # Use only the filename for -s, as the tool resolves it relative to SourceFolder
            $SetupFileName = Split-Path -Path $SetupFile -Leaf
            $Arguments = "-c `"$SourceFolder`" -s `"$SetupFileName`" -o `"$OutputFolder`" -q"

            Write-Verbose "Running: $AppUtilPath $Arguments"

            if ($PSCmdlet.ShouldProcess($SetupFile, "Create .intunewin package")) {
                $Process = Start-Process -FilePath $AppUtilPath -ArgumentList $Arguments`
                                         -Wait -PassThru
                if ($process.ExitCode -ne 0) {
                    throw "IntuneWinAppUtil.exe failed with exit code $($Process.ExitCode)."
                }
            }

            # Return the generated .intunewin file
            $OutputFile = Join-Path -Path $OutputFolder -ChildPath (
                [System.IO.Path]::GetFileNameWithoutExtension($SetupFileName) + ".intunewin")

            if (Test-Path $OutputFile) {
                return Get-Item -Path $OutputFile
            }
        } finally {
            # Clean up the executable if it was downloaded to the default temp path
            if ($isDefaultAppUtilPath -and (Test-Path -Path $AppUtilPath)) {
                Write-Verbose "Removing temporary executable '$AppUtilPath'."
                Remove-Item -Path $AppUtilPath -Force
            }
        }

    } catch {
        $PSCmdlet.WriteError($_)
    }
}

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
        [ValidateNotNullOrEmpty()]
        [string]$Title = "Select",

        [ValidateNotNullOrEmpty()]
        [string]$InitialDirectory = [Environment]::GetFolderPath("Desktop"),

        [ValidateNotNullOrEmpty()]
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