# Pre-Requisites
Ensure required interface(veth), vlan, and usb storage are correctly configured and make cloudflare token available as environment variable.
```
/container envs
add key=TUNNEL_TOKEN list=cloudflared value="YOUR_ACTUAL_TOKEN_HERE"
```
---
# Cloudflared Updater Script
```
/system script add dont-require-permissions=yes name="cloudflared-updater" owner="admin" policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon source={
    # --- PRE-CONDITION: WAIT FOR INTERNET ---
    :local maxWait 300; # 5 minutes max
    :local waitStep 10;
    :local elapsed 0;
    :local internetUp false;

    :put "Checking connectivity before processing containers..."

    :while ($elapsed < $maxWait && $internetUp = false) do={
        # We ping a public IP AND check if we can resolve a domain
        :if ([/ping 8.8.8.8 count=1] > 0) do={
            :do {
                :resolve google.com;
                :set internetUp true;
                :put "Internet is UP and DNS is working. Proceeding...";
            } on-error={
                :put "Ping OK, but DNS failed. Waiting...";
            }
        } else={
            :put "WAN is DOWN. Waiting $waitStep seconds... ($elapsed/$maxWait)";
        }
        
        :if ($internetUp = false) do={
            :delay $waitStep;
            :set elapsed ($elapsed + $waitStep);
        }
    }

    :if ($internetUp = false) do={
        :error "Aborting: Internet connection not established within $maxWait seconds.";
    }
    # --- CONFIGURATION ---
    :local cNs {"cloudflared-tunnel-1"; "cloudflared-tunnel-2"; "cloudflared-tunnel-3"}
    :local img "cloudflare/cloudflared:latest"
    :local vInt "veth1"
    :local eL "cloudflared"
    :local dP "usb1/container"
    :local cmd "tunnel --no-autoupdate run"

    :put "--- Phase 1: Global Health Check ---"
    :local repairList [:toarray ""]
    :local updateList [:toarray ""]

    :foreach n in=$cNs do={
        :local exists ([:len [/container find name=$n]] > 0)
        :local isRunning ([/container print count-only where name=$n running] > 0)
        
        :if (!$exists || !$isRunning) do={
            :set repairList ($repairList, $n)
        } else={
            :set updateList ($updateList, $n)
        }
    }

    :local masterList ($repairList, $updateList)
    :put "Queue order: Repairs first, then Updates."

    # --- Phase 2: Processing ---
    :foreach n in=$masterList do={
        :if ([:len $n] > 0) do={
            :put ("Current Target: " . $n)
            
            # 1. REMOVE
            :if ([:len [/container find name=$n]] > 0) do={
                :put "Stopping..."
                :do { /container stop [find name=$n] } on-error={}
                :while ([/container print count-only where name=$n stopped] = 0) do={ :delay 500ms }

                :put "Removing..."
                :local remSuccess false
                :while ($remSuccess = false) do={
                    :do { 
                        /container remove [find name=$n]
                        :set remSuccess true
                    } on-error={ :delay 1s }
                }
            }

            # 2. CREATE (start-on-boot changed to no)
            :put "Adding new container..."
            /container add name=$n remote-image=$img interface=$vInt envlist=$eL root-dir=($dP . "/" . $n) cmd=$cmd start-on-boot=no

            # 3. WAIT FOR EXTRACTION (E Flag)
            :put "Extracting (E flag)..."
            :while ([/container print count-only where name=$n extracting] > 0) do={ :delay 1s }

            # 4. START & LOCK (R Flag)
            :put "Starting..."
            /container start [find name=$n]
            
            # Script blocks here until this tunnel is fully Running
            :while ([/container print count-only where name=$n running] = 0) do={ :delay 1s }

            :put ("SUCCESS: " . $n . " is healthy and running.")
            :put "----------------------------"
        }
    }

    :put "--- All 3 Tunnels Updated & Verified ---"
}
```
---
# Schedulers
## Cloudflared Daily Updater
```
/system scheduler add name="update-cloudflared" on-event="/system/script/run cloudflared-updater" policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon start-date=2025-10-19 start-time=03:00:00 interval=1d
```
## Cloudflared Fix on Boot
```
/system scheduler add name="Containers fix on boot" on-event="/system/script/run cloudflared-updater" policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon start-time=startup
```
