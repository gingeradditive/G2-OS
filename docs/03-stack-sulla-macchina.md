# 03 — Lo stack sulla macchina

Cosa esiste davvero su una G2 avviata dall'immagine prodotta oggi da `develop`.

## Servizi systemd

| Servizio | Da chi installato | Utente | Note |
|---|---|---|---|
| `kalico.service` | `50-kalico` | `pi` | Il firmware. `Before=moonraker.service` |
| `moonraker.service` | `51-moonraker` | `pi` | API su :7125 |
| `nginx` | `52-gingerview` | `www-data` | Reverse proxy + serve GingerView su :80 |
| `crowsnest.service` | `53-crowsnest` | `pi` | Streaming webcam su :8080–:8083 |
| `sonar.service` | `55-sonar` | `pi` | Keepalive Wi-Fi |
| `g2-service.service` | `69-G2-Service` → `install.sh` | **`root`** | FastAPI/uvicorn su :8000 |
| `headless_nm.service` | `30-headless-nm` | `root` | Configurazione Wi-Fi da file su partizione di boot |
| `splashscreen.service` | `63-SplashScreen` | `root` | `fbi` su tty1 |
| `usbstick-handler@.service` | `61-EnableUSB` | `root` | Template, attivato da udev all'inserimento |
| `moonraker-obico.service` | `58-Obico` | `pi` | Monitoraggio cloud di terze parti |

## Porte

| Porta | Chi ascolta | Esposta all'esterno |
|---|---|---|
| 80 | nginx | **sì** — è l'unico ingresso previsto |
| 7125 | Moonraker | sì (bind `0.0.0.0`), ma l'accesso normale passa da nginx |
| 8000 | G2-Service | sì (bind `0.0.0.0`), previsto l'accesso via nginx su `/service/` |
| 8080–8083 | Crowsnest / mjpg-streamer | sì, proxati da nginx su `/webcam/`…`/webcam4/` |

> ⚠️ **Collisione potenziale su :8080.** `upstreams.conf` mappa `mjpgstreamer1` su
> `127.0.0.1:8080`. Il README di G2-Service parla di un "server principale su 8080": se una
> versione futura di G2-Service usasse davvero quella porta, si scontrerebbe con Crowsnest. Oggi
> il `.service` reale di G2-Service usa `--port 8000` e la sua `location` nginx punta a 8000,
> quindi **non c'è collisione in atto** — ma il README di G2-Service è fuorviante su questo punto.

## Percorsi

```
/home/pi/
├── kalico/              # sorgenti Kalico (git)
├── kalico-env/          # venv Kalico
├── moonraker/           # sorgenti Moonraker (git)
├── moonraker-env/       # venv Moonraker
├── mainsail/            # ← GingerView (nome storico!), con build/ pre-compilata
├── G2-Service/          # sorgenti + venv/ di G2-Service
├── crowsnest/  sonar/  kiauh/  moonraker-obico/
├── Klipper-Adaptive-Meshing-Purging/
└── printer_data/
    ├── config/          # printer.cfg, moonraker.conf, KAMP_Settings.cfg, timelapse.cfg
    ├── comms/           # kalico.sock, kalico.serial
    ├── logs/            # kalico.log, mainsail-access.log, mainsail-error.log (symlink)
    ├── systemd/         # kalico.env
    └── gcodes/          # + symlink media/ → /media per le chiavette USB
```

### Perché GingerView sta in `~/mainsail`

Non è una svista: è un nome mantenuto di proposito. Tre cose lo referenziano e cambiarlo
richiederebbe di toccarle tutte insieme:

- `root /home/pi/mainsail/build;` nel site nginx;
- `path: ~/mainsail` nel blocco `[update_manager GingerView]` di `moonraker.conf`;
- i symlink dei log `mainsail-access.log` / `mainsail-error.log`.

Rinominarlo è un cleanup a sé stante, con un costo di migrazione sulle macchine già in campo
(l'update_manager di Moonraker punterebbe a una directory inesistente). Vedi [Q&A.md](Q&A.md), **Q6**.

## nginx

Un solo site, `/etc/nginx/sites-available/mainsail`, generato da
[`modules/generic/files/mainsail-nginx/mainsail`](../modules/generic/files/mainsail-nginx/mainsail).

```
root /home/pi/mainsail/build;    # SPA statica di GingerView

/                    → try_files … /index.html      (routing SPA)
/websocket           → apiserver (Moonraker :7125)
/moonraker/          → apiserver
/printer|api|access|machine|server/  → apiserver     (regex)
/webcam/ … /webcam4/ → mjpgstreamer1..4 (:8080-:8083)
```

`try_files $uri $uri/ /index.html` è ciò che rende funzionante il routing client-side di
GingerView; `index.html` è servito con `Cache-Control: no-store` così un aggiornamento della SPA
non resta appeso alla cache del browser.

### ⚠️ Manca la `include` per G2-Service

G2-Service, in `Scripts/install.sh`, deposita `/etc/nginx/g2-locations.d/g2-service.conf` con
dentro un `location ^~ /service/`. Il contratto — documentato in
`G2-Service/docs/05-deploy-e-sviluppo.md` — è che il site di G2-OS contenga, **dentro il blocco
`server`**:

```nginx
include /etc/nginx/g2-locations.d/*.conf;
```

**Questa riga non c'è.** Conseguenza: `/service/` non è raggiungibile dalla porta 80; G2-Service
risponde solo direttamente su `:8000`. Lo script di G2-Service se ne accorge e stampa un warning,
ma la build non fallisce.

Da notare: la documentazione di G2-Service attribuisce il site nginx a *GingerView*. È
impreciso — il site è un file di **G2-OS** (`modules/generic/files/mainsail-nginx/mainsail`), ed è
qui che va aggiunta la riga. Vedi [TODO.md](TODO.md), voce **T1**.

Una `include` con wildcard che non trova file non è un errore per nginx, quindi la modifica è
sicura anche su macchine senza G2-Service.

## `moonraker.conf`

Depositato da `51-moonraker` da
[`modules/generic/files/moonraker.conf`](../modules/generic/files/moonraker.conf). Punti che
contano:

```ini
[server]
klippy_uds_address: ~/printer_data/comms/kalico.sock   # ← rinominato per Kalico

[update_manager GingerView]
type: git_repo
primary_branch: main
path: ~/mainsail
origin: https://github.com/gingeradditive/GingerView.git
is_system_service: False

[update_manager klipper]           # ← il nome della sezione è ancora "klipper"
type: git_repo
path: ~/kalico
origin: https://github.com/KalicoCrew/kalico.git
channel: dev
managed_services: kalico
```

Il nome `[update_manager klipper]` **va lasciato invariato**, anche se stona: `klipper` e
`moonraker` sono per Moonraker due voci di update predefinite, e questa sezione le sovrascrive.
Rinominandola in `kalico` si otterrebbe una voce di update in più e, in parallelo, la voce
`klipper` di default che continua a puntare a un `~/klipper` inesistente. Quello che conta è che
`path`, `origin` e `managed_services` puntino a Kalico — ed è così.

Tre lacune:

1. **G2-Service non ha un blocco `[update_manager]`.** Il suo `install.sh` è scritto apposta per
   essere idempotente e usabile come `install_script` di Moonraker, ma nessuno lo registra: la
   macchina non può aggiornare G2-Service da sola. Vedi [TODO.md](TODO.md), **T2**.
2. **`[update_manager GingerView]` non dichiara un `install_script`.** Un aggiornamento fa solo
   `git pull`: funziona **solo perché GingerView committa `build/`** sul branch `main`. Se quella
   convenzione cambiasse in GingerView, gli aggiornamenti in campo servirebbero una SPA vecchia
   senza errori evidenti. È una dipendenza implicita fra due repo che vale la pena rendere esplicita.
3. **Residui MainsailOS**: `cors_domains` elenca ancora `my.mainsail.xyz`, e `[announcements]` è
   iscritto a `subscriptions: mainsail`. Innocui, ma sono annunci di un altro progetto mostrati
   in un'interfaccia che non è Mainsail.

## Kalico

```ini
# /home/pi/printer_data/systemd/kalico.env
KALICO_ARGS="/home/pi/kalico/klippy/klippy.py \
  /home/pi/printer_data/config/printer.cfg \
  -l /home/pi/printer_data/logs/kalico.log \
  -I /home/pi/printer_data/comms/kalico.serial \
  -a /home/pi/printer_data/comms/kalico.sock"
```

L'eseguibile è `klippy/klippy.py`: Kalico è un fork di Klipper e ha mantenuto i nomi interni.
Verificato contro il repository `KalicoCrew/kalico`, che espone `klippy/` e
`scripts/klippy-requirements.txt`. Sono esattamente le due correzioni dei commit `9a208bd` e
`5f33b94`, dopo che una sostituzione globale klipper→kalico li aveva rotti.

**`printer.cfg` non è depositato da G2-OS**: arriva da G2-Service. Se G2-Service non è installato,
Kalico parte e va in errore per configurazione mancante.

## Il flusso di primo boot

Questo è il flusso **attuale** (`init_format: systemd`):

1. Il RPi Imager scrive `firstrun.sh` che crea l'utente scelto dall'utente e, se diverso da `pi`,
   rinomina la home.
2. `rc.local` esegue `/postrename`, che ripara tutti i riferimenti a `/home/pi` (servizi, venv,
   file `.env`, polkit).
3. `headless_nm.service` legge `headless_nm.txt` dalla partizione di boot, se presente, e
   configura il Wi-Fi.
4. Kalico → Moonraker → nginx partono; G2-Service è indipendente e parte anche senza rete (per
   design: è da GingerView che si configura il Wi-Fi).

> ⚠️ **I passi 1 e 2 cambiano col riallineamento** — deciso il 03/08/2026 ([Q8](Q&A.md)):
> si adotta `INIT_FORMAT: cloudinit-rpi`. Diventeranno:
>
> 1. **cloud-init** legge `user-data` dalla partizione di boot e crea l'utente.
> 2. `mainsailos-prerename.service` e `mainsailos-postrename.service` fanno il lavoro che oggi
>    fa `/postrename`, usando la stessa logica spostata in
>    `/usr/local/lib/mainsailos/postrename-lib`.
>
> `rc.local` e `/postrename` **spariscono dall'immagine**: `61-postrename` esce subito per via
> del gate su `INIT_FORMAT`. I passi 3 e 4 restano identici.
>
> È qui che vivono le nostre personalizzazioni Kalico (`SERVICES`, `venvs`), ed è la parte più
> delicata del riallineamento: [05-piano-rebase.md](05-piano-rebase.md#postrename--postrename-lib).

## Hardware: cosa fa `boot-config.txt`

Appeso a `/boot/firmware/config.txt` da `10-config-raspberry`:

- `enable_uart=1` + `dtoverlay=disable-bt` — libera la **UART hardware PL011** per la
  comunicazione seriale con l'MCU. Il Bluetooth viene sacrificato: è la scelta corretta su una
  stampante, ma va saputa.
- `dtparam=spi=on` — richiesto dall'input shaper (accelerometro ADXL345).
- `dtparam=i2c_arm=on` — MCU host di Kalico.
- `gpu_mem=256` sotto `[pi4]`.

In più, `10-config-raspberry` toglie `console=serial0,115200` da `cmdline.txt`: senza questo la
console di debug del kernel occuperebbe la stessa UART.
