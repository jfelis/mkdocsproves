# Pràctica 2

## Informe de Pràctica: SSH-Guard i SSH-Audit

**Autor:** Sergi Giovanni Vique Villafuerte  
**Curs:** 1º CES-CIBER  
**Assignatura:** Bastionat de xarxes i sistemes  
**Professors:** Maite Martí, Juanjo Felis  

---

## Introducció

En aquesta pràctica de ciberseguretat s'ha realitzat un exercici complet de bastionat de serveis SSH, el qual inclou:

1. **Simulació d'un atac de força bruta** contra un servidor SSH.
2. **Implementació de mesures defensives** mitjançant **SSH-Guard**.
3. **Auditoria del sistema** mitjançant l'eina **SSH-Audit**.

### Entorn de proves
* **Servidor SSH:** IP `192.168.122.200`
* **Client atacant:** IP en la mateixa subxarxa `192.168.122.0/24`

---

## 1. Simulació d'atac de força bruta

### 1.1 Preparació de l'entorn

En la màquina client es va crear un fitxer de diccionari anomenat `fichero` que conté 6 paraules, entre les quals es troba la contrasenya vàlida de l'usuari `iker` en el servidor SSH.

```bash
osboxes@osboxes:~$ cat fichero
123456
password
test1234
iker
admin
EOF
```

### 1.2 Execució de l'atac

Es va utilitzar l'eina **Hydra** per simular l'atac de força bruta amb la següent ordre:

```bash
hydra -l iker -P fichero -s 2222 -t 10 -f -V ssh://192.168.122.200 -o hydra_resultados
```

Exemple d'execució:

```plain
osboxes@osboxes:~$ hydra -l iker -P fichero -s 2222 -t 10 -f -V ssh://192.168.122.200 -o hydra_resultados
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak
Hydra starting at 2025-10-30 12:21:44
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 6 tasks per 1 server, overall 6 tasks, 6 login tries (l:1/p:6), 1 try per task
[DATA] attacking ssh://192.168.122.200:2222/
[ATTEMPT] target 192.168.122.200 login "iker" pass "123456" 1 of 6
[ATTEMPT] target 192.168.122.200 login "iker" pass "password" 2 of 6
[ATTEMPT] target 192.168.122.200 login "iker" pass "test1234" 3 of 6
[2222][ssh] host: 192.168.122.200   login: iker   password: iker
[STATUS] attack finished for 192.168.122.200 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra finished at 2025-10-30 12:21:45
```

**Paràmetres utilitzats:**
* `-l iker`: Especifica l'usuari objectiu.
* `-P fichero`: Defineix el diccionari de contrasenyes.
* `-s 2222`: Indica el port del servei SSH.
* `-t 10`: Executa 10 tasques en paral·lel.
* `-f`: Finalitza l'atac en trobar la primera combinació vàlida.
* `-V`: Mode verbós (mostra el procés en detall).
* `-o hydra_resultados`: Desa el resultat en un fitxer.

### 1.3 Resultat de l'atac

En verificar el fitxer de resultats:

```bash
osboxes@osboxes:~$ cat hydra_resultados
# Hydra v9.5 run at 2025-10-30 12:21:44 on 192.168.122.200 ssh
[2222][ssh] host: 192.168.122.200   login: iker   password: iker
```

Es va confirmar que s'havia après la informació de login i password correctes, demostrant la vulnerabilitat del servidor SSH sense mesures de protecció.

---

## 2. Implementació de SSH-Guard

### 2.1 Instal·lació i configuració

Després d'instal·lar **SSH-Guard** en el servidor, es va configurar el fitxer `/etc/sshguard/sshguard.conf`:

```ini
#### REQUIRED CONFIGURATION ####
# Full path to backend executable
BACKEND="/usr/libexec/sshguard/sshg-fw-nft-sets"

# Shell command that provides logs on standard output
LOGREADER="LANG=C journalctl -afb -p info -n1 -t sshd -o cat"

#### OPTIONS ####
THRESHOLD=40
BLOCK_TIME=30
DETECTION_TIME=60
WHITELIST_FILE=/etc/sshguard/whitelist
```

!!! info "Explicació dels paràmetres"
    * **`THRESHOLD=40`**: Requereix 40 punts de perillositat acumulats per activar el bloqueig.
    * **`BLOCK_TIME=30`**: Bloqueig temporal de 30 segons per als usuaris no autoritzats.
    * **`DETECTION_TIME=60`**: Finestra temporal de 60 segons per a l'acumulació de punts d'atacs.

### 2.2 Prova d'efectivitat amb SSH-Guard actiu

Mentre s'executava novament l'atac de Hydra des del client, es va monitoritzar el registre del servidor mitjançant:

```bash
sudo tail -f /var/log/auth.log
```

**Registre obtingut:**
```plain
2025-10-30T16:39:59.785167+00:00 giovanniserer sshguard[13273]: Attack from "192.168.122.50" on service SSH with danger 10.
2025-10-30T16:39:59.788046+00:00 giovanniserer sshguard[13273]: message repeated 2 times: [ Attack from "192.168.122.50" on service SSH with danger 10. ]
2025-10-30T16:39:59.788236+00:00 giovanniserer sshguard[13273]: Blocking 192.168.122.50/32 for 30 secs (4 attacks in 3 secs, after 1 abuses over 3 secs.)
```

**Interpretació del log:**
1. Es detecten 4 atacs en 3 segons.
2. S'acumulen punts suficients per superar el `THRESHOLD`.
3. SSH-Guard bloqueja automàticament la IP `192.168.122.50` durant 30 segons.

### 2.3 Verificació de les regles de filtrat (`nftables`)

**Estat inicial (sense bloquejos):**
```bash
iker@giovanniserer:~$ sudo nft list table sshguard
table ip sshguard {
    set attackers {
        type ipv4_addr
        flags interval
    }
    chain blacklist {
        type filter hook input priority filter + 10; policy accept;
        ip saddr @attackers drop
    }
}
```

**Estat posterior a l'atac (IP bloquejada):**
```bash
iker@giovanniserer:~$ sudo nft list table sshguard
table ip sshguard {
    set attackers {
        type ipv4_addr
        flags interval
        elements = { 192.168.122.50 }
    }
    chain blacklist {
        type filter hook input priority filter + 10; policy accept;
        ip saddr @attackers drop
    }
}
```

### 2.4 Conclusió de SSH-Guard

La integració de **SSH-Guard** amb **nftables** permet una protecció eficient a nivell de nucli (*kernel*), bloquejant automàticament adreces hostils sense afectar el rendiment global ni perjudicar els usuaris legítims.

---

## 3. Auditoria SSH amb SSH-Audit

### 3.1 Instal·lació de l'eina

En la màquina client s'instal·la l'eina d'auditoria:

```bash
sudo apt install ssh-audit -y
```

### 3.2 Execució de la primera auditoria

Es realitza el primer anàlisi contra el port SSH personalitzat:

```bash
ssh-audit -p 2222 192.168.122.200
```

**Resultats obtinguts (extracte):**

```plain
# general
(gen) banner: SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.14
(gen) software: OpenSSH 9.6p1
(gen) compatibility: OpenSSH 8.5+, Dropbear SSH 2018.76+
(gen) compression: enabled (zlib@openssh.com)

# key exchange algorithms
(kex) sntrup761x25519-sha512@openssh.com      -- [info] available since OpenSSH 8.5
(kex) curve25519-sha256                       -- [info] available since OpenSSH 7.4
(kex) ecdh-sha2-nistp256                      -- [fail] using elliptic curves that are suspected of being backdoored by the U.S. NSA
(kex) ecdh-sha2-nistp384                      -- [fail] using elliptic curves that are suspected of being backdoored by the U.S. NSA
(kex) ecdh-sha2-nistp521                      -- [fail] using elliptic curves that are suspected of being backdoored by the U.S. NSA
```

!!! failure "Vulnerabilitats detectades"
    L'eina assenyala l'ús d'algoritmes d'intercanvi de claus febles o sospitosos (`ecdh-sha2-nistp256/384/521`), associats a corbes el·líptiques amb possibles portes darrere de la NSA.

---

### 3.3 Millores de seguretat aplicades (`/etc/ssh/sshd_config`)

Per corregir les febleses trobades, s'ha modificat el fitxer de configuració del dimoni SSH:

```bash
sudo nano /etc/ssh/sshd_config
```

Ajustos realitzats:

```ini
Protocol 2
HostKey /etc/ssh/ssh_host_ed25519_key
Ciphers aes128-ctr,aes192-ctr,aes256-ctr
MACs hmac-sha2-256,hmac-sha2-512
```

!!! tip "Resum de millores aplicades"
    * **`Protocol 2`**: Elimina qualsevol suport per al protocol obsolet SSH-1.
    * **`HostKey`**: Deshabilita claus antigues i utilitza l'algoritme modern **ed25519**.
    * **`MACs`**: Limita els algorismes d'integritat a opcions robustes basades en SHA-2.
    * **`Ciphers`**: Estableix cifrats de bloc segurs en mode CTR.

Reina el servei per aplicar els canvis:

```bash
sudo systemctl restart sshd
```

---

### 3.4 Segona auditoria de comprovació

Després d'aplicar les millores, es compara el resultat de les dues auditories desar en fitxers separats (`ficherooooo` vs `bajsdjasdbkjafs`):

```bash
grep -Fxvf ficherooooo bajsdjasdbkjafs
```

**Eixida de diferències:**

```plain
(rec) +aes128-gcm@openssh.com    enc algorithm to append
(rec) +aes256-gcm@openssh.com    enc algorithm to append
(rec) +rsa-sha2-256              key algorithm to append
(rec) +rsa-sha2-512              key algorithm to append
```

**Conclusió final:**  
Les alertes crítiques sobre algoritmes febles han desaparegut. L'eina ara només suggereix afegir opcionalment algoritmes moderns com **AES-GCM** o **RSA-SHA2**, confirmant un enduriment (*hardening*) efectiu del servidor SSH.
