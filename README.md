# SOC Home Lab: Detecting an SSH Brute-Force Attack

> A hands-on blue-team lab where I attacked my own Linux server, detected the
attack in the logs, investigated it as a SOC Analyst L1, and contained it.
> 

**Author:** Gerald Barrera · **Date:** November 2026
**Tools:** VirtualBox · Ubuntu Server · Kali Linux · Nmap · Hydra · tcpdump · Wireshark · UFW

---

## Objective

Simulate a real SSH password-guessing attack in an isolated lab, then practice the
full SOC Analyst L1 workflow: **detect → investigate → contain → report**.

## Lab architecture

| Machine | Role | IP |
| --- | --- | --- |
| Ubuntu Server (`ubuntu-lab`) | Victim / defended server | 10.0.2.4 |
| Kali Linux | Attacker | 10.0.2.5 |

Both VMs run on a VirtualBox **NAT Network** (`NatNetwork`), isolated from my home network.

> ⚠️ **Safety:** every attack was run against my own lab VMs only. Scanning or
attacking systems without permission is illegal.
> 

---

## Part 1 — Setup (Ubuntu victim)

```bash
# Create the weak victim user (password: sunshine, on purpose)
sudo adduser student

# Confirm SSH is listening on port 22
sudo ss -tlnp | grep :22

# Confirm SSH accepts password authentication
sudo grep -ri passwordauthentication /etc/ssh/sshd_config /etc/ssh/sshd_config.d/

# Find the server IP
ip a

# Confirm the auth log exists
ls -l /var/log/auth.log
```

## Part 2 — The attack (Kali)

```bash
# Start capturing SSH traffic on Ubuntu (run on the victim)
sudo tcpdump -i any -w /home/analyst/attack.pcap port 22 &

# 1. Reconnaissance: scan for open ports and services  (MITRE T1046)
nmap -sV 10.0.2.4

# 2. Build a small password list
cat > passwords.txt << 'EOF'
123456
password
qwerty
admin
letmein
welcome
football
iloveyou
monkey
dragon
sunshine
princess
EOF

# 3. Brute force the SSH login  (MITRE T1110.001)
hydra -l student -P passwords.txt -t 4 ssh://10.0.2.4

# 4. Log in with the cracked password  (MITRE T1078)
ssh student@10.0.2.4

# 5. Attacker discovery commands  (MITRE T1082, T1087.001)
whoami
id
hostname
uname -a
cat /etc/passwd
ls -la /home
sudo -l                 # fails: student is not in sudoers
touch /tmp/.backdoor    # drop a hidden marker file
exit

# Stop the capture on Ubuntu
sudo pkill tcpdump
```

## Part 3 — Investigation (Ubuntu, as the analyst)

```bash
# Count failed logins
sudo grep -c "Failed password" /var/log/auth.log

# Which IPs attacked, and how many times each
sudo grep "Failed password" /var/log/auth.log | grep -oE "from [0-9.]+" | sort | uniq -c | sort -rn

# Which usernames were targeted
sudo grep "Failed password" /var/log/auth.log | grep -oE "for (invalid user )?[a-z]+" | sort | uniq -c

# First and last failed login times
sudo grep "Failed password" /var/log/auth.log | head -1
sudo grep "Failed password" /var/log/auth.log | tail -1

# THE KEY QUESTION: did the attacker get in?
sudo grep "Accepted password" /var/log/auth.log

# See failures and the success together, in order
sudo grep -E "Failed password|Accepted password" /var/log/auth.log

# Did the attacker try to escalate privileges?
sudo grep "NOT in sudoers" /var/log/auth.log

# What commands did the attacker run?
sudo cat /home/student/.bash_history

# Did the attacker leave files behind?
ls -la /tmp

# Analyze the captured traffic
sudo tcpdump -nn -r /home/analyst/attack.pcap 'tcp[tcpflags] == tcp-syn' | wc -l
# (or open attack.pcap in Wireshark and filter: tcp.flags.syn == 1 && tcp.flags.ack == 0)
```

## Part 4 — Response and containment (Ubuntu)

```bash
# 1. Preserve evidence BEFORE changing anything
sudo cp /var/log/auth.log /home/analyst/auth-evidence.log

# 2. Kill the attacker's sessions and lock the account
sudo pkill -KILL -u student
sudo passwd -l student

# 3. Block the attacker IP with the firewall
sudo ufw allow 22/tcp
sudo ufw insert 1 deny from 10.0.2.5
sudo ufw enable
sudo ufw status numbered

# 4. Remove the attacker's file
sudo rm /tmp/.backdoor

# 5. Verify the block (from Kali, SSH should now time out)
ssh student@10.0.2.4
```

---

## Evidence summary

| Item | Value |
| --- | --- |
| Attack type | SSH brute force → account compromise |
| Attacker IP | 10.0.2.5 |
| Target | 10.0.2.4, port 22 |
| Compromised account | `student` |
| Failed logins | 13 |
| TCP connections (Wireshark) | 8 |
| Successful login | 1 (right after the failures) |
| Privilege escalation | Attempted, **failed** (not in sudoers) |
| File created by attacker | `/tmp/.backdoor` |
| Passwords visible in traffic? | No — SSH is encrypted |
| Verdict | **True positive** |

## MITRE ATT&CK mapping

| Tactic | Technique |
| --- | --- |
| Discovery | T1046 Network Service Discovery |
| Credential Access | T1110.001 Brute Force: Password Guessing |
| Initial Access | T1078 Valid Accounts |
| Discovery | T1082 System Information Discovery |
| Discovery | T1087.001 Account Discovery: Local Account |

---

## What I learned

- **Linux auth logs (`/var/log/auth.log`) are the primary evidence** for SSH attacks.
`Failed password` and `Accepted password` lines tell the whole story.
- **A success right after many failures = a compromise.** The 1-second gap between
the last failure and the login is strong evidence of an automated tool, not a human.
- **The network and the logs tell different parts of the story.** Wireshark showed 8 TCP
connections, but the logs showed 13 login attempts — because SSH allows several
password attempts per connection. A good analyst correlates both sources.
- **SSH encrypts credentials in transit**, so passwords are never visible in the capture.
- **`grep`, `sort`, `uniq` and `wc` are powerful triage tools** for finding patterns in logs.
- **Preserve evidence before containment** — copy the logs before you lock accounts or
change the system.
- **Containment has layers:** lock the account, block the IP, remove attacker artifacts,
then verify each action worked.
- **Context decides the verdict.** My own mistyped passwords also appeared as failed
logins from Kali’s IP — a *benign true positive*. Analysts must check context before
escalating.

## How to prevent this attack

1. Disable SSH password login; use **SSH keys**.
2. Install **fail2ban** to auto-block IPs after repeated failures.
3. Enforce a **strong password policy**.
4. Alert on **“successful login after 5+ failures”** in the SIEM.

---

*Lab completed as part of my journey toward a SOC Analyst role.* 

*-Gerald Barrera Solano*
