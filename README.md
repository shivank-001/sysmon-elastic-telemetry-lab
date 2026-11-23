# Sysmon + Elastic Stack Lab — Endpoint Logging and Noise Tuning

This is my first hands-on detection engineering project. I built a small home lab to answer one basic but important question: when a Windows computer runs, how do you actually capture what's happening on it in enough detail to catch something bad, and how do you stop that logging from drowning you in useless noise?

I'm a SOC Analyst with about 10 months of experience, so I already work with alerts and logs daily. But in my day job, someone else already built and tuned the pipeline that feeds my SIEM. This project was me building that pipeline myself, from a blank VM, so I actually understand what's happening before the dashboard shows up.

Nothing here touches a real company system. Everything was built and tested inside VMware on my own laptop.

---

## What is Sysmon, and why bother with it?

Windows already keeps its own logs (Event Viewer), but the built-in logging is thin. It'll tell you a process started, but often not much about what it actually did — what command line it ran with, what file it touched, what network connection it opened.

**Sysmon** (System Monitor) is a free tool from Microsoft Sysinternals that sits on a Windows machine and logs much richer detail: every process that starts (with its full command line and parent process), every file created, every registry change, every network connection, and more. It writes all of this into its own Windows Event Log channel, separate from the normal logs.

Download page: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

By itself, Sysmon with no configuration logs almost everything, which is way too much — a single Windows machine can generate tens of thousands of events a day. So nearly everyone who uses Sysmon pairs it with a **config file**, which is just an XML file telling Sysmon what to include and what to ignore. I used the config built by a well-known security researcher called SwiftOnSecurity, which is the de-facto community standard starting point:

https://github.com/SwiftOnSecurity/sysmon-config

That config already excludes a lot of harmless, repetitive Windows background noise (antivirus heartbeat, OS maintenance tasks, etc.) while keeping the stuff that actually matters for spotting attacks.

## What is Elasticsearch and Kibana, and why did I need them?

Sysmon only writes logs to the local machine. If that machine gets wiped, reimaged, or if you simply have more than one machine, you lose visibility or have to check each one by hand. A real SOC collects logs from every machine into one central place — that's a SIEM (Security Information and Event Management system).

I originally planned to use Wazuh (a popular free SIEM), but its installer refused to run because my Ubuntu VM didn't meet its minimum hardware requirement (it wants 4 GB RAM and 2 CPU cores, and my VM was a bit under that at the time). Rather than fight that, I switched to the **Elastic Stack**, which does the same basic job:

- **Elasticsearch** — the database that stores and indexes all the log data so it can be searched quickly
- **Kibana** — the web dashboard you use to search, filter, and visualize that data
- **Elastic Agent / Fleet** — the piece that actually runs on each machine, reads the Sysmon log channel, and ships the events over to Elasticsearch

Elastic download/docs: https://www.elastic.co/downloads

## The lab setup

**VM 1 — Windows 10 workstation**
Built in VMware Workstation. 4 GB RAM, 2 CPU cores, NAT networking (needed internet access to download Sysmon and the Elastic Agent). This is the machine being monitored.

**VM 2 — Ubuntu Server (the SIEM)**
Also built in VMware. Started at 15 GB disk / 2 GB RAM, which turned out to be too small once Elasticsearch and Kibana were both running, so I expanded the disk to 58 GB and the RAM to 4 GB (and later a bit more). This VM runs Elasticsearch, Kibana, and Fleet Server, and receives logs from the Windows VM.

I connected to the Ubuntu VM over SSH from Windows Terminal instead of using VMware's built-in console window, because copy-paste into a text-only server VM through the console is unreliable — SSH makes it a normal terminal where I can paste commands cleanly.

---

## Part 1 — Getting Sysmon running on Windows

After downloading Sysmon and the SwiftOnSecurity config file into the same folder, I opened PowerShell **as Administrator** (Sysmon needs this because it installs a kernel-level driver and a Windows service — a normal, non-admin PowerShell window will refuse to install it) and ran:

```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

What each part means:
- `-accepteula` skips the license pop-up, since Sysinternals tools normally ask you to click through an agreement the first time
- `-i sysmonconfig-export.xml` tells Sysmon to install itself using that specific config file, rather than its default (which, again, logs way too much)

Output when it worked:
```
Sysmon64 installed.
SysmonDrv installed.
Starting SysmonDrv.
SysmonDrv started.
Starting Sysmon64..
Sysmon64 started.
```

Checked the service was actually alive:
```powershell
Get-Service Sysmon64
```

Then I wanted a real number for how much Sysmon logs during normal, boring use — not a guess. I ran this right after install:
```powershell
(Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational").Count
```
That gave **41**. I then just used the VM normally for about 20 minutes — opened Notepad, browsed a couple of folders, opened Edge, ran `whoami` — and ran the same command again. It gave **67**. So roughly 26 events from 20 minutes of light, single-user activity, which is a genuinely low, manageable rate. I also broke the events down by type:
```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" | Group-Object Id | Sort-Object Count -Descending
```
Process Create (Event ID 1) was the largest category, which makes sense — nearly everything you do on a computer starts with launching some process.

---

## Part 2 — Fixing and expanding the Ubuntu VM

Before installing anything, the VM's disk was only 15 GB with barely 2 GB free, which is nowhere near enough for a SIEM. Growing a Linux disk inside VMware is a two-stage process: first you grow the virtual disk file itself in VMware's settings, and then you have to tell Linux to actually use that new space, because Linux doesn't automatically notice it.

```bash
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

In order: `growpart` extends the actual disk partition to use the new space VMware made available; `pvresize` tells LVM (the disk management layer Ubuntu uses by default) that the underlying physical volume is bigger now; `lvextend` grows the logical volume — think of this as the "usable container" — to take up all that newly freed space; and `resize2fs` finally grows the actual filesystem so Linux can write files into that new space. Skipping any one of these steps leaves you with wasted space you can see but can't use.

I checked the result with:
```bash
df -h /
```
which showed 58 GB total, about 47 GB free.

I also moved this VM's files off a folder that was syncing with OneDrive onto a plain folder on the D: drive, because a VM constantly writing and rewriting gigabytes of data inside a cloud-sync folder is both slow and a real corruption risk if OneDrive tries to sync mid-write.

---

## Part 3 — Getting Elasticsearch and Kibana actually running

Elasticsearch was already partially installed on this VM from an earlier attempt, but it wasn't starting. The systemd status only gave a vague "failed" message, so I checked the application's own log file directly:
```bash
sudo tail -n 50 /var/log/elasticsearch/elasticsearch.log
```
The real error was buried a few lines in:
```
Caused by: java.nio.file.AccessDeniedException: /etc/elasticsearch/service_tokens
```
Elasticsearch runs under its own system user account (also called `elasticsearch`), and that file was owned by `root` with permissions that locked everyone else out — including the very account that needed to read it. I fixed the ownership and permissions:
```bash
sudo chown elasticsearch:elasticsearch /etc/elasticsearch/service_tokens
sudo chmod 660 /etc/elasticsearch/service_tokens
sudo systemctl restart elasticsearch
```
After that it started cleanly.

Elasticsearch ships with security (login) turned on by default, and I didn't know the existing password, so I reset it:
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

Kibana was already running but stuck waiting, because it needs to be paired with Elasticsearch through a short-lived enrollment token and a verification code shown in the browser the first time you set it up:
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
sudo /usr/share/kibana/bin/kibana-verification-code
```
I opened `http://<ubuntu-vm-ip>:5601` in a browser on my actual laptop, pasted the token, entered the code, and logged in with the elastic account.

One more blocker came up after this: Kibana's Fleet feature (used for managing agents centrally) refused to initialize because Kibana was missing its own encryption keys — these are needed to securely store things like agent tokens. I generated them and added them to Kibana's config file:
```bash
sudo /usr/share/kibana/bin/kibana-encryption-keys generate
sudo tee -a /etc/kibana/kibana.yml > /dev/null <<'EOF'

xpack.encryptedSavedObjects.encryptionKey: <generated key 1>
xpack.reporting.encryptionKey: <generated key 2>
xpack.security.encryptionKey: <generated key 3>
EOF
sudo systemctl restart kibana
```
After that, Fleet loaded properly.

---

## Part 4 — Fleet Server and getting the Windows VM connected

Fleet Server is the piece that actually manages agents — without it, Elastic Agent can run "standalone" but Kibana can't see or centrally control it. I set up Fleet Server first on the Ubuntu VM itself, giving it the VM's own address:
```
URL: https://<ubuntu-vm-ip>:8220
```
Kibana then generated an install command containing a one-time service token and a certificate fingerprint, which I ran on the Ubuntu VM. It downloads the Elastic Agent package, unpacks it, and installs it specifically in Fleet Server mode. I had to double-check the file name matched my CPU architecture (x86_64, not arm64 — the default Kibana suggested was for arm64, which wouldn't run on my Ryzen laptop) and match the agent version to Fleet Server's own version, because an agent newer than the server it's enrolling into will fail.

Once Fleet Server showed as **Healthy** in Kibana's Fleet > Agents page, I repeated almost the same process for the Windows VM — downloaded the matching Elastic Agent zip for Windows, unzipped it, and ran:
```powershell
.\elastic-agent.exe install --url=https://<ubuntu-vm-ip>:8220 --enrollment-token=<token> --insecure
```
The `--insecure` flag is needed because Fleet Server's certificate is self-signed (it wasn't issued by a trusted certificate authority) — that's fine for a private lab, but it's something you'd fix properly before doing this in a real company environment.

To actually tell the agent what to collect, I added a **Custom Windows Event Logs** integration in Kibana, pointed at:
```
Microsoft-Windows-Sysmon/Operational
```
(Elastic doesn't have a dedicated "Sysmon" option for Windows specifically — on Windows, Sysmon's output is just another event log channel, so the generic Windows event log integration is the correct tool, pointed at that specific channel name.)

Once both agents showed Healthy, I confirmed data was actually arriving by searching in Kibana's Discover screen:
```
winlog.channel : "Microsoft-Windows-Sysmon/Operational"
```
and seeing live Sysmon events from the Windows VM.

---

## Part 5 — Finding the noise and actually tuning it

With data flowing for a couple of hours, I checked the overall count: **981 events**. Then I broke that down by event type to see what was eating the most volume:

- Event ID 11 (File Create): 336 events — 34% of everything
- Event ID 13 (Registry Value Set): 311 events — 32% of everything

That's two-thirds of all logging coming from just two event types, so these were the obvious places to look for noise. For Event ID 13 specifically, I checked which process was actually causing it, using the `winlog.event_data.Image` field in Kibana:

**CompatTelRunner.exe** — a background Windows process that runs scheduled telemetry and compatibility checks — was responsible for 46% of all Event ID 13 records, roughly 143 individual events, on its own.

I opened the Sysmon config file and added an exclusion for that process under the Registry exclude section (this took a couple of tries — I initially put the line in the wrong block, which actually told Sysmon to log it rather than ignore it, so I had to find and remove that mistake first). The final change was one line:
```xml
<Image condition="is">C:\Windows\system32\CompatTelRunner.exe</Image>
```
I reloaded the live config with:
```powershell
Sysmon64.exe -c "sysmonconfig-export.xml"
```
which printed `Configuration file validated.` and `Configuration updated.`

To actually prove the fix worked rather than just assuming it did, I forced CompatTelRunner to run again on demand:
```powershell
Start-Process "C:\Windows\system32\CompatTelRunner.exe" -ArgumentList "-m:appraiser.dll -f:DoScheduledTelemetryRun -cv:test" -WindowStyle Hidden -Wait
```
It ran for about 57 seconds, so it genuinely did real work rather than exiting instantly. I then searched Kibana for the same event again, for the time window after the fix:
```
winlog.channel : "Microsoft-Windows-Sysmon/Operational" and event.code : "13" and winlog.event_data.Image : "C:\\Windows\\system32\\compattelrunner.exe"
```

**Result: 0 hits.** Before the fix, this exact search returned 143. After the fix, even with the process forced to run manually, it produced nothing.

---

## The result, in one line

| | Before | After |
|---|---|---|
| CompatTelRunner registry-event (Event ID 13) hits | 143 | 0 |

The config change behind that result was a single added line.

---

## What I didn't fully finish (being honest about it)

- CompatTelRunner still appeared under Event ID 11 (File Create), even though an exclusion for it already existed in that section before I started tuning anything. I never went back and re-measured Event 11 specifically after my changes, so I'm not claiming that's fixed — it's an open question I'd dig into next.
- My "after" measurement came from one forced run of the process, not a full day of normal background activity. The number (0 hits) is real and verified, but a longer natural observation window would make the comparison stronger.
- `--insecure` was used to enroll the Windows agent because of the self-signed Fleet Server certificate. Acceptable for a lab, not something to carry into a real deployment without fixing properly.

## Problems I actually ran into

| Problem | What was really going on | How I fixed it |
|---|---|---|
| Wazuh installer refused to run | VM didn't meet its minimum hardware check (4 GB RAM / 2 CPU) | Switched to the Elastic Stack instead |
| Elasticsearch service wouldn't start | The `service_tokens` file was owned by root with permissions that blocked the elasticsearch user from reading it | Changed ownership and permissions on the file, restarted the service |
| Fleet wouldn't initialize in Kibana | Kibana had no encryption keys set in its config | Generated the keys and added them to `kibana.yml` |
| Elastic Agent install kept failing on Windows | Tried installing agent version 9.5.6, one version ahead of the 9.5.1 Fleet Server | Used the matching 9.5.1 agent package |
| Ubuntu VM running sluggish, risk of corruption | VM files were sitting inside a OneDrive-synced folder | Moved the whole VM folder to a plain D: drive location |

## Tools and links used

- Sysmon — https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
- SwiftOnSecurity Sysmon config — https://github.com/SwiftOnSecurity/sysmon-config
- Elastic Stack (Elasticsearch, Kibana, Fleet, Elastic Agent) — https://www.elastic.co/downloads
- VMware Workstation
- PowerShell / Windows Terminal (SSH client)

## What I'd do differently next time

- Check a VM's RAM and CPU against a tool's stated minimum requirements before starting the install, not after it fails partway through.
- Keep every VM's files off any cloud-sync folder from the very beginning, rather than fixing it mid-project.
- When tuning one noisy event type, re-check every other event type the same process shows up in, not just the one I was focused on at the time.
