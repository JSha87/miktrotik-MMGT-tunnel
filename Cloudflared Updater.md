# ===== cf-update: rolling refresh of cloudflared containers (RouterOS 7.24.5) =====
# Pass 1: missing/failed/not-running containers (no quorum needed, they aren't serving).
# Pass 2: running containers, one at a time, behind a quorum check.
# Health = running + tunnel HA connections >= minConns (read from cloudflared /metrics).
# Each container's comment records the real cloudflared version: cfver=<version>.
# Optional globals: cfRepairOnly=true (skip pass 2), cfDry=true (log only, change nothing).

:global cfNotify do={
    :log error $1
    # optional: :do { /tool fetch url="https://YOUR_WEBHOOK" http-method=post http-data=$1 keep-result=no } on-error={}
}
:global cfRunning do={
    :return ([:len [/container find where name=$1 running]] > 0)
}
:global cfFailed do={
    :local f false
    :do { :set f ([:len [/container find where name=$1 failed]] > 0) } on-error={}
    :return $f
}
:global cfWaitStopped do={
    # $1 name, $2 timeout seconds
    :global cfRunning
    :local t 0
    :while ($t < $2) do={
        :if (![$cfRunning $1]) do={ :return true }
        :delay 2s
        :set t ($t + 2)
    }
    :return false
}
:global cfWaitGone do={
    # $1 name, $2 timeout seconds
    :local t 0
    :while ($t < $2) do={
        :if ([:len [/container find where name=$1]] = 0) do={ :return true }
        :delay 2s
        :set t ($t + 2)
    }
    :return false
}
:global cfStable do={
    # $1 name, $2 seconds it must stay running
    :global cfRunning
    :global cfFailed
    :local t 0
    :while ($t < $2) do={
        :if (![$cfRunning $1]) do={ :return false }
        :if ([$cfFailed $1]) do={ :return false }
        :delay 1s
        :set t ($t + 1)
    }
    :return true
}
:global cfHaConns do={
    # $1 ip, $2 metrics port -> active HA connections (0 on any error)
    :do {
        :local r [/tool fetch url=("http://" . $1 . ":" . $2 . "/metrics") output=user as-value]
        :local d ($r->"data")
        :local k "\ncloudflared_tunnel_ha_connections "
        :local p [:find $d $k]
        :if ([:typeof $p] = "nil") do={ :return 0 }
        :local s ($p + [:len $k])
        :local e [:find $d "\n" $s]
        :return [:tonum [:pick $d $s $e]]
    } on-error={ :return 0 }
}
:global cfVersion do={
    # $1 ip, $2 metrics port -> real cloudflared version from build_info ("" on any error)
    :do {
        :local r [/tool fetch url=("http://" . $1 . ":" . $2 . "/metrics") output=user as-value]
        :local d ($r->"data")
        :local p [:find $d "\nbuild_info{"]
        :if ([:typeof $p] = "nil") do={ :return "" }
        :local e [:find $d "\n" ($p + 1)]
        :local line [:pick $d ($p + 1) $e]
        :local vk ",version=\""
        :local vp [:find $line $vk]
        :if ([:typeof $vp] = "nil") do={ :return "" }
        :local vs ($vp + [:len $vk])
        :local ve [:find $line "\"" $vs]
        :return [:pick $line $vs $ve]
    } on-error={ :return "" }
}
:global cfCountOthersUp do={
    # $1 name=port map, $2 veth ip, $3 name to exclude, $4 min HA connections
    :global cfRunning
    :global cfHaConns
    :local c 0
    :foreach k,v in=$1 do={
        :if (($k != $3) && ([$cfRunning $k])) do={
            :local id [/container find where name=$k]
            :local cm [/container get $id comment]
            :if ([:find $cm "cfver="] = 0) do={
                :if ([$cfHaConns $2 $v] >= $4) do={ :set c ($c + 1) }
            } else={
                # legacy container (no known metrics port yet): running counts
                :set c ($c + 1)
            }
        }
    }
    :return $c
}
:global cfDeploy do={
    # named args: cn image ver veth env root cmd stable ip port conns
    :global cfRunning
    :global cfFailed
    :global cfWaitStopped
    :global cfWaitGone
    :global cfStable
    :global cfHaConns
    :global cfVersion
    :global cfDry
    :if ($cfDry = true) do={ :log info ("cf DRY: would deploy " . $cn . " at " . $ver); :return true }
    :local id [/container find where name=$cn]
    :if ([:len $id] > 0) do={
        # stop auto-restart from fighting the stop, then stop and let it settle
        :do { /container set $id restart-policy=no } on-error={}
        :do { /container stop $id } on-error={}
        :if (![$cfWaitStopped $cn 90]) do={
            :log error ("cf: " . $cn . " did not stop")
            :do { /container set [/container find where name=$cn] restart-policy=always } on-error={}
            :return false
        }
        :delay 10s
        :local gone false
        :local tries 0
        :while ((!$gone) && ($tries < 6)) do={
            :do { /container remove [/container find where name=$cn] } on-error={}
            :if ([$cfWaitGone $cn 10]) do={ :set gone true } else={ :set tries ($tries + 1); :delay 5s }
        }
        :if (!$gone) do={
            :log error ("cf: could not remove " . $cn . " (running=" . [:tostr [$cfRunning $cn]] . " failed=" . [:tostr [$cfFailed $cn]] . ")")
            :do { /container set [/container find where name=$cn] restart-policy=always; /container start [/container find where name=$cn] } on-error={}
            :return false
        }
    }
    :do {
        /container add name=$cn remote-image=($image . ":" . $ver) interface=$veth envlist=$env root-dir=($root . "/" . $cn) cmd=$cmd comment=("cfver=" . $ver) start-on-boot=yes restart-policy=always logging=yes
    } on-error={ :log error ("cf: add failed for " . $cn); :return false }
    # no reliable "extracting" flag: retry start until running, failed, or timeout
    :local t 0
    :while (($t < 300) && (![$cfRunning $cn]) && (![$cfFailed $cn])) do={
        :do { /container start [/container find where name=$cn] } on-error={}
        :delay 5s
        :set t ($t + 5)
    }
    :if (![$cfRunning $cn]) do={ :log error ("cf: " . $cn . " did not start (extract failed or timeout)"); :return false }
    :if (![$cfStable $cn $stable]) do={ :log error ("cf: " . $cn . " unstable after start"); :return false }
    :local c 0
    :local up false
    :while (($c < 120) && (!$up)) do={
        :if ([$cfHaConns $ip $port] >= $conns) do={ :set up true } else={ :delay 5s; :set c ($c + 5) }
    }
    :if (!$up) do={ :log error ("cf: " . $cn . " running but no tunnel connections"); :return false }
    # stamp the real version into the comment
    :local actual [$cfVersion $ip $port]
    :if ([:len $actual] > 0) do={
        :do { /container set [/container find where name=$cn] comment=("cfver=" . $actual) } on-error={}
    }
    :return true
}

# ---------------- CONFIG ----------------
:local selfName "cloudflared-updater"
:local cNs {"cloudflared-tunnel-1"=21001; "cloudflared-tunnel-2"=21002; "cloudflared-tunnel-3"=21003}
:local img "cloudflare/cloudflared"
:local ver "latest"
:local vInt "veth1"
:local vethIp "192.168.10.200"
:local eL "cloudflared"
:local dP "usb1/container"
:local stableSecs 45
:local minOthers 2
:local minConns 1

# ---------------- MAIN ----------------
:global cfRunning
:global cfNotify
:global cfDeploy
:global cfHaConns
:global cfVersion
:global cfCountOthersUp
:global cfRepairOnly
:global cfDry

# refuse to run if another instance of this script is already running (stale-lock proof)
:local others 0
:do { :set others ([:len [/system script job find where script=$selfName]] - 1) } on-error={}
:if ($others > 0) do={ :log warning "cf-update: another instance is running; exiting"; :error "already running" }

:log info ("cf-update: start (version=" . $ver . ", repairOnly=" . [:tostr $cfRepairOnly] . ", dry=" . [:tostr $cfDry] . ")")

:do {
    # --- internet + DNS ---
    :local up false
    :local t 0
    :while ((!$up) && ($t < 300)) do={
        :if ([/ping 1.1.1.1 count=3] > 0) do={ :do { :resolve cloudflare.com; :set up true } on-error={} }
        :if (!$up) do={ :delay 10s; :set t ($t + 10) }
    }
    :if (!$up) do={ $cfNotify "cf-update: no internet/DNS, nothing changed"; :error "no internet" }

    # --- pass "down": missing, failed or not running (safe to fix anytime)
    # --- pass "up": running containers, one at a time behind a quorum check
    :foreach pass in={"down"; "up"} do={
        :foreach n,port in=$cNs do={
            :local isUp [$cfRunning $n]
            :if ((($pass = "down") && (!$isUp)) || (($pass = "up") && $isUp && ($cfRepairOnly != true))) do={
                :local cmdStr ("tunnel --no-autoupdate --metrics 0.0.0.0:" . $port . " run")
                :local id [/container find where name=$n]
                :local cur ""
                :if ([:len $id] > 0) do={ :set cur [/container get $id comment] }
                :local prev ""
                :if ([:find $cur "cfver="] = 0) do={ :set prev [:pick $cur 6 [:len $cur]] }

                :if (($prev = $ver) && $isUp && ($ver != "latest")) do={
                    :if ([$cfHaConns $vethIp $port] < $minConns) do={
                        :log warning ("cf-update: " . $n . " at " . $ver . " but has no tunnel connections")
                    } else={
                        :log info ("cf-update: " . $n . " already at " . $ver)
                    }
                } else={
                    :if ($isUp) do={
                        # taking a serving replica down: require enough healthy others
                        :local w 0
                        :while ((([$cfCountOthersUp $cNs $vethIp $n $minConns]) < $minOthers) && ($w < 600)) do={ :delay 10s; :set w ($w + 10) }
                        :if (([$cfCountOthersUp $cNs $vethIp $n $minConns]) < $minOthers) do={
                            $cfNotify ("cf-update: other replicas not healthy; refusing to touch " . $n)
                            :error "quorum"
                        }
                    }
                    :if ($isUp) do={ :log info ("cf-update: updating " . $n . " [" . $prev . "] -> " . $ver) } else={ :log info ("cf-update: repairing missing/failed " . $n . " -> " . $ver) }
                    :local ok [$cfDeploy cn=$n image=$img ver=$ver veth=$vInt env=$eL root=$dP cmd=$cmdStr stable=$stableSecs ip=$vethIp port=$port conns=$minConns]
                    :if (!$ok) do={
                        :if (([:len $prev] > 0) && ($prev != $ver) && ($prev != "latest")) do={
                            :log warning ("cf-update: rolling " . $n . " back to " . $prev)
                            :if ([$cfDeploy cn=$n image=$img ver=$prev veth=$vInt env=$eL root=$dP cmd=$cmdStr stable=$stableSecs ip=$vethIp port=$port conns=$minConns]) do={
                                $cfNotify ("cf-update: " . $n . " failed on " . $ver . ", rolled back to " . $prev)
                            } else={
                                $cfNotify ("cf-update: ROLLBACK FAILED, " . $n . " is down")
                            }
                        } else={
                            $cfNotify ("cf-update: " . $n . " failed on " . $ver . ", no previous version to roll back to")
                        }
                        :error "deploy failed"
                    }
                    :log info ("cf-update: " . $n . " healthy at version " . [$cfVersion $vethIp $port])
                }
            }
        }
    }
    :log info "cf-update: complete"
} on-error={
    :log warning "cf-update: aborted (see earlier cf-update log lines)"
}
