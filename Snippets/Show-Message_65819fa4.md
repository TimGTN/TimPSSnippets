<!--
id: 65819fa4
title: Show-Message
language: powershell
tags: function, dialog
description: Displays a Windows Forms message box and optionally returns the user's choice
created: 2026-04-17T09:42:25.085Z
updated: 2026-04-17T22:26:29
-->

# Show-Message

> Displays a Windows Forms message box and optionally returns the user's choice

<p><code>function</code> <code>dialog</code></p>

```powershell
function Show-Message {
    <#
        .SYNOPSIS
            Displays a Windows Forms message box and optionally returns the user's choice.

        .DESCRIPTION
            Wraps System.Windows.Forms.MessageBox::Show with named, validated parameters
            for button set, icon type, and window title.
            Returns a DialogResult value unless -NoReturn is specified.

        .PARAMETER Message
            The text to display in the message box body.

        .PARAMETER Title
            The message box window title.

        .PARAMETER Button
            The set of buttons to display. Defaults to Ok.
            Valid values: Ok, OkCancel, AbortRetryIgnore, YesNoCancel, YesNo, RetryCancel.

        .PARAMETER Type
            The icon to display. Defaults to None.
            Valid values: None, Information, Warning, Error, Question.

        .PARAMETER NoReturn
            When specified, suppresses the DialogResult return value.

        .OUTPUTS
            System.Windows.Forms.DialogResult
            None — when -NoReturn is specified.

        .EXAMPLE
            Show-Message -Message "Operation complete." -Type Information

        .EXAMPLE
            $result = Show-Message -Message "Continue?" -Button YesNo -Type Question
            if ($result -eq 'Yes') { ... }

        .NOTES
            Author:      Tim GILLOTIN
            Contact:     @TimGTN
            Created:     2023-05-12
            Version history:
            1.0.0 - (2023-05-12) Function created
            2.0.0 - (2026-04-17) Add-Type guard
    #>
    [OutputType([System.Windows.Forms.DialogResult])]
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$Message,

        [string]$Title = '',

        [ValidateSet('Ok', 'OkCancel', 'AbortRetryIgnore', 'YesNoCancel', 'YesNo', 'RetryCancel')]
        [string]$Button = 'Ok',

        [ValidateSet('None', 'Information', 'Warning', 'Error', 'Question')]
        [string]$Type = 'None',

        [switch]$NoReturn
    )

    begin {
        if (-not ('System.Windows.Forms.MessageBox' -as [type])) {
            Add-Type -AssemblyName System.Windows.Forms
        }
    }

    process {
        try {
            $Result = [System.Windows.Forms.MessageBox]::Show(
                $Message,
                $Title,
                [System.Windows.Forms.MessageBoxButtons]::$Button,
                [System.Windows.Forms.MessageBoxIcon]::$Type
            )
            if (-not $NoReturn) { return $Result }
        }
        catch {
            $PSCmdlet.WriteError($_)
        }
    }
}
```