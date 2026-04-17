<!--
id: 5818d7bb
title: Get-WebView2WpfDlls
language: powershell
tags: function, webview, wpf
description: 
created: 2026-04-17T06:37:56.913Z
updated: 2026-04-17T08:39:38
-->

# Get-WebView2WpfDlls

> 

<p><code>function</code> <code>webview</code> <code>wpf</code></p>

```powershell
function Get-WebView2WpfDlls {
<#
    .SYNOPSIS
        Finds and extract WebView2Dlls for WPF from "Microsoft.Web.WebView2" nuget package.

    .DESCRIPTION
        Finds and extract WebView2Dlls for WPF from "Microsoft.Web.WebView2" nuget package.

    .PARAMETER OutputFolder
        Specify the folder where the DLLs should be placed.

    .PARAMETER NoClean
        Prevent extracted nuget package deletion at the end of function.

    .NOTES
        Author:      Tim GILLOTIN
        Contact:     @TimGTN
        Created:     2025-04-24

        Version history:
        1.0.1 - (2026-01-30) Added support to extract NetCore Dlls too (PS7+)
        1.0.0 - (2025-04-24) Function created
#>
    param(
        [Parameter()]
        [ValidateScript({Test-Path $_ -PathType Container})]
        $OutputFolder = "C:\temp",

        [Parameter()]
        [switch]$NoClean
    )    
    try {
        $PackageName = "Microsoft.Web.WebView2"
        $Source = "https://www.nuget.org/api/v2"

        if (Find-Package -Name $PackageName -Source $Source -OutVariable Package){
            # Save the package to specified directory
            $Package | Save-Package -Path $OutputFolder -Force | Out-Null
             
            if (Get-Item -Path "$OutputFolder\*" -Include $Package.PackageFilename -OutVariable File){
                
                # Change file extension .nuget -> .nuget.zip
                $NewName = $File.FullName + ".zip"
                if (Test-Path $NewName) { Remove-Item $NewName -Force } # Remove duplicate
                $ZipFile = $File | Rename-Item -NewName $NewName -PassThru

                # Extract content from zip
                $ExtractFolder = Join-Path $OutputFolder $PackageName
                if (Test-Path $ExtractFolder) { Remove-Item $ExtractFolder -Recurse -Force } # Remove duplicate
                Expand-Archive -Path $ZipFile.FullName -DestinationPath $ExtractFolder

                ### Find and copy DLLs
                # .NET Framework (PS 5)
                $NetFrameworkDir = Get-ChildItem -Directory -Path "$ExtractFolder\lib" | Where-Object {$_.Name -notlike "*core*"} | `
                                   Sort-Object -Property Name -Descending | Select-Object -First 1
                if ($NetFrameworkDir){
                    $DestFolder = Join-Path $OutputFolder -ChildPath "NetFramework"
                    if (Test-Path $DestFolder) {Remove-Item $DestFolder -Recurse -Force}
                    New-Item $DestFolder -ItemType Directory | Out-Null

                    Copy-Item "$($NetFrameworkDir.FullName)\Microsoft.Web.WebView2.Core.dll" $DestFolder -Force
                    Copy-Item "$($NetFrameworkDir.FullName)\Microsoft.Web.WebView2.Wpf.dll" $DestFolder -Force
                    Copy-Item "$ExtractFolder\runtimes\win-x64\native\WebView2Loader.dll" $DestFolder -Force
                } else {
                    Write-Warning "Unable to find .NET Framework Dlls (for PS5). Extract them manually from `"$ExtractFolder`"."
                    $NoClean = $true
                }

                # .NET Core (PS 7+)
                $NetCoreDir = Get-ChildItem -Directory -Path "$ExtractFolder\lib_manual" | Where-Object {$_.Name -like "*core*"} | `
                                   Sort-Object -Property Name -Descending | Select-Object -First 1
                if ($NetCoreDir){
                    $DestFolder = Join-Path $OutputFolder -ChildPath "NetCore"
                    if (Test-Path $DestFolder) {Remove-Item $DestFolder -Recurse -Force}
                    New-Item $DestFolder -ItemType Directory | Out-Null
                    
                    Copy-Item "$($NetCoreDir.FullName)\Microsoft.Web.WebView2.Core.dll" $DestFolder -Force
                    Copy-Item "$($NetCoreDir.FullName)\Microsoft.Web.WebView2.Wpf.dll" $DestFolder -Force
                    Copy-Item "$ExtractFolder\runtimes\win-x64\native\WebView2Loader.dll" $DestFolder -Force
                } else {
                    Write-Warning "Unable to find .NET Core Dlls (for PS7+). Extract them manually from `"$ExtractFolder`"."
                    $NoClean = $true
                }

                # Clean
                Remove-Item $NewName -Force
                if (-not $NoClean){
                    Remove-Item $ExtractFolder -Recurse -Force    
                }
            }
        } else {
            Write-Warning -Message "Package `"$PackageName`" not found on specified source `"$Source`""
        }
    }

    # Error handling
    catch {
        $PSCmdlet.WriteError($_)
    }
}
```