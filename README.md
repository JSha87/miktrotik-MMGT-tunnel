#Pre-requisites:

## 1. On RouterOS 7.5 +
### https://mikrotik.com/download
## 2. Container environment setup
### https://unixhost.pro/blog/2022/11/setup-docker-container-on-routeros-mikrotik/
## 3. Container "envlist" with key TOKEN_TOKEN defined by Cloudflared Token
## 4. Paste script into System/scripts
5. 
## 5. Create Scheduled Task in System/Scheduler to trigger script
ie:
```
Name: Containers fix on boot
Start Time: startup
OnEvent: /system/script/run cloudflared-updater
```
```
Name: update-cloudflared
Start Time: 03:00:00
Interval: 1d 00:00:00
OnEvent: /system/script/run cloudflared-updater
```
### For updating manually without too much downtime

## 1. Check and download OS update
```
/system package update check-for-updates
/system package update download
```
## 2. Install and reboot (This will drop your connection)
```
/system package update install
```
## 3. AFTER REBOOT: Upgrade hardware firmware
```
/system routerboard upgrade
/system reboot
```
