# Stage 1: Hardware Firmware Check (Runs on boot)
```
/system script add name="AutoUpdate-Firmware" policy=reboot,read,write,policy,test,sensitive source={
    :delay 15s;
    :local currentFw [/system routerboard get current-firmware];
    :local upgradeFw [/system routerboard get upgrade-firmware];
    :if ($currentFw != $upgradeFw) do={
        :log info "Upgrading Routerboard firmware from $currentFw to $upgradeFw...";
        /system routerboard upgrade;
        :delay 5s;
        /system reboot;
    } else={
        :log info "Routerboard firmware is up to date.";
    }
}
```
# Stage 2: OS Update Check (Runs weekly)
```
/system script add name="AutoUpdate-OS" policy=reboot,read,write,policy,test,sensitive source={
    :log info "Checking for RouterOS updates...";
    /system package update set channel=stable;
    /system package update check-for-updates once;
    :delay 3s;
    :if ([/system package update get status] = "New version is available") do={
        :log info "New RouterOS version found. Downloading and installing...";
        /system package update install;
    } else={
        :log info "RouterOS is already up to date.";
    }
}
```
---
# Scheduled Triggers
```
/system scheduler add name="Schedule-FirmwareUpdate" on-event="/system/script/run AutoUpdate-Firmware" policy=reboot,read,write,policy,test,sensitive start-time=startup
```
```
/system scheduler add name="Schedule-OSUpdate" on-event="/system/script/run AutoUpdate-OS" policy=reboot,read,write,policy,test,sensitive start-date=jun/14/2026 start-time=03:00:00 interval=1w
```
---
