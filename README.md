# rudesktop-reset-trial
Widnows .bat script file that resets rudesktop (remote PC control) trial licence

```bat
@echo off
setlocal EnableExtensions
title RuDesktop ID Reset

rem ===========================================================================
rem  Stops the RuDesktop service and every RuDesktop process, sets  id = '0'
rem  in both RuDesktop.toml config files (user + service), removes the
rem  HKLM\SOFTWARE\RuDesktop registry key (both registry views), then starts
rem  back whatever was actually running before.
rem  A .bak copy of each config and a .reg export of each deleted key are kept.
rem ===========================================================================

rem --- needs admin: the service and its config live under %WINDIR% -----------
net session >nul 2>&1
if errorlevel 1 (
    echo Requesting administrator rights...
    powershell -NoProfile -Command "Start-Process -FilePath '%~f0' -Verb RunAs"
    exit /b
)

set "SVC=RuDesktop"
set "USERCFG=%APPDATA%\RuDesktop\config\RuDesktop.toml"
set "SVCCFG=%WINDIR%\ServiceProfiles\LocalService\AppData\Roaming\RuDesktop\config\RuDesktop.toml"
set "REGKEY=HKLM\SOFTWARE\RuDesktop"

rem --- if elevation switched to another account, find the real user config ---
if not exist "%USERCFG%" (
    for /d %%U in ("%SystemDrive%\Users\*") do (
        if exist "%%~fU\AppData\Roaming\RuDesktop\config\RuDesktop.toml" set "USERCFG=%%~fU\AppData\Roaming\RuDesktop\config\RuDesktop.toml"
    )
)

rem --- resolve rudesktop.exe from the service registration -------------------
set "EXE="
for /f "usebackq delims=" %%P in (`powershell -NoProfile -Command "$p=(Get-CimInstance Win32_Service -Filter ('Name=' + [char]39 + 'RuDesktop' + [char]39)).PathName; if ($p) { $p = $p.Trim([char]34, [char]32); $i = $p.ToLower().IndexOf('rudesktop.exe'); if ($i -ge 0) { $p.Substring(0, $i + 13).Trim([char]34) } }"`) do set "EXE=%%P"
if not defined EXE set "EXE=A:\Program Files\RuDesktop\rudesktop.exe"
if not exist "%EXE%" set "EXE=%ProgramFiles%\RuDesktop\rudesktop.exe"

echo.
echo  service config : %SVCCFG%
echo  user config    : %USERCFG%
echo  registry key   : %REGKEY%
echo  executable     : %EXE%
echo.

rem --- remember what is running right now ------------------------------------
set "WASSVC=0"
set "WASAPP=0"
sc query "%SVC%" 2>nul | find "RUNNING" >nul && set "WASSVC=1"
tasklist /fi "imagename eq rudesktop.exe" /fi "sessionname eq Console" 2>nul | find /i "rudesktop.exe" >nul && set "WASAPP=1"
echo  [state] service running: %WASSVC%    desktop app running: %WASAPP%
echo.

rem --- stop everything --------------------------------------------------------
echo  [1/4] stopping RuDesktop...
sc stop "%SVC%" >nul 2>&1
for /l %%i in (1,1,20) do (
    sc query "%SVC%" 2>nul | find "STOPPED" >nul && goto :svc_stopped
    ping -n 2 127.0.0.1 >nul
)
:svc_stopped
taskkill /f /t /im rudesktop.exe >nul 2>&1
taskkill /f /t /im RuntimeBroker_rudesktop.exe >nul 2>&1
ping -n 3 127.0.0.1 >nul

rem --- rewrite the id ---------------------------------------------------------
echo  [2/4] setting id = '0' ...
call :SetId "%SVCCFG%"
call :SetId "%USERCFG%"

rem --- drop the HKLM registry data --------------------------------------------
echo  [3/4] removing %REGKEY% ...
set "STAMP="
for /f "usebackq delims=" %%T in (`powershell -NoProfile -Command "(Get-Date).ToString('yyyyMMdd-HHmmss')"`) do set "STAMP=%%T"
if not defined STAMP set "STAMP=backup"
call :DelKey 64 "%~dp0RuDesktop-HKLM-64bit-%STAMP%.reg"
call :DelKey 32 "%~dp0RuDesktop-HKLM-32bit-%STAMP%.reg"

rem --- start back only what was running ---------------------------------------
echo  [4/4] restarting...
if "%WASSVC%"=="1" (
    sc start "%SVC%" >nul 2>&1
    echo        service started
) else (
    echo        service was not running - left stopped
)
if "%WASAPP%"=="1" (
    if exist "%EXE%" (
        ping -n 2 127.0.0.1 >nul
        start "" "%EXE%"
        echo        desktop app started
    ) else (
        echo        [warn] executable not found: %EXE%
    )
) else (
    echo        desktop app was not running - not started
)

echo.
echo  Done. RuDesktop will generate a fresh ID on next start.
echo.
pause
exit /b 0

rem ===========================================================================
:SetId
set "TOMLFILE=%~1"
if not exist "%TOMLFILE%" (
    echo        [skip] not found: %TOMLFILE%
    goto :eof
)
copy /y "%TOMLFILE%" "%TOMLFILE%.bak" >nul 2>&1
powershell -NoProfile -ExecutionPolicy Bypass -Command "$f = $env:TOMLFILE; $t = [IO.File]::ReadAllText($f); $rx = [regex]'(?m)^id[ \t]*=.*$'; $old = if ($rx.IsMatch($t)) { $rx.Match($t).Value } else { '(none)' }; if ($rx.IsMatch($t)) { $t = $rx.Replace($t, 'id = ''0''', 1) } else { $t = 'id = ''0''' + [Environment]::NewLine + $t }; [IO.File]::WriteAllText($f, $t, (New-Object Text.UTF8Encoding $false)); Write-Host ('       ' + $old + '  ->  ' + 'id = ' + [char]39 + '0' + [char]39)"
goto :eof

rem ===========================================================================
rem  :DelKey  <32|64>  <backup .reg file>
rem  Exports %REGKEY% from the given registry view, then deletes it. A missing
rem  key is skipped, and the key is only deleted once the backup was written.
rem  Restore later with:  reg import "<backup .reg file>"
:DelKey
set "VIEW=%~1"
set "REGBAK=%~2"
reg query "%REGKEY%" /reg:%VIEW% >nul 2>&1
if errorlevel 1 (
    echo        [skip] %VIEW%-bit view: key not present
    goto :eof
)
reg export "%REGKEY%" "%REGBAK%" /y /reg:%VIEW% >nul 2>&1
if errorlevel 1 (
    echo        [abort] %VIEW%-bit view: export failed, key left in place
    goto :eof
)
reg delete "%REGKEY%" /f /reg:%VIEW% >nul 2>&1
if errorlevel 1 (
    echo        [fail] %VIEW%-bit view: delete failed
    goto :eof
)
echo        %VIEW%-bit view: deleted
echo        backup: %REGBAK%
goto :eof
```
