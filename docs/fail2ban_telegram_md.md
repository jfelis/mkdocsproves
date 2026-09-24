# fail2ban i Telegram

## Fail2ban

**Fail2Ban** és una eina de seguretat per a servidors Linux que ajuda a protegir contra atacs de força bruta. La seva principal funció és monitoritzar els fitxers de registre del sistema per detectar patrons específics relacionats amb intents d'inici de sessió fallits, i després pren accions com bloquejar temporalment l'adreça IP de l'origen d'aquests intents.

Aquí tens com instal·lar i configurar Fail2Ban en un sistema Ubuntu:

### 1. Actualitza el sistema

Abans d'instal·lar nous paquets, és una bona pràctica actualitzar la llista de paquets disponibles i actualitzar els paquets ja instal·lats.

```bash
sudo apt update
sudo apt upgrade
```

### 2. Instal·la Fail2Ban

Utilitza el gestor de paquets `apt` per instal·lar `fail2ban`.

```bash
sudo apt install fail2ban
```

### 3. Configuració bàsica de Fail2Ban

Després de la instal·lació, Fail2Ban crea els seus fitxers de configuració a `/etc/fail2ban/`. El fitxer principal és `fail2ban.conf`, i les regles específiques es configuren a través de fitxers de jail com `jail.conf` o `jail.local`.

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

### 4. Configura Jail per a SSH (`sshd`)

Obre el fitxer de configuració de jail local amb un editor de text. Pots utilitzar `nano` o qualsevol editor que prefereixis.

```bash
sudo nano /etc/fail2ban/jail.local
```

Cerca la secció `[sshd]` i assegura't que estigui configurada per habilitar Fail2Ban per a SSH. Pots ajustar el valor de `maxretry` segons les teves preferències.

```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
```

Guarda els canvis i tanca l'editor.

### 5. Inicia i habilita el servei Fail2Ban

Inicia el servei Fail2Ban i habilita que s'executi en iniciar-se el sistema.

```bash
sudo service fail2ban start
sudo service fail2ban enable
```

O utilitzant `systemctl`:

```bash
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
```

A partir d'aquest moment, Fail2Ban està monitoritzant els intents d'inici de sessió fallits a través del registre del sistema (en aquest cas, `/var/log/auth.log` per SSH) i pren accions com bloquejar temporalment les adreces IP que superin el límit especificat.

Pots revisar els registres i configurar més opcions mitjançant els fitxers de configuració de Fail2Ban. També pots crear regles personalitzades per a altres serveis i protocols. La documentació oficial de Fail2Ban és un bon recurs per a més detalls i configuracions avançades: [Fail2Ban Wiki](https://github.com/fail2ban/fail2ban/wiki).

---

## Enviament de notificacions

### Crear bot de Telegram

#### Crear bot i grup
Has de crear un bot de Telegram i un grup, i afegir el bot al grup per realitzar la pràctica. Has d'obtenir el token del bot i l'identificador de la conversa per configurar el script.

Crear un nou bot a Telegram és un procés senzill que es fa a través de Telegram amb l'ajuda del **BotFather**, un bot oficial de Telegram que permet crear i gestionar bots. Aquí tens els passos per crear un nou bot:

1. **Inicia una conversa amb el BotFather:**
   * Vés a la barra de cerca de Telegram i escriu `BotFather`.
   * Inicia una conversa amb el BotFather fent clic sobre ell i després fes clic a "Iniciar" o envia el missatge `/start`.
2. **Crea un nou bot:**
   * Envia el missatge `/newbot` per començar el procés de creació.
3. **Proporciona un nom per al teu bot:**
   * El BotFather et demanarà un nom per al teu bot. Escull un nom que sigui únic, ja que aquest nom s'utilitzarà com a nom d'usuari visible del bot.
4. **Proporciona un nom d'usuari per al teu bot:**
   * Aquest nom d'usuari ha de ser únic i acabar obligatòriament amb la paraula `bot` (per exemple, `example_bot`).
5. **Rebràs el token del teu bot:**
   * Una vegada completat el procés, BotFather t'enviarà un missatge amb el token. Aquest token és una clau única que identifica el teu bot i s'utilitza per a les comunicacions amb l'API de Telegram.

!!! warning "Seguretat del Token"
    Guarda aquest token de manera segura, ja que el necessitaràs per programar o configurar el teu bot i permet el control total d'aquest.

Per a més informació sobre com interactuar amb l'API de Telegram, pots revisar la documentació oficial: [Telegram Bot API](https://core.telegram.org/bots/api).

---

### Exemple de script per enviar un text per Telegram

Fitxer: `/etc/fail2ban/scripts/test_telegram.sh`

```bash
#!/bin/bash

USERID=""                              # Chat ID al que volem enviar el missatge.
KEY=""                                 # API Key generada per BotFather.
TIMEOUT="10"                           # Timeout de la petició a la API.
URL="https://api.telegram.org/bot$KEY/sendMessage" # URL de l'API per enviar missatges.

LOG="envio_telegram_$(date "+%d%m%Y").log" # Log d'enviament de missatges.
SONIDO=0                               # 0 = amb so de notificació, 1 = silenciosa.
FECHA_EJEC="$(date "+%d %b %H:%M:%S")" # Data i hora d'execució.

# Condicional per enviar so o no amb la notificació (segon paràmetre)
if [ "$2" -eq 1 ]; then
    SONIDO=1
fi

TEXTO="<b>$FECHA_EJEC:</b>\n<pre>$1</pre>" # Text a enviar: Data d'execució i primer paràmetre del script.

# Realitzem la petició a l'API amb la informació recopilada
curl -s --max-time $TIMEOUT -d "parse_mode=HTML&disable_notification=$SONIDO&chat_id=$USERID&disable_web_page_preview=true&text=$TEXTO" $URL >> $LOG 2>&1

echo "" >> $LOG # Introduïm una línia nova en el log.
```

#### Paràmetres del script
Es pot modificar el script perquè rebi la IP bloquejada com a paràmetre per comunicar-la.

---

### Configurar Fail2Ban per executar el script

#### 1. Afegir l'acció a `/etc/fail2ban/jail.local` (o `jail.conf`)

Edita la secció `[sshd]` per afegir l'acció de Telegram:

```ini
[sshd]
# To use more aggressive sshd modes set filter parameter "mode" in jail.local:
# normal (default), ddos, extra or aggressive (combines all).
# See "tests/files/logs/sshd" or "filter.d/sshd.conf" for usage example and details.
#mode    = normal
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
action  = iptables[name=SSH, port=22, protocol=tcp]
          telegram
```

!!! failure "Atenció a la tabulació"
    Hi ha que tabular correctament `telegram` sota l'opció `action` o el servei no funcionarà!!

#### 2. Crear l'arxiu de configuració d'acció `/etc/fail2ban/action.d/telegram.conf`

```ini
[Definition]
actionstart = /etc/fail2ban/scripts/test_telegram.sh "iniciant telegram"
actionstop  = /etc/fail2ban/scripts/test_telegram.sh "parant telegram"
actioncheck = 
actionban   = /etc/fail2ban/scripts/test_telegram.sh "s'ha bloquejat una IP"
actionunban = /etc/fail2ban/scripts/test_telegram.sh "s'ha desbloquejat una IP"

[Init]
init = 123
```

!!! warning "Carpeta scripts"
    Ubica el script en la carpeta `/etc/fail2ban/scripts` i comprova que tingui permisos d'execució:
    ```bash
    chmod +x /etc/fail2ban/scripts/test_telegram.sh
    ```

---

### Resposta i verificació de Telegram

T'aconselle que proves primer el script i comproves que envia correctament a Telegram. Si és així, a més a més d'aparèixer els missatges a la xat, es crea un registre (*log*) en una estructura semblant a aquesta:

```json
{
  "ok": true,
  "result": {
    "message_id": 19,
    "from": {
      "id": 6440810779,
      "is_bot": true,
      "first_name": "fail2baninformerbot",
      "username": "fail2baninformerbot"
    },
    "chat": {
      "id": -4000475834,
      "title": "Avisos fail2ban",
      "type": "group",
      "all_members_are_administrators": true
    },
    "date": 1702915556,
    "text": "18 de des. 18:05:56:\nhola",
    "entities": [
      { "offset": 0, "length": 20, "type": "bold" },
      { "offset": 21, "length": 4, "type": "pre" }
    ]
  }
}
```

### Funcionament

Si tot funciona correctament, quan es bloqueja una IP el bot de Telegram es comunica enviant missatges com els següents:

* `18 de des. 18:15:41:` -> **iniciant fail2ban**
* `18 de des. 18:29:32:` -> **un equip bloquejat**
* `18 de des. 18:29:42:` -> **un equip desbloquejat**

Pots consultar el log detallat de Fail2Ban en qualsevol moment mitjançant:

```bash
cat /var/log/fail2ban.log
```

Exemple d'eixida del log:
```plain
2023-12-18 18:15:41,685 fail2ban.jail   [655]: INFO    Jail sshd started
2023-12-18 18:28:57,667 fail2ban.filter [655]: INFO    [sshd] Found 10.2.0.146 - 2023-12-18 18:28:57
2023-12-18 18:29:01,512 fail2ban.filter [655]: INFO    [sshd] Found 10.2.0.146 - 2023-12-18 18:29:01
2023-12-18 18:29:07,419 fail2ban.filter [655]: INFO    [sshd] Found 10.2.0.146 - 2023-12-18 18:29:06
2023-12-18 18:29:12,934 fail2ban.filter [655]: INFO    [sshd] Found 10.2.0.146 - 2023-12-18 18:29:12
2023-12-18 18:29:32,655 fail2ban.filter [655]: INFO    [sshd] Found 10.2.0.146 - 2023-12-18 18:29:32
2023-12-18 18:29:32,869 fail2ban.actions[655]: NOTICE  [sshd] Ban 10.2.0.146
2023-12-18 18:29:42,456 fail2ban.actions[655]: NOTICE  [sshd] Unban 10.2.0.146
```