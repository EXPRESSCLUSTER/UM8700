# armload into service resources for UM8700 cluster

This memo describes how to upgrade UM8700 clustering with EXPRESSCLUSTER X3.3 into EXPRESSCLUSTER X5.1.

---

Assume the script resource `UM8700 Services Script` has the following start.bat and stop.bat in it.

- start.bat

  ```bat
  rem ***************************************
  rem Normal Startup process
  rem ***************************************
  net share D$=D: /grant:everyone,full
  net share Reports=d:\cx\reports /grant:everyone,full
  RMDIR /S /Q D:\CX\SentinelCloudUsage
  MKDIR D:\CX\SentinelCloudUsage
  armload PERLE /S /A /R 3 "TruePortSrv"
  armload MYSQL /S /A /R 3 "MySQLCore"
  armload NRS /S /A /R 3 "Nuance Recognition Service"
  armload NSS /S /A /R 3 "NuanceSpeechServer"
  armload SLM /S /A /R 3 "hasplms"
  armload CXBS /S /A /R 3 "AT_GenBackground"
  armload CXLS /S /A /R 3 "AT_LicServer"
  armload CXDS /S /A /R 3 "AT_DAL"
  armload CXCS /S /A /R 3 "CDAUSServer"
  armload CXFM /S /A /R 3 "AT_FileManager"
  armload CXGDS /S /A /R 3 "CXGData"
  armload CXGS /S /A /R 3 "AT_GrammarServer"
  armload CXSS /S /A /R 3 "AT_SOAPServer"
  armload CXTM /S /A /R 3 "AT_TextMessaging"
  armload CX /S /A /R 3 /WAIT 0 "Launcher"
  armload CXUS /S /A "UCConnect Server"
  ```

- stop.bat

  ```bat
  rem ***************************************
  rem Normal Stopping process
  rem ***************************************
  tasklist /FI "IMAGENAME eq AT_Admin.exe" 2>NUL | find /I /N "AT_Admin.exe">NUL
  if "%ERRORLEVEL%"=="0" taskkill /IM AT_Admin.exe /F
  
  tasklist /FI "IMAGENAME eq AT_Archive.exe" 2>NUL | find /I /N "AT_Archive.exe">NUL
  if "%ERRORLEVEL%"=="0" taskkill /IM AT_Archive.exe /F
  
  tasklist /FI "IMAGENAME eq AT_Setup.exe" 2>NUL | find /I /N "AT_Setup.exe">NUL
  if "%ERRORLEVEL%"=="0" taskkill /IM AT_Setup.exe /F
  
  tasklist /FI "IMAGENAME eq AT_Diag.exe" 2>NUL | find /I /N "AT_Diag.exe">NUL
  if "%ERRORLEVEL%"=="0" taskkill /IM AT_Diag.exe /F
  
  tasklist /FI "IMAGENAME eq AT_VpimAdmin.exe" 2>NUL | find /I /N "AT_VpimAdmin.exe">NUL
  if "%ERRORLEVEL%"=="0" taskkill /IM AT_VpimAdmin.exe /F
  
  tasklist /FI "IMAGENAME eq AT_LineStatus.exe" 2>NUL | find /I /N "AT_LineStatus.exe">NUL
  if "%ERRORLEVEL%"=="0" taskkill /IM AT_LineStatus.exe /F
  
  tasklist /FI "IMAGENAME eq AT_SoftwareUpdate.exe" 2>NUL | find /I /N "AT_SoftwareUpdate.exe">NUL
  if "%ERRORLEVEL%"=="0" taskkill /IM AT_SoftwareUpdate.exe /F
  
  tasklist /FI "IMAGENAME eq AT_Report.exe" 2>NUL | find /I /N "AT_Report.exe">NUL
  if "%ERRORLEVEL%"=="0" taskkill /IM AT_Report.exe /F
  
  tasklist /FI "IMAGENAME eq AT_SystemStatus.exe" 2>NUL | find /I /N "AT_SystemStatus.exe">NUL
  if "%ERRORLEVEL%"=="0" taskkill /IM AT_SystemStatus.exe /F
  
  tasklist /FI "IMAGENAME eq UCConnectWizardVB.NET.exe" 2>NUL | find /I /N "UCConnectWizardVB.NET.exe">NUL
  if "%ERRORLEVEL%"=="0" taskkill /IM UCConnectWizardVB.NET.exe /F
  
  tasklist /FI "IMAGENAME eq TeamQScriptConfig.exe" 2>NUL | find /I /N "TeamQScriptConfig.exe">NUL
  if "%ERRORLEVEL%"=="0" taskkill /IM TeamQScriptConfig.exe /F
  armkill CXUS  /T 0
  armkill CX  /T 0
  armkill CXTM /T 0
  armkill CXSS /T 0
  armkill CXGS /T 0
  armkill CXGDS /T 0
  armkill CXFM /T 0
  armkill CXCS /T 0
  armkill CXDS /T 0
  armkill CXLS /T 0
  armkill CXBS /T 0
  armkill SLM /T 0
  armkill NSS /T 0
  armkill NRS /T 0
  armkill MYSQL /T 0
  armkill PERLE /T 0
  RMDIR /S /Q D:\CX\SentinelCloudUsage
  MKDIR D:\CX\SentinelCloudUsage
  ```

The `armload` in the 9th line of the start.bat means:

- armload command starts `MySQLCore` service.
- `MYSQL` is "watchID" that armkill command uses to stop the service `MySQLCore`.
- `/S` specifies that the string `MySQLCore` being controlled is a Windows service.
- `/A` specifies that the service `MySQLCore` is monitored even if it is already started.
  Without `/A`, the starting operation for the running service fails with "it already running".
- `/R 3` means that armload command monitors the specified service `MySQLCore` and restarts them if it detects an abnormality. If restarting does not return the service to the `Running` state, it will attempt to restart the service up to the specified number of times (3 times).

The 22nd line of the start.bat has a parameter `/WAIT 0` which specifies the time to wait for the completion of a service startup by the second. When this parameter is specified, this command does not return controls while waiting for the service to complete startup (SERVICE_RUNNING) or within the wait time.
The value from 0 through 3600 can be specified. When 0 is specified, the command waits for the completion of a startup endlessly.

Note that the order of the lines specifies the starting order of the services.
`NuanceSpeechServer` service is started after `Nuance Recognition Service` which is started after `MySQLCore` service.

## Edit clp.conf with `clpcfadm` command

Open `cmd.exe` On a member node of the older cluster (ECX3), and export cluster configuration files on the current directory by `clpcfctrl` command.

```bat
clpcfctrl --pull -x .
```

Zip the `clp.conf` file into `clp-old.zip` and copy `scripts` directory into the `clp-old.zip`.

On the WebUI of the newer cluster (ECX4 or later), import the `clp-old.zip` and export the configuration and name it as `clp-new.zip`.

Extract `clp.conf` from the `clp-new.zip` to the current directory.

Open `cmd.exe`.
Change directory to where the `clp.conf` was extracted.
Issue `clpcfadm.py` command as follows to edit the clp.conf in the current directory.

Assumption:

- Python 3.6.8 or later is installed on the (client) host where WebUI runs, and is configured to run on cmd.exe.
- The name of the failover group is `failover`.
- Setting "Stopping the cluster service and rebooting the OS" for if a failure occurred on starting or stopping services. It could be good for major systems.

```bat
REM Service resources

clpcfadm.py add rsc failover service service-PERLE
clpcfadm.py add rsc failover service service-MYSQL
clpcfadm.py add rsc failover service service-NRS
clpcfadm.py add rsc failover service service-NSS
clpcfadm.py add rsc failover service service-SLM
clpcfadm.py add rsc failover service service-CXBS
clpcfadm.py add rsc failover service service-CXLS
clpcfadm.py add rsc failover service service-CXDS
clpcfadm.py add rsc failover service service-CXCS
clpcfadm.py add rsc failover service service-CXFM
clpcfadm.py add rsc failover service service-CXGDS
clpcfadm.py add rsc failover service service-CXGS
clpcfadm.py add rsc failover service service-CXSS
clpcfadm.py add rsc failover service service-CXTM
clpcfadm.py add rsc failover service service-CX
clpcfadm.py add rsc failover service service-CXUS

clpcfadm.py mod -t resource/service@service-PERLE/parameters/name -set "TruePortSrv"
clpcfadm.py mod -t resource/service@service-MYSQL/parameters/name -set "MySQLCore"
clpcfadm.py mod -t resource/service@service-NRS/parameters/name -set "Nuance Recognition Service"
clpcfadm.py mod -t resource/service@service-NSS/parameters/name -set "NuanceSpeechServer"
clpcfadm.py mod -t resource/service@service-SLM/parameters/name -set "hasplms"
clpcfadm.py mod -t resource/service@service-CXBS/parameters/name -set "AT_GenBackground"
clpcfadm.py mod -t resource/service@service-CXLS/parameters/name -set "AT_LicServer"
clpcfadm.py mod -t resource/service@service-CXDS/parameters/name -set "AT_DAL"
clpcfadm.py mod -t resource/service@service-CXCS/parameters/name -set "CDAUSServer"
clpcfadm.py mod -t resource/service@service-CXFM/parameters/name -set "AT_FileManager"
clpcfadm.py mod -t resource/service@service-CXGDS/parameters/name -set "CXGData"
clpcfadm.py mod -t resource/service@service-CXGS/parameters/name -set "AT_GrammarServer"
clpcfadm.py mod -t resource/service@service-CXSS/parameters/name -set "AT_SOAPServer"
clpcfadm.py mod -t resource/service@service-CXTM/parameters/name -set "AT_TextMessaging"
clpcfadm.py mod -t resource/service@service-CX/parameters/name -set "Launcher"
clpcfadm.py mod -t resource/service@service-CXUS/parameters/name -set "UCConnect Server"

clpcfadm.py mod -t resource/service@service-PERLE/parameters/started -set 1
clpcfadm.py mod -t resource/service@service-MYSQL/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-NRS/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-NSS/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-SLM/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-CXBS/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-CXLS/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-CXDS/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-CXCS/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-CXFM/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-CXGDS/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-CXGS/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-CXSS/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-CXTM/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-CX/parameters/started --set 1
clpcfadm.py mod -t resource/service@service-CXUS/parameters/started --set 1

clpcfadm.py mod -t resource/service@service-PERLE/deact/action --set 5
clpcfadm.py mod -t resource/service@service-MYSQL/deact/action --set 5
clpcfadm.py mod -t resource/service@service-NRS/deact/action --set 5
clpcfadm.py mod -t resource/service@service-NSS/deact/action --set 5
clpcfadm.py mod -t resource/service@service-SLM/deact/action --set 5
clpcfadm.py mod -t resource/service@service-CXBS/deact/action --set 5
clpcfadm.py mod -t resource/service@service-CXLS/deact/action --set 5
clpcfadm.py mod -t resource/service@service-CXDS/deact/action --set 5
clpcfadm.py mod -t resource/service@service-CXCS/deact/action --set 5
clpcfadm.py mod -t resource/service@service-CXFM/deact/action --set 5
clpcfadm.py mod -t resource/service@service-CXGDS/deact/action --set 5
clpcfadm.py mod -t resource/service@service-CXGS/deact/action --set 5
clpcfadm.py mod -t resource/service@service-CXSS/deact/action --set 5
clpcfadm.py mod -t resource/service@service-CXTM/deact/action --set 5
clpcfadm.py mod -t resource/service@service-CX/deact/action --set 5
clpcfadm.py mod -t resource/service@service-CXUS/deact/action --set 5

clpcfadm.py add rscdep service service-PERLE "UM8700 Services Script"
clpcfadm.py add rscdep service service-MYSQL service-PERLE
clpcfadm.py add rscdep service service-NRS service-MYSQL
clpcfadm.py add rscdep service service-NSS service-NRS
clpcfadm.py add rscdep service service-SLM service-NSS
clpcfadm.py add rscdep service service-CXBS service-SLM
clpcfadm.py add rscdep service service-CXLS service-CXBS
clpcfadm.py add rscdep service service-CXDS service-CXLS
clpcfadm.py add rscdep service service-CXCS service-CXDS
clpcfadm.py add rscdep service service-CXFM service-CXCS
clpcfadm.py add rscdep service service-CXGDS service-CXFM
clpcfadm.py add rscdep service service-CXGS service-CXGDS
clpcfadm.py add rscdep service service-CXSS service-CXGS
clpcfadm.py add rscdep service service-CXTM service-CXSS
clpcfadm.py add rscdep service service-CX service-CXTM
clpcfadm.py add rscdep service service-CXUS service-CX

REM Service Monitor resources

clpcfadm.py add mon servicew servicew-PERLE
clpcfadm.py add mon servicew servicew-MYSQL
clpcfadm.py add mon servicew servicew-NRS
clpcfadm.py add mon servicew servicew-NSS
clpcfadm.py add mon servicew servicew-SLM
clpcfadm.py add mon servicew servicew-CXBS
clpcfadm.py add mon servicew servicew-CXLS
clpcfadm.py add mon servicew servicew-CXDS
clpcfadm.py add mon servicew servicew-CXCS
clpcfadm.py add mon servicew servicew-CXFM
clpcfadm.py add mon servicew servicew-CXGDS
clpcfadm.py add mon servicew servicew-CXGS
clpcfadm.py add mon servicew servicew-CXSS
clpcfadm.py add mon servicew servicew-CXTM
clpcfadm.py add mon servicew servicew-CX
clpcfadm.py add mon servicew servicew-CXUS

clpcfadm.py mod -t monitor/servicew@servicew-PERLE/target --set service-PERLE
clpcfadm.py mod -t monitor/servicew@servicew-MYSQL/target --set service-MYSQL
clpcfadm.py mod -t monitor/servicew@servicew-NRS/target --set service-NRS
clpcfadm.py mod -t monitor/servicew@servicew-NSS/target --set service-NSS
clpcfadm.py mod -t monitor/servicew@servicew-SLM/target --set service-SLM
clpcfadm.py mod -t monitor/servicew@servicew-CXBS/target --set service-CXBS
clpcfadm.py mod -t monitor/servicew@servicew-CXLS/target --set service-CXLS
clpcfadm.py mod -t monitor/servicew@servicew-CXDS/target --set service-CXDS
clpcfadm.py mod -t monitor/servicew@servicew-CXCS/target --set service-CXCS
clpcfadm.py mod -t monitor/servicew@servicew-CXFM/target --set service-CXFM
clpcfadm.py mod -t monitor/servicew@servicew-CXGDS/target --set service-CXGDS
clpcfadm.py mod -t monitor/servicew@servicew-CXGS/target --set service-CXGS
clpcfadm.py mod -t monitor/servicew@servicew-CXSS/target --set service-CXSS
clpcfadm.py mod -t monitor/servicew@servicew-CXTM/target --set service-CXTM
clpcfadm.py mod -t monitor/servicew@servicew-CX/target --set service-CX
clpcfadm.py mod -t monitor/servicew@servicew-CXUS/target --set service-CXUS

clpcfadm.py mod -t monitor/servicew@servicew-PERLE/parameters/name --set "TruePortSrv"
clpcfadm.py mod -t monitor/servicew@servicew-MYSQL/parameters/name --set "MySQLCore"
clpcfadm.py mod -t monitor/servicew@servicew-NRS/parameters/name --set "Nuance Recognition Service"
clpcfadm.py mod -t monitor/servicew@servicew-NSS/parameters/name --set "NuanceSpeechServer"
clpcfadm.py mod -t monitor/servicew@servicew-SLM/parameters/name --set "hasplms"
clpcfadm.py mod -t monitor/servicew@servicew-CXBS/parameters/name --set "AT_GenBackground"
clpcfadm.py mod -t monitor/servicew@servicew-CXLS/parameters/name --set "AT_LicServer"
clpcfadm.py mod -t monitor/servicew@servicew-CXDS/parameters/name --set "AT_DAL"
clpcfadm.py mod -t monitor/servicew@servicew-CXCS/parameters/name --set "CDAUSServer"
clpcfadm.py mod -t monitor/servicew@servicew-CXFM/parameters/name --set "AT_FileManager"
clpcfadm.py mod -t monitor/servicew@servicew-CXGDS/parameters/name --set "CXGData"
clpcfadm.py mod -t monitor/servicew@servicew-CXGS/parameters/name --set "AT_GrammarServer"
clpcfadm.py mod -t monitor/servicew@servicew-CXSS/parameters/name --set "AT_SOAPServer"
clpcfadm.py mod -t monitor/servicew@servicew-CXTM/parameters/name --set "AT_TextMessaging"
clpcfadm.py mod -t monitor/servicew@servicew-CX/parameters/name --set "Launcher"
clpcfadm.py mod -t monitor/servicew@servicew-CXUS/parameters/name --set "UCConnect Server"

clpcfadm.py mod -t monitor/servicew@servicew-PERLE/relation/name --set service-PERLE --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-MYSQL/relation/name --set service-MYSQL --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-NRS/relation/name --set service-NRS --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-NSS/relation/name --set service-NSS --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-SLM/relation/name --set service-SLM --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXBS/relation/name --set service-CXBS --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXLS/relation/name --set service-CXLS --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXDS/relation/name --set service-CXDS --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXCS/relation/name --set service-CXCS --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXFM/relation/name --set service-CXFM --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXGDS/relation/name --set service-CXGDS --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXGS/relation/name --set service-CXGS --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXSS/relation/name --set service-CXSS --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXTM/relation/name --set service-CXTM --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CX/relation/name --set service-CX --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXUS/relation/name --set service-CXUS --nocheck

clpcfadm.py mod -t monitor/servicew@servicew-PERLE/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-MYSQL/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-NRS/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-NSS/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-SLM/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXBS/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXLS/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXDS/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXCS/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXFM/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXGDS/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXGS/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXSS/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXTM/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CX/relation/type --set rsc --nocheck
clpcfadm.py mod -t monitor/servicew@servicew-CXUS/relation/type --set rsc --nocheck
```

copy `clp.conf` in the current directory back to `clp-new.zip`.

On the WebUI of the newer cluster (ECX4 or later), import the `clp-new.zip`.
Edit the start.bat and stop.bat scripts in the script resource `UM8700 Services Script` to delete `armload` and `armkill` command lines.
Then apply the configuration.
