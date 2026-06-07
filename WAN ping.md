# WAN Ping script
```
/system script add dont-require-permissions=no name="WANPing" owner="admin" policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon source={
    :local continue true;
    :local counter 0;
    :local maxcounter 12;
    :local sleepseconds 10;
    :local goodpings 0;

    :log error "Script will further test ping to WAN in $sleepseconds seconds and will continue for $maxcounter x $sleepseconds seconds.";

    :while ($continue) do={
        :set counter ($counter + 1);
        :delay $sleepseconds;
        
        :if ([/ping 8.8.8.8 interval=1 count=1] =0) do={
            :log warning "Ping to WAN failed on attempt $counter of $maxcounter - Will try again in $sleepseconds seconds.";
        } else={
            :log warning "Ping to WAN succeeded on attempt $counter of $maxcounter - No Further testing needed and script will exit.";
            :set continue false;
            :set goodpings ($goodpings + 1);
        };
        
        :if ($counter = $maxcounter) do={
            :set continue false;
        }
    }

    # --- THE FAIL ACTION ---
    :if ($goodpings = 0) do={
        :log error "All connection tests failed. Taking action..."
        
        # OPTION 1: Bounce the WAN interface (Recommended - Replace 'ether1' with your WAN name)
        # /interface disable ether1
        # :delay 5
        # /interface enable ether1
        
        # OPTION 2: The Nuclear Option (From your first script)
        /system reboot
    }
}
```
