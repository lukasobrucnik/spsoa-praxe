# Úkol 1 – Domácí laboratoř (SSH přístup k Ubuntu Serveru)

Cílem bylo rozjet Ubuntu Server ve VirtualBoxu a nastavit k němu bezpečný přístup přes SSH klíč (bez hesla).

## Co jsem použil
- Oracle VirtualBox
- Ubuntu Server 24.04 LTS (ISO stažené z ubuntu.com/download/server)
- Windows 11 (hostitel) + PowerShell
- SSH klíč typu ed25519

## Postup

### 1. Instalace VirtualBoxu a Ubuntu Serveru
1. Stáhl jsem si ISO Ubuntu Serveru z oficiálních stránek (ubuntu.com/download/server) – je to zdarma, žádná registrace.
2. Ve VirtualBoxu jsem vytvořil nový virtuální stroj:
   - Typ: Linux, verze Ubuntu (64-bit)
   - RAM 2 GB, disk 20 GB (VDI, dynamicky alokovaný)
3. Do nastavení VM jsem vložil stažené ISO jako optický disk a spustil instalaci.
4. Během instalace jsem nastavil:
   - jméno serveru (hostname): `obrucnik-labs`
   - uživatele: `lukas`
   - zaškrtl jsem instalaci OpenSSH serveru rovnou v instalátoru (i tak jsem ho nakonec musel doinstalovat ručně, viz níže)
5. Po dokončení instalace jsem z VM vyndal ISO, aby se nebootovala znovu do instalátoru, a restartoval VM.

### 2. Síť – z NAT na Bridged Adapter
Původně jsem měl VM na NAT síti, ale z hostitele jsem se na ni nemohl SSH připojit (connection timed out). Přepnul jsem síťovou kartu VM na **Bridged Adapter** (Nastavení VM → Network → Attached to: Bridged Adapter), aby VM dostala IP adresu přímo z mého routeru, ve stejné síti jako počítač.

Po přepnutí jsem musel ještě ručně obnovit síť uvnitř VM:
```bash
sudo netplan apply
```
Pak jsem zjistil IP adresu:
```bash
ip a
```
(u mě vyšlo `192.168.0.18`, ale IP se může lišit projekt od projektu podle DHCP).

### 3. Instalace SSH serveru
Zjistil jsem, že SSH server nakonec nebyl nainstalovaný, tak jsem ho doinstaloval ručně:
```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
```
Ověření, že běží:
```bash
sudo systemctl status ssh
```

### 4. Vygenerování SSH klíče na hostiteli
Na svém Windows PC (ne uvnitř VM!) jsem v PowerShellu spustil:
```powershell
ssh-keygen -t ed25519 -C "obrucnik-lab"
```
Enter na výchozí umístění (`C:\Users\jméno\.ssh\id_ed25519`), Enter na passphrase (bez hesla ke klíči).

### 5. Nahrání veřejného klíče na server
Protože na Windows nemám `ssh-copy-id`, použil jsem tenhle příkaz:
```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh lukas@192.168.0.18 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"
```
Zadal jsem heslo uživatele (poslední použití hesla).

### 6. Otestování přihlášení klíčem
```powershell
ssh lukas@192.168.0.18
```
Přihlásilo mě to bez zadání hesla ✅

### 7. Vypnutí přihlašování heslem
Ve VM jsem upravil konfiguraci SSH serveru:
```bash
sudo nano /etc/ssh/sshd_config
```
Nastavil jsem:Uložil (Ctrl+O, Enter, Ctrl+X) a restartoval službu:
```bash
sudo systemctl restart ssh
```

**Než jsem zavřel aktuální spojení**, otevřel jsem nové okno a ověřil, že se pořád dostanu dovnitř klíčem – teprve pak jsem si byl jistý, že jsem se sám nezamkl venku.

### 8. Ověření, že heslo už nefunguje
```powershell
ssh -o PubkeyAuthentication=no lukas@192.168.0.18
```
Výsledek: `Permission denied (publickey)` – přihlášení heslem je opravdu vypnuté, funguje jen klíč.

### 9. Finální ověření (screenshot)
Po přihlášení přes SSH jsem spustil:
```bash
hostnamectl
date
```
a screenshot celého okna (příkaz, přihlášení, výstupy) jsem přiložil k odevzdání.

## Poznámky / co bych příště udělal jinak
- Zaškrtnutí OpenSSH serveru přímo v instalátoru bych příště zkontroloval hned po instalaci (`systemctl status ssh`), ať vím rovnou, jestli běží.
- Bridged síť je pro tenhle případ jednodušší než NAT + port forwarding, i když se mi IP adresa mezi restarty měnila (řešení: static DHCP lease na routeru, nebo si zjistit IP znovu přes `ip a`).

## Repozitář
Odkaz na tento a další úkoly z praxe: https://github.com/lukasobrucnik/spsoa-praxe

## Na co jsem využil AI
- generování tohoto readme
- pomoc s příkazy které jsme neznal a nevěděl bych jinak jak pokračovat. Věděl jsem co chci udělat a nastavit ale nevěděl jsme jaké správné comandy použít a v jaké postoupnosti. U každého jsme si nechal v krátkosti napsat co vlastně dělá a proč ho píšu.
