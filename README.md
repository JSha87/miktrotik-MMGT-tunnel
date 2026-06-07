# Pre-requisites:

## 1. On RouterOS 7.5 +
### https://mikrotik.com/download
## 2. Container environment setup
### https://unixhost.pro/blog/2022/11/setup-docker-container-on-routeros-mikrotik/
## 3. Paste scripts and triggers from listed Feature into terminal 
## 4. Confirm scripts and triggers have been created:
```
/system script export
```
```
/system scheduler export
```
---
# Updating RouterOS manually

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
