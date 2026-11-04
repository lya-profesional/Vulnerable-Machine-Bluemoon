# System Hacking – Lab Assignment 2 (IKB21403 Vulnerability Analysis)

Penetration testing walkthrough of a vulnerable machine (**BlueMoon:2021**, target IP `192.168.0.13`) performed from a Kali Linux attacker box (`192.168.0.9`), covering reconnaissance, enumeration, exploitation, and privilege escalation.

> **Course:** IKB21403 – Vulnerability Analysis
> **Institution:** Universiti Kuala Lumpur – Malaysian Institute of Information Technology (UniKL MIIT)
> **Assignment:** Lab Assignment 2 (System Hacking)

## ⚠️ Disclaimer

This write-up documents work carried out **only** against a machine deliberately set up for this course, inside an isolated lab network. The techniques shown here must **never** be used against systems you do not own or do not have explicit written authorization to test. This repository is for educational purposes only.

## Objective

To identify, enumerate, and exploit vulnerabilities on a target machine in order to gain unauthorized (lab-authorized) access, escalate privileges to `root`, and demonstrate the full attack chain expected in a system hacking / penetration test exercise.

## Environment & Tools

| Tool | Purpose |
|---|---|
| Kali Linux | Attacker machine |
| `ifconfig` | Identify attacker's own network interface/IP |
| ARP scanner | Discover live hosts on the subnet |
| `nmap` | Host discovery and port/service enumeration |
| `gobuster` | Web directory brute-forcing |
| OWASP DirBuster | GUI-based web directory/file brute-forcing |
| `dirb` | Web content scanning with a custom wordlist |
| Web browser | Manual inspection of discovered pages |
| `nc` (Netcat) | Reverse shell listener |
| `md5sum` | Password hash generation/checking |
| Python (`pty` module) | TTY shell upgrade |
| `sudo`, `/etc/sudoers` | Privilege escalation |

## Methodology

### 1. Reconnaissance

Checked the attacker machine's own network configuration and confirmed the target was alive on the network.

![ifconfig](screenshots/01-ifconfig-attacker-ip.png)
![ARP scan](screenshots/02-arp-scan-host-discovery.png)
![nmap ping scan](screenshots/03-nmap-ping-scan.png)

- `ifconfig` showed the attacker IP as `192.168.0.9`.
- An ARP scan across `192.168.0.0/16` enumerated live hosts on the subnet, including the target at `192.168.0.13`.
- `nmap -sn 192.168.0.13` confirmed the host was up and resolved the hostname `bluemoon`.

### 2. Port & Service Enumeration

![nmap service scan](screenshots/04-nmap-service-scan.png)

Ran `nmap -sC -sV -Pn -vv 192.168.0.13`, revealing three open ports:

- `21/tcp` – FTP
- `22/tcp` – SSH
- `80/tcp` – HTTP

### 3. Web Enumeration

![gobuster](screenshots/05-gobuster-dir-enum.png)
![DirBuster setup](screenshots/06-dirbuster-setup.png)
![DirBuster results](screenshots/07-dirbuster-results.png)
![custom wordlist](screenshots/08-custom-wordlist-created.png)
![dirb scan](screenshots/09-dirb-scan-indexhtml-found.png)
![cat wordlist](screenshots/10-cat-wordlist.png)

- `gobuster dir -u http://192.168.0.13 --wordlist /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt` brute-forced directories.
- Cross-checked with the **OWASP DirBuster** GUI against the same target/wordlist, finding `/` and `/icons/`.
- Built a custom wordlist (`wordlist.txt`) seeded with a likely filename (`index`).
- `dirb http://192.168.0.13 ~/wordlist.txt -X .php,.txt,.html` uncovered a hidden page: **`index.html`** (HTTP 200).

### 4. Exploitation

![browser index.html](screenshots/11-browser-indexhtml-bluemoon.png)
![netcat listener](screenshots/12-netcat-listener.png)

- Browsed to `http://192.168.0.13/index.html`, revealing a **"BlueMoon:2021 – The Game Begins"** landing page, confirming the correct target.
- Set up a Netcat listener (`nc -nlvp 4444`) on the attacker machine to catch an incoming reverse shell.
- A PHP web shell (`shell.php`) was used to call back to the listener, landing an initial shell as the `kali` user (`shell.php` and `wordlist.txt` visible under `/home/kali`).
- Stabilized the shell into a full TTY:
  ```bash
  SHELL=/bin/bash script -q /dev/null
  python -c 'import pty; pty.spawn("/bin/bash")'
  stty raw -echo && fg
  ```

![pty upgrade](screenshots/15-python-pty-shell-upgrade.png)

### 5. Privilege Escalation

![md5sum](screenshots/13-md5sum-password-hash.png)
![sudo root shell](screenshots/14-sudo-root-shell.png)
![sudoers backdoor attempt](screenshots/16-sudoers-backdoor-attempt.png)

- Verified a candidate password (`rootroot1`) by comparing its MD5 hash.
- Used `sudo bash` with the recovered credentials to escalate to `root` (`whoami` confirmed `root`).
- Also attempted a `sudoers` backdoor:
  ```bash
  echo "echo 'armour ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers" > backup.sh
  sh backup.sh
  ```
  This failed with a permission-denied error, so escalation proceeded via `sudo su` instead.
- Searched for the lab's proof/flag file (`proof.txt`) under `/home/kali` and `/root`; it was not found in the locations checked at the time of capture.

### 6. Covering Tracks

![history clear attempt](screenshots/17-history-clear-attempt.png)

Attempted to clear shell history with `history -c` as both the `kali` user and `root`.

## Key Findings

- FTP, SSH, and HTTP were exposed; a hidden `index.html` page was discoverable only via directory brute-forcing with a tailored wordlist.
- A weak/guessable `sudo` password (`rootroot1`) allowed direct escalation to `root` via `sudo bash` / `sudo su`.
- An uploaded PHP web shell gave initial remote code execution before any credentials were known.
- A sudoers-file backdoor attempt via a shell script failed due to insufficient permissions, confirming `/etc/sudoers` was not writable by the low-privileged user.

## Repository Structure

```
.
├── README.md
└── screenshots/
    ├── 01-ifconfig-attacker-ip.png
    ├── 02-arp-scan-host-discovery.png
    ├── 03-nmap-ping-scan.png
    ├── 04-nmap-service-scan.png
    ├── 05-gobuster-dir-enum.png
    ├── 06-dirbuster-setup.png
    ├── 07-dirbuster-results.png
    ├── 08-custom-wordlist-created.png
    ├── 09-dirb-scan-indexhtml-found.png
    ├── 10-cat-wordlist.png
    ├── 11-browser-indexhtml-bluemoon.png
    ├── 12-netcat-listener.png
    ├── 13-md5sum-password-hash.png
    ├── 14-sudo-root-shell.png
    ├── 15-python-pty-shell-upgrade.png
    ├── 16-sudoers-backdoor-attempt.png
    └── 17-history-clear-attempt.png
```

## Author

NUR ELLYANA AZZATTI BINTI ROSLI — 52215125831
