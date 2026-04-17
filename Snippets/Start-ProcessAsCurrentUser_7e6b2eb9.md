<!--
id: 7e6b2eb9
title: Start-ProcessAsCurrentUser
language: powershell
tags: function, user, process
description: Start a process in the current user session
created: 2026-04-17T13:04:24.721Z
updated: 2026-04-17T15:07:44
-->

# Start-ProcessAsCurrentUser

> Start a process in the current user session

<p><code>function</code> <code>user</code> <code>process</code></p>

```powershell
function Start-ProcessAsCurrentUser {
<#
    .SYNOPSIS
        Start a process in the current user session.

    .DESCRIPTION
        Available methods:
            Token : Uses CreateProcessAsUserW via P/Invoke. Requires SYSTEM privileges.
                    Window is always hidden natively (CREATE_NO_WINDOW), no flash.
            Task  : Uses a scheduled task + VBScript (optional) to hide the process window.
                    Works without SYSTEM privileges.
            Auto  : Automatically detects whether SeTcbPrivilege is available and picks
                    the appropriate method.

        Returns a [System.Diagnostics.Process] object when -PassThru is specified.

    .PARAMETER Mode
        Auto (default), Token or Task.

    .PARAMETER FilePath
        Absolute path or command name resolved via PATH (e.g. "powershell", "notepad.exe").

    .PARAMETER ArgumentList
        Arguments passed to the executable.

    .PARAMETER WorkingDirectory
        Working directory for the launched process.

    .PARAMETER Visible
        Shows the process window.

    .PARAMETER Wait
        Waits for the process to exit before returning.

    .PARAMETER PassThru
        Returns the [System.Diagnostics.Process] object.
        Without -Wait, ExitCode is not yet available on the returned object.

    .PARAMETER Elevated
        Token mode only. Attempts to obtain the full/linked token when the session
        token is a limited UAC token.

    .EXAMPLE
        Start-ProcessAsCurrentUser -FilePath 'powershell' -ArgumentList '-Command "Start-Sleep 5"' -Wait

    .NOTES
        Author:      Tim GILLOTIN
        Contact:     @TimGTN
        Created:     2026-04-09

        Version history:
        1.0.0 - (2026-04-09) Function created
        2.0.0 - (2026-04-17) Fixed VBS written to SYSTEM temp (access denied); .lnk files now forced to Task mode
#>
    [CmdletBinding()]
    param(
        [ValidateSet('Auto', 'Token', 'Task')]
        [string]$Mode = 'Auto',

        [Parameter(Mandatory)]
        [string]$FilePath,

        [ValidateNotNullOrEmpty()]
        [string]$ArgumentList,

        [ValidateNotNullOrEmpty()]
        [string]$WorkingDirectory,

        [switch]$Visible,
        [switch]$Wait,
        [switch]$PassThru,

        [switch]$Elevated
    )

    # C# source for TOKEN mode
    if ($Mode -ne "Task" -and -not ([System.Management.Automation.PSTypeName]'RunAsUser.ProcessExtensions').Type) {
        $Source = @'
            using Microsoft.Win32.SafeHandles;
            using System;
            using System.Collections.Generic;
            using System.Runtime.InteropServices;
            using System.Security.Principal;
            using System.Text;

            namespace ProcessAsUser
            {
                internal class NativeHelpers
                {
                    [StructLayout(LayoutKind.Sequential)]
                    public struct LUID { public int LowPart; public int HighPart; }

                    [StructLayout(LayoutKind.Sequential)]
                    public struct LUID_AND_ATTRIBUTES { public LUID Luid; public PrivilegeAttributes Attributes; }

                    [StructLayout(LayoutKind.Sequential)]
                    public struct PROCESS_INFORMATION
                    {
                        public IntPtr hProcess;
                        public IntPtr hThread;
                        public int    dwProcessId;
                        public int    dwThreadId;
                    }

                    [StructLayout(LayoutKind.Sequential)]
                    public struct STARTUPINFO
                    {
                        public  int    cb;
                        public  String lpReserved;
                        public  String lpDesktop;
                        public  String lpTitle;
                        public  uint   dwX;
                        public  uint   dwY;
                        public  uint   dwXSize;
                        public  uint   dwYSize;
                        public  uint   dwXCountChars;
                        public  uint   dwYCountChars;
                        public  uint   dwFillAttribute;
                        public  uint   dwFlags;
                        public  short  wShowWindow;
                        public  short  cbReserved2;
                        public  IntPtr lpReserved2;
                        public  IntPtr hStdInput;
                        public  IntPtr hStdOutput;
                        public  IntPtr hStdError;
                    }

                    [StructLayout(LayoutKind.Sequential)]
                    public struct TOKEN_PRIVILEGES
                    {
                        public int PrivilegeCount;
                        [MarshalAs(UnmanagedType.ByValArray, SizeConst = 1)]
                        public LUID_AND_ATTRIBUTES[] Privileges;
                    }

                    [StructLayout(LayoutKind.Sequential)]
                    public struct WTS_SESSION_INFO
                    {
                        public readonly UInt32 SessionID;
                        [MarshalAs(UnmanagedType.LPStr)]
                        public readonly String pWinStationName;
                        public readonly WTS_CONNECTSTATE_CLASS State;
                    }

                    [StructLayout(LayoutKind.Sequential)]
                    public struct SECURITY_ATTRIBUTES
                    {
                        public Int32  nLength;
                        public IntPtr lpSecurityDescriptor;
                        public int    bInheritHandle;
                    }
                }

                internal class NativeMethods
                {
                    [DllImport("kernel32.dll", SetLastError = true)]
                    public static extern int WaitForSingleObject(IntPtr hHandle, int dwMilliseconds);

                    [DllImport("kernel32.dll", SetLastError = true)]
                    public static extern bool CloseHandle(IntPtr hSnapshot);

                    [DllImport("userenv.dll", SetLastError = true)]
                    public static extern bool CreateEnvironmentBlock(ref IntPtr lpEnvironment, SafeHandle hToken, bool bInherit);

                    [DllImport("advapi32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
                    public static extern bool CreateProcessAsUserW(
                        SafeHandle                            hToken,
                        String                                lpApplicationName,
                        StringBuilder                         lpCommandLine,
                        IntPtr                                lpProcessAttributes,
                        IntPtr                                lpThreadAttributes,
                        bool                                  bInheritHandle,
                        uint                                  dwCreationFlags,
                        IntPtr                                lpEnvironment,
                        String                                lpCurrentDirectory,
                        ref NativeHelpers.STARTUPINFO         lpStartupInfo,
                        out NativeHelpers.PROCESS_INFORMATION lpProcessInformation);

                    [DllImport("userenv.dll", SetLastError = true)]
                    [return: MarshalAs(UnmanagedType.Bool)]
                    public static extern bool DestroyEnvironmentBlock(IntPtr lpEnvironment);

                    [DllImport("advapi32.dll", SetLastError = true)]
                    public static extern bool DuplicateTokenEx(
                        SafeHandle                   ExistingTokenHandle,
                        uint                         dwDesiredAccess,
                        IntPtr                       lpThreadAttributes,
                        SECURITY_IMPERSONATION_LEVEL ImpersonationLevel,
                        TOKEN_TYPE                   TokenType,
                        out SafeNativeHandle         DuplicateTokenHandle);

                    [DllImport("kernel32.dll")]
                    public static extern IntPtr GetCurrentProcess();

                    [DllImport("advapi32.dll", SetLastError = true)]
                    public static extern bool GetTokenInformation(
                        SafeHandle              TokenHandle,
                        TOKEN_INFORMATION_CLASS TokenInformationClass,
                        SafeMemoryBuffer        TokenInformation,
                        int                     TokenInformationLength,
                        out int                 ReturnLength);

                    [DllImport("advapi32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
                    public static extern bool LookupPrivilegeName(
                        string                 lpSystemName,
                        ref NativeHelpers.LUID lpLuid,
                        StringBuilder          lpName,
                        ref Int32              cchName);

                    [DllImport("advapi32.dll", SetLastError = true)]
                    public static extern bool OpenProcessToken(
                        IntPtr               ProcessHandle,
                        TokenAccessLevels    DesiredAccess,
                        out SafeNativeHandle TokenHandle);

                    [DllImport("wtsapi32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
                    public static extern bool WTSEnumerateSessions(
                        IntPtr     hServer,
                        int        Reserved,
                        int        Version,
                        ref IntPtr ppSessionInfo,
                        ref int    pCount);

                    [DllImport("wtsapi32.dll")]
                    public static extern void WTSFreeMemory(IntPtr pMemory);

                    [DllImport("kernel32.dll")]
                    public static extern uint WTSGetActiveConsoleSessionId();

                    [DllImport("Wtsapi32.dll", SetLastError = true)]
                    public static extern bool WTSQueryUserToken(uint SessionId, out SafeNativeHandle phToken);
                }

                internal class SafeMemoryBuffer : SafeHandleZeroOrMinusOneIsInvalid
                {
                    public SafeMemoryBuffer(int cb) : base(true) { base.SetHandle(Marshal.AllocHGlobal(cb)); }
                    public SafeMemoryBuffer(IntPtr handle) : base(true) { base.SetHandle(handle); }
                    protected override bool ReleaseHandle() { Marshal.FreeHGlobal(handle); return true; }
                }

                internal class SafeNativeHandle : SafeHandleZeroOrMinusOneIsInvalid
                {
                    public SafeNativeHandle() : base(true) { }
                    public SafeNativeHandle(IntPtr handle) : base(true) { this.handle = handle; }
                    protected override bool ReleaseHandle() { return NativeMethods.CloseHandle(handle); }
                }

                internal enum SECURITY_IMPERSONATION_LEVEL
                {
                    SecurityAnonymous      = 0,
                    SecurityIdentification = 1,
                    SecurityImpersonation  = 2,
                    SecurityDelegation     = 3,
                }

                internal enum SW { SW_HIDE = 0, SW_SHOW = 5 }

                internal enum TokenElevationType
                {
                    TokenElevationTypeDefault = 1,
                    TokenElevationTypeFull,
                    TokenElevationTypeLimited,
                }

                internal enum TOKEN_TYPE { TokenPrimary = 1, TokenImpersonation = 2 }

                internal enum TOKEN_INFORMATION_CLASS
                {
                    TokenPrivileges    = 3,
                    TokenElevationType = 18,
                    TokenLinkedToken   = 19,
                }

                internal enum WTS_CONNECTSTATE_CLASS
                {
                    WTSActive, WTSConnected, WTSConnectQuery, WTSShadow,
                    WTSDisconnected, WTSIdle, WTSListen, WTSReset, WTSDown, WTSInit,
                }

                [Flags]
                public enum PrivilegeAttributes : uint
                {
                    Disabled         = 0x00000000,
                    EnabledByDefault = 0x00000001,
                    Enabled          = 0x00000002,
                    Removed          = 0x00000004,
                    UsedForAccess    = 0x80000000,
                }

                public class Win32Exception : System.ComponentModel.Win32Exception
                {
                    private readonly string _msg;
                    public Win32Exception(string message) : this(Marshal.GetLastWin32Error(), message) { }
                    public Win32Exception(int errorCode, string message) : base(errorCode)
                    {
                        _msg = String.Format("{0} ({1}, Win32ErrorCode {2} - 0x{2:X8})", message, base.Message, errorCode);
                    }
                    public override string Message { get { return _msg; } }
                }

                public static class ProcessExtensions
                {
                    private const int  CREATE_UNICODE_ENVIRONMENT = 0x00000400;
                    private const int  CREATE_NO_WINDOW           = 0x08000000;
                    private const int  CREATE_NEW_CONSOLE         = 0x00000010;
                    private const uint INVALID_SESSION_ID         = 0xFFFFFFFF;
                    private static readonly IntPtr WTS_CURRENT_SERVER_HANDLE = IntPtr.Zero;

                    public static int StartProcessAsCurrentUser(
                        string appPath,
                        string cmdLine  = null,
                        string workDir  = null,
                        bool   visible  = false,
                        int    waitMs   = 0,
                        bool   elevated = false)
                    {
                        var si = new NativeHelpers.STARTUPINFO
                        {
                            cb          = Marshal.SizeOf(typeof(NativeHelpers.STARTUPINFO)),
                            lpDesktop   = "winsta0\\default",
                            wShowWindow = (short)(visible ? SW.SW_SHOW : SW.SW_HIDE),
                        };

                        uint creationFlags = CREATE_UNICODE_ENVIRONMENT |
                                            (uint)(visible ? CREATE_NEW_CONSOLE : CREATE_NO_WINDOW);

                        var procInfo  = new NativeHelpers.PROCESS_INFORMATION();
                        var cmdLineSb = cmdLine != null ? new StringBuilder(cmdLine) : null;

                        using (var hToken = GetSessionUserToken(elevated))
                        {
                            var pEnv = IntPtr.Zero;
                            if (!NativeMethods.CreateEnvironmentBlock(ref pEnv, hToken, false))
                                throw new Win32Exception("CreateEnvironmentBlock failed.");
                            try
                            {
                                if (!NativeMethods.CreateProcessAsUserW(
                                        hToken, appPath, cmdLineSb,
                                        IntPtr.Zero, IntPtr.Zero, false,
                                        creationFlags, pEnv, workDir,
                                        ref si, out procInfo))
                                    throw new Win32Exception("CreateProcessAsUserW failed.");

                                if (waitMs != 0)
                                    NativeMethods.WaitForSingleObject(procInfo.hProcess, waitMs);
                            }
                            finally
                            {
                                NativeMethods.DestroyEnvironmentBlock(pEnv);
                            }
                        }

                        // Always close handles -- caller receives the PID and uses Get-Process
                        CloseIfValid(ref procInfo.hThread);
                        CloseIfValid(ref procInfo.hProcess);

                        return procInfo.dwProcessId;
                    }

                    public static bool HasTcbPrivilege()
                    {
                        try
                        {
                            var privs = GetTokenPrivileges();
                            PrivilegeAttributes attr;
                            if (!privs.TryGetValue("SeTcbPrivilege", out attr)) return false;
                            return (attr & (PrivilegeAttributes.Enabled | PrivilegeAttributes.EnabledByDefault)) != 0;
                        }
                        catch { return false; }
                    }

                    private static SafeNativeHandle GetSessionUserToken(bool elevated)
                    {
                        var activeSessionId = INVALID_SESSION_ID;
                        var pSessionInfo    = IntPtr.Zero;
                        var sessionCount    = 0;

                        if (NativeMethods.WTSEnumerateSessions(
                                WTS_CURRENT_SERVER_HANDLE, 0, 1, ref pSessionInfo, ref sessionCount))
                        {
                            try
                            {
                                var elementSize = Marshal.SizeOf(typeof(NativeHelpers.WTS_SESSION_INFO));
                                var current     = pSessionInfo;
                                for (var i = 0; i < sessionCount; i++)
                                {
                                    var si = (NativeHelpers.WTS_SESSION_INFO)Marshal.PtrToStructure(
                                        current, typeof(NativeHelpers.WTS_SESSION_INFO));
                                    current = IntPtr.Add(current, elementSize);
                                    if (si.State == WTS_CONNECTSTATE_CLASS.WTSActive)
                                    {
                                        activeSessionId = si.SessionID;
                                        break;
                                    }
                                }
                            }
                            finally { NativeMethods.WTSFreeMemory(pSessionInfo); }
                        }

                        if (activeSessionId == INVALID_SESSION_ID)
                            activeSessionId = NativeMethods.WTSGetActiveConsoleSessionId();

                        SafeNativeHandle hImpersonation;
                        if (!NativeMethods.WTSQueryUserToken(activeSessionId, out hImpersonation))
                            throw new Win32Exception("WTSQueryUserToken failed.");

                        using (hImpersonation)
                        {
                            if (elevated &&
                                GetTokenElevationType(hImpersonation) == TokenElevationType.TokenElevationTypeLimited)
                            {
                                using (var linked = GetTokenLinkedToken(hImpersonation))
                                    return DuplicateTokenAsPrimary(linked);
                            }
                            return DuplicateTokenAsPrimary(hImpersonation);
                        }
                    }

                    private static SafeNativeHandle DuplicateTokenAsPrimary(SafeHandle hToken)
                    {
                        SafeNativeHandle dup;
                        if (!NativeMethods.DuplicateTokenEx(hToken, 0, IntPtr.Zero,
                                SECURITY_IMPERSONATION_LEVEL.SecurityImpersonation,
                                TOKEN_TYPE.TokenPrimary, out dup))
                            throw new Win32Exception("DuplicateTokenEx failed.");
                        return dup;
                    }

                    private static TokenElevationType GetTokenElevationType(SafeHandle hToken)
                    {
                        using (var buf = GetTokenInformation(hToken, TOKEN_INFORMATION_CLASS.TokenElevationType))
                            return (TokenElevationType)Marshal.ReadInt32(buf.DangerousGetHandle());
                    }

                    private static SafeNativeHandle GetTokenLinkedToken(SafeHandle hToken)
                    {
                        using (var buf = GetTokenInformation(hToken, TOKEN_INFORMATION_CLASS.TokenLinkedToken))
                            return new SafeNativeHandle(Marshal.ReadIntPtr(buf.DangerousGetHandle()));
                    }

                    private static SafeMemoryBuffer GetTokenInformation(SafeHandle hToken, TOKEN_INFORMATION_CLASS infoClass)
                    {
                        int returnLength;
                        NativeMethods.GetTokenInformation(hToken, infoClass,
                            new SafeMemoryBuffer(IntPtr.Zero), 0, out returnLength);

                        int err = Marshal.GetLastWin32Error();
                        // 122 = ERROR_INSUFFICIENT_BUFFER, 24 = ERROR_BAD_LENGTH -- both expected on sizing call
                        if (err != 122 && err != 24)
                            throw new Win32Exception(err,
                                String.Format("GetTokenInformation({0}) sizing call failed.", infoClass));

                        var buf = new SafeMemoryBuffer(returnLength);
                        if (!NativeMethods.GetTokenInformation(hToken, infoClass, buf, returnLength, out returnLength))
                            throw new Win32Exception(String.Format("GetTokenInformation({0}) failed.", infoClass));
                        return buf;
                    }

                    private static SafeNativeHandle OpenProcessToken(IntPtr process, TokenAccessLevels access)
                    {
                        SafeNativeHandle hToken;
                        if (!NativeMethods.OpenProcessToken(process, access, out hToken))
                            throw new Win32Exception("OpenProcessToken failed.");
                        return hToken;
                    }

                    private static Dictionary<string, PrivilegeAttributes> GetTokenPrivileges()
                    {
                        var result = new Dictionary<string, PrivilegeAttributes>();
                        using (var hToken   = OpenProcessToken(NativeMethods.GetCurrentProcess(), TokenAccessLevels.Query))
                        using (var tokenBuf = GetTokenInformation(hToken, TOKEN_INFORMATION_CLASS.TokenPrivileges))
                        {
                            var info = (NativeHelpers.TOKEN_PRIVILEGES)Marshal.PtrToStructure(
                                tokenBuf.DangerousGetHandle(), typeof(NativeHelpers.TOKEN_PRIVILEGES));

                            var ptr = IntPtr.Add(tokenBuf.DangerousGetHandle(),
                                Marshal.SizeOf(info.PrivilegeCount));

                            for (int i = 0; i < info.PrivilegeCount; i++)
                            {
                                var la = (NativeHelpers.LUID_AND_ATTRIBUTES)Marshal.PtrToStructure(
                                    ptr, typeof(NativeHelpers.LUID_AND_ATTRIBUTES));

                                int nameLen = 0;
                                var luid    = la.Luid;
                                NativeMethods.LookupPrivilegeName(null, ref luid, null, ref nameLen);

                                var name = new StringBuilder(nameLen + 1);
                                if (!NativeMethods.LookupPrivilegeName(null, ref luid, name, ref nameLen))
                                    throw new Win32Exception("LookupPrivilegeName failed.");

                                result[name.ToString()] = la.Attributes;
                                ptr = IntPtr.Add(ptr, Marshal.SizeOf(typeof(NativeHelpers.LUID_AND_ATTRIBUTES)));
                            }
                        }
                        return result;
                    }

                    private static void CloseIfValid(ref IntPtr handle)
                    {
                        if (handle != IntPtr.Zero)
                        {
                            NativeMethods.CloseHandle(handle);
                            handle = IntPtr.Zero;
                        }
                    }
                }
            }
'@
        try { Add-Type -TypeDefinition $Source -Language CSharp -ErrorAction Stop }
        catch { 
            $PSCmdlet.WriteError($_)
            if ($Mode -ne "Token") {
                Write-Warning "TOKEN mode isn't available, fallback to TASK mode."
                $Mode = 'Task' # Fallback
            } else {
                throw "TOKEN mode isn't available."
            }
        }
    }

    # AUTO - Pick the suitable mode between TOKEN and TASK
    if ($Mode -eq "Auto") {
        $Mode = if ([ProcessAsUser.ProcessExtensions]::HasTcbPrivilege()) { 'Token' } else { 'Task' }
        Write-Verbose "AUTO mode resolsed to : $($Mode.ToUpper())"
    }

    # Token mode cannot resolve .lnk files; Task mode also preserves the shortcut icon
    if ([IO.Path]::GetExtension($FilePath) -eq '.lnk' -and $Mode -eq 'Token') {
        Write-Verbose "Shortcut detected (.lnk), switching to TASK mode."
        $Mode = 'Task'
    }

    # FilePath resolution
    if (Get-command $FilePath -OutVariable Resolved -ErrorAction SilentlyContinue) {      
        Write-Verbose "FilePath resolved to : `"$($Resolved.Source)`"."
    } else {
        throw "Cannot find executable '$FilePath'."
    }

    # WorkingDirectory validation
    if ($PSBoundParameters.ContainsKey('WorkingDirectory') -and
        -not (Test-Path $WorkingDirectory -PathType Container)) {

        throw "Working directory '$WorkingDirectory' does not exist."
    }

    # User session validation
    New-PSDrive -PSProvider Registry -Root HKEY_USERS -Name HKU -ErrorAction SilentlyContinue | Out-Null
    $User = Get-ItemProperty "HKU:\*\Volatile Environment" -ErrorAction SilentlyContinue |
            Select-Object USERDOMAIN, USERNAME, LOCALAPPDATA
    if ($User) {
        $UserName = $User.USERDOMAIN, $User.USERNAME -join "\"
        Write-Verbose "User connected : `"$UserName`"."
    } else {    
        throw "Cannot find connected user."
    }

    # ==========
    # TOKEN mode
    # ==========
    if ($Mode -eq "Token") {

        $ProcessId = [ProcessAsUser.ProcessExtensions]::StartProcessAsCurrentUser(
            $FilePath,
            $ArgumentList,
            $WorkingDirectory,
            $Visible.IsPresent,
            0,
            $Elevated.IsPresent
        )
        $Process = Get-Process -Id $ProcessId -ErrorAction SilentlyContinue
        $Process.EnableRaisingEvents = $True
    }

    # =========
    # TASK mode
    # =========
    elseif ($Mode -eq "Task") {

        if ($Elevated) {
            Write-Verbose "Parameter `"-Elevated`" is not available for TASK mode and will be ignored."
        }

        # Helper to retry commands
        function Wait-Until {
            param(
                [scriptblock] $Scriptblock,
                [int]         $TimeoutSeconds = 30,
                [int]         $IntervalMs     = 200
            )

            $Stopwatch = [System.Diagnostics.Stopwatch]::StartNew()
            do {
                $Result = & $Scriptblock
                if ($Result) { $Stopwatch.Stop(); return $Result }
                Start-Sleep -Milliseconds $IntervalMs
            } while ($Stopwatch.Elapsed.TotalSeconds -lt $TimeoutSeconds)

            $Stopwatch.Stop()
            return $null
        }
        
        # Task name
        $ShortGuid = [guid]::NewGuid().ToString('N').Substring(0, 8)
        $BaseName  = [IO.Path]::GetFileNameWithoutExtension($FilePath)
        $TaskName  = "PAU_{0}_{1}" -f $BaseName, $ShortGuid

        # Task action
        Write-Verbose "Process window visibility : $($Visible.IsPresent)."
        if ($Visible) {
            $Action = New-ScheduledTaskAction -Execute $FilePath -Argument $ArgumentList
        } else {
            $Command = "`"$FilePath`" $ArgumentList".Trim().Replace('"', '""')

            # Use the user's TEMP folder (GetTempPath() returns SYSTEM's temp when running as SYSTEM)
            $Temp = if ([System.IO.Directory]::Exists($User.LOCALAPPDATA)) { $User.LOCALAPPDATA } else { "$env:SystemRoot\Temp" }
            $VbsPath = [IO.Path]::Combine($Temp, "$TaskName.vbs")

            $VbsContent = "
                Rtn = CreateObject(`"Wscript.Shell`").Run(`"$Command`", 0, True)
                If IsEmpty(Rtn) Or IsNull(Rtn) Then Rtn = 666
                WScript.Quit Rtn
            "
            $VbsContent | Set-Content -Path $VbsPath -Encoding ASCII
            Write-Verbose "VBS wrapper created : `"$VbsPath`"."
            $Action = New-ScheduledTaskAction -Execute 'wscript.exe' -Argument "`"$VbsPath`""
        }
        if ($WorkingDirectory) { $Action.WorkingDirectory = $WorkingDirectory }

        # Register and start the task
        $Settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries

        try {
            $Task = Register-ScheduledTask -TaskName $TaskName `
                                           -Action   $Action `
                                           -User     $UserName `
                                           -Settings $Settings `
                                           -Force `
                                           -ErrorAction Stop
            $Date = [datetime]::Now # To filter events
            Start-ScheduledTask -InputObject $Task       

            # Wait for the task engine to spin up
            $TaskService = New-Object -ComObject 'Schedule.Service'
            $TaskService.Connect()

            $RunningTask = Wait-Until {
                $TaskService.GetRunningTasks(1) | Where-Object { $_.Name -eq $TaskName }
            } -IntervalMs 50

            # Handle when task didn't start
            if (-not $RunningTask) {
                $Events = Get-WinEvent -FilterHashtable @{
                    LogName   = 'Microsoft-Windows-TaskScheduler/Operational'
                    Level     = 2 # Error
                    StartTime = $Date
                } -ErrorAction SilentlyContinue

                foreach ($evt in $events | Sort-Object TimeCreated) {
                    Write-Warning $evt.Message
                }

                throw "Scheduled task did not start."
            }

            ## Task has started
            Write-Verbose "Scheduled task started (EnginePID: $($RunningTask.EnginePID))."

            # Resolve the target process
            $EnginePID = $RunningTask.EnginePID
            $Process = Wait-Until {
                if ($Visible) {
                    Get-Process -Id $EnginePID -ErrorAction SilentlyContinue
                } else {
                    Get-CimInstance Win32_Process -Filter "ParentProcessId = $EnginePID" -Verbose:$false |
                        Select-Object -First 1 |
                        ForEach-Object { Get-Process -Id $_.ProcessId -ErrorAction SilentlyContinue }
                }
            } -IntervalMs 50

            if ($Process) {
                Write-Verbose "Target process identified (PID: $($Process.Id))."
                if ($Passthru) { $Process.EnableRaisingEvents = $True }
            } else {
                throw "Target process could not be identified."
            }
        } 
        catch {
            $PSCmdlet.ThrowTerminatingError($_)
        } 
        finally {
            Unregister-ScheduledTask -InputObject $Task -Confirm:$false -ErrorAction SilentlyContinue
            if ($VbsPath -and (Test-Path $VbsPath -ErrorAction SilentlyContinue)) {
                Remove-Item -Path $VbsPath -Force -ErrorAction SilentlyContinue
            }
        }    
    }

    # Wait
    if ($Wait) {
        Write-Verbose "Waiting for process (PID: $($Process.Id)) to exit..."
        Wait-Process -Id $Process.Id
        Write-Verbose "Process exited (ExitCode: $($Process.ExitCode))."
    }

    if ($PassThru -and $Process) {
        return $Process
    } else {
        return
    }
}
```