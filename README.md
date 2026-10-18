# Home SOC Lab — Catching a Phishing Attack from Click to Alert

I built a home security lab to see if I could catch a simulated attack the way a real SOC (Security Operations Center) analyst would — from the moment someone clicks a phishing link, all the way to an automated alert going off.

## What this project actually is

I used two computers: one acting as the "victim," and one acting as my security monitoring station. I sent a fake phishing email to the victim machine, clicked the link (on purpose, to simulate a real user falling for it), then ran real attack techniques that hackers actually use — things like credential-stealing tools and sneaky ways of running PowerShell commands. Then I went over to my monitoring machine and wrote a rule to catch that exact behavior, tested whether it also falsely flagged normal, everyday computer use, and finally set it up so it would alert me automatically instead of me having to go looking for trouble myself.

## Tools Used

| Tool | What it did in this project |
|---|---|
| **Splunk Enterprise** | The SIEM — collects and searches all the security logs (the "dashboard") |
| **Splunk Universal Forwarder** | Lightweight agent on the victim PC that ships its logs over to Splunk |
| **Sysmon** (SwiftOnSecurity config) | Gives much richer Windows logging — full command lines, network connections, registry changes |
| **GoPhish** | Sends and tracks the simulated phishing email and landing page |
| **Atomic Red Team** | Runs real, documented attack techniques (based on the MITRE ATT&CK framework) instead of made-up scripts |
| **Wireshark** | Captures raw network traffic, used to independently confirm the phishing click at the packet level |
| **osTicket**  | Ticketing system for the incident-response step |
| **Windows 10** (x2) | One victim PC, one host/monitoring PC, both on the same home network |

## How the pieces fit together

```
 [Victim PC]                          [My monitoring PC]
 ──────────                           ──────────────────
 Logs everything that happens          Splunk (the SIEM — think of
 on it (Sysmon) and ships those         it as the "security dashboard"
 logs over  ───────────────────►       that collects and searches logs)
 PowerShell attacks run here            GoPhish (sends the fake phishing
         │                              email and tracks who clicked)
         │  victim clicks the link
         └─────────────────────────────►│
                                                │
                                     My detection rule checks the logs
                                                │
                                     Alert fires every 5 minutes if something matches
```

## Why I picked the tools I did

I originally tried to use a tool called Wazuh, but it needed a virtual machine, and the networking on my Wi-Fi kept blocking it — no matter what I tried, the virtual machine couldn't talk to the internet properly. After burning a lot of time on that, I switched to Splunk instead, since it installs directly on Windows with no virtual machine needed at all. That ended up being the right call.

I also used a tool called Sysmon, because Windows' built-in logging is honestly pretty weak on its own — it doesn't show you the full command someone typed, or what they connected to online. Sysmon fills in that gap.

For the phishing part, I used GoPhish because it's free, I can run it myself, and it tracks exactly who clicked the link and when — which real phishing-simulation tools companies use also do.

For the actual "attack," I used something called Atomic Red Team, which runs real, well-documented attack techniques (the same ones listed in a framework called MITRE ATT&CK that most security teams reference) instead of me just making something up myself.

Finally, I used Wireshark to double check everything at the network level — basically watching the raw traffic between the two machines, so I had a second, independent way of confirming the attack actually happened, separate from what my logs said.

## What I actually found

My detection rule — essentially a search I wrote that looks for suspicious patterns in PowerShell commands — successfully caught over 20 different malicious things during my test, including:
- Attempts to steal saved passwords (a tool called Mimikatz)
- Attempts to map out a fake company network (AD recon tools)
- Sneaky ways of downloading and running code without saving a file first
- Commands disguised/scrambled to look less obviously suspicious

Just as important: I also tested my rule against totally normal, harmless commands — stuff like checking what time it is or listing files — and it correctly caught **none** of those. That matters because a detection rule that goes off on normal activity all the time is actually a problem in real security teams; people start ignoring it.

I also lined up the timing: the phishing click I saw in Wireshark happened right before the PowerShell attack I saw in Splunk — so I could show the whole story connects together, not just random separate pieces.

## The actual search I wrote (for anyone technical)

```spl
index=main source="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
Image="C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"
| regex CommandLine="(?i)(IEX|Invoke-Expression|-enc|-EncodedCommand|-nop|-NoProfile|DownloadString|AtomicRedTeam|SOAPHound|mimikatz|sekurlsa)"
| table _time, host, Image, CommandLine
```

I saved this as an alert that automatically checks every 5 minutes and flags anything it finds as high severity.

## What's in this folder

| Folder | What's in it |
|---|---|
| [`01-lab-setup/`](./01-lab-setup) | How I set everything up — the diagram, configs, and proof the two machines could actually talk to each other |
| [`02-attack-simulation/`](./02-attack-simulation) | The phishing campaign results, the attack commands I ran, and the network capture of the click |
| [`03-detection/`](./03-detection) | My actual detection rule and the test results (both the "it worked" test and the "it doesn't false-alarm" test) |
| [`04-response/`](./04-response) | Proof the alert actually fired, plus what I'd do next if this were a real incident |
| [`evidence/`](./evidence) | All my raw screenshots and exports |
| [`05-lessons-learned.md`](./05-lessons-learned.md) | What went wrong, what I'd do differently next time |
| [`IOCs.md`](./IOCs.md) | A list of the specific suspicious patterns/commands this lab generated |

## Things I learned the hard way

- Wazuh's virtual machine kept failing to get online because of how my Wi-Fi network was set up — I switched to Splunk instead of fighting it further.
- My logging agent (the Universal Forwarder) was running with permissions that were too low, so it silently couldn't read one of my log sources. I had to bump its permissions up to fix it.
- One of my config files was set to keep logs in a raw format that made them basically unsearchable — a single setting change fixed it.
- At one point I ran a real credential-stealing tool (not a fake/simulated version) on a machine that still had my actual email and password hash on it. In hindsight, that kind of test should only ever happen on a totally throwaway machine with nothing real on it — lesson learned.

More detail on all of this is in [`05-lessons-learned.md`](./05-lessons-learned.md).

