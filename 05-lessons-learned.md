# Lessons Learned

Things that went wrong during this project, what caused them, and what I'd do differently.

## Wazuh → Splunk pivot

I originally planned to use Wazuh as the SIEM, running inside a VirtualBox VM. The VM kept failing to get network access — stuck on NAT mode at `10.0.2.15`, and switching to a bridged adapter also failed because the home Wi-Fi network (college/institutional-style network) blocked the kind of networking VirtualBox needed.

After burning significant time trying to fix the VM's networking, I switched to **Splunk Enterprise**, which installs natively on Windows with no VM required at all. This ended up being the right call — not just a workaround, but genuinely simpler for a two-machine home lab setup.

**Takeaway:** when a tool's infrastructure requirement (in this case, a VM) fights the environment it's running in, it's worth stepping back and asking whether a different tool sidesteps the problem entirely, rather than continuing to debug the environment.

## Universal Forwarder running with insufficient permissions

After configuring `inputs.conf` to collect Sysmon logs, the data wasn't showing up in Splunk. The forwarder service (`SplunkForwarder`) was running as the low-privilege virtual account `NT SERVICE\SplunkForwarder`, which failed to subscribe to the Sysmon event channel (error code 5 — Access Denied), but failed silently with no obvious error in the Splunk UI.

**Fix:**
```
sc.exe config SplunkForwarder obj= LocalSystem
```
followed by a service restart.

**Takeaway:** when a data source should be flowing but isn't, check the permissions of the service doing the collecting, not just the configuration file — a config can be perfectly correct and still fail if the process reading it doesn't have access to the underlying resource.

## `renderXml = true` hid the EventCode field

Even after the forwarder was reading the Sysmon log correctly, searching `EventCode=1` in Splunk returned 0 results. The cause was `renderXml = true` in `inputs.conf` — this ships the entire Windows Event Log entry as a single raw XML blob, so fields like `EventCode` are buried inside the XML text instead of being extracted as a separate, searchable Splunk field.

**Fix:** set `renderXml = false`, restart the forwarder.

**Takeaway:** "the data is arriving but my search returns nothing" is a different problem from "the data isn't arriving at all" — worth checking field extraction settings specifically before assuming a connectivity issue.

## Real-time search window missed historical events

When testing the detection rule, a search using a real-time time range (`earliest=rt-30m&latest=rt`) returned "No results in current time range" even though matching events existed from earlier test runs. Real-time search windows only watch forward from the moment the search starts; they don't reliably surface already-indexed historical events.

**Fix:** switch to a standard historical time range (e.g. "Last 24 hours" or "All time") instead of a real-time window when investigating past activity.

## Running genuine Mimikatz on a machine with real credentials

When running the full Atomic Red Team T1059.001 batch with `-GetPrereqs`, the real (non-simulated) Mimikatz binary executed successfully — `sekurlsa::logonpasswords` extracted a genuine NTLM hash tied to my actual Microsoft account email. This happened because the "victim" PC still had my real account logged in, rather than being a fully disposable test machine.

**What I did about it:** did not publish the raw output anywhere, and treated it as a precautionary reason to review the account's password strength.

**Takeaway:** credential-dumping tests should only ever be run on a machine with no real, personally-meaningful credentials present — a throwaway VM with a disposable local account, not a daily-use PC repurposed as the "victim." This is a lab hygiene practice I'd apply strictly next time, even in a home lab with low real-world risk.

## WSL2/Docker blocker for osTicket

Setting up osTicket via Docker Desktop required WSL2, which in turn required the `Microsoft-Windows-Subsystem-Linux` and `VirtualMachinePlatform` Windows features. Both kept reporting as **Disabled** even after enabling them via PowerShell, multiple restarts, and a full DISM health-repair sequence (CheckHealth → ScanHealth → RestoreHealth, all completing without finding corruption).

**Fix:** abandoned the Docker/WSL2 path and used **XAMPP** instead — a direct Apache/MySQL/PHP install with no virtualization layer required. osTicket was running within the hour once this pivot was made.

**Takeaway:** similar to the Wazuh situation — when a dependency chain (Docker → WSL2 → Windows feature flags) has multiple points of failure outside my direct control, a simpler, more direct alternative was worth switching to rather than continuing to debug each link in the chain.
