# Lab notes — Phase 0

## Important commands

```bash
sudo sshd -t
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication|...)$'
sudo systemctl reload ssh
sudo ufw status verbose
sudo fail2ban-client status sshd
timedatectl
sudo ss -tulpn
sudo apt-get update && sudo apt-get upgrade -y
test -f /var/run/reboot-required && echo reboot_needed || echo no_reboot
```

## Decisions

- SSH hardening via drop-in `99-kubernetes-lab-hardening.conf`, not a full rewrite.
- Kept port 22; left TCP forwarding enabled for Cursor Remote SSH.
- UFW SSH left as Anywhere; rely on DigitalOcean Cloud Firewall for source IP restriction when stable.
- Fail2ban: local `sshd.local` override; modest ban timers for a lab VPS.
- UTC retained; no timezone change to match operator laptop.
- System SSH backups stored under `/home/sysadmin/backups/` (outside Git).
- Temporary passwordless sudo used for agent automation, then removed; passworded sudo restored.
- Operator reconfirmed effective SSH settings after Phase 0; no further SSH changes planned.

## Problems / unexpected behavior

- Agent shell had no TTY; passworded sudo blocked automation until NOPASSWD was temporarily enabled, then removed.
- Host already had substantial SSH lockdown and active UFW/Fail2ban from earlier manual prep; Phase 0 validated and formalized rather than inventing from zero.
- Effective `LoginGraceTime` was 120 until the drop-in was loaded and SSH reloaded (then 30).
- `ssh.service` shows as disabled while `ssh.socket` is enabled (Ubuntu socket activation) — expected; SSH still active.

## Reboot status

No reboot required after Phase 0 updates (`/var/run/reboot-required` absent).

## Questions for later phases

- How will UFW coexist with K3s CNI and kube-proxy?
- Should API server access be limited to `<TRUSTED_ADMIN_IP>` at both cloud and host layers?
- Which droplet-agent / metadata paths remain acceptable on a hardened lab node?
