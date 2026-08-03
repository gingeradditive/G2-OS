# 02 — Build e moduli

## Il meccanismo, in breve

G2-OS non usa CustomPiOS "classico": usa **[CustoPiZer](https://github.com/OctoPrint/CustoPiZer)**,
la variante di OctoPrint, invocata come action dalla CI. Il modello è semplice:

1. Si scarica un'immagine Raspberry Pi OS Lite ufficiale e la si monta.
2. Si esegue `chroot` dentro l'immagine.
3. Si eseguono **in ordine alfabetico** tutti gli script depositati nella cartella `scripts/`.
4. Si smonta e si ricomprime.

Tutto quello che sta in questo repo esiste per popolare quella cartella `scripts/`.

## Da `modules/` a `scripts/`

Il passaggio avviene in due step della CI
([`build.yml`](../.github/workflows/build.yml), step "📋 Copy generic and type scripts"):

```bash
cp -r modules/generic/* scripts/
cp -r modules/${matrix.type}/* scripts/     # per noi: modules/raspberry/*
rm scripts/*.disabled
```

Poi, se il target dichiara `special_modules`, i moduli corrispondenti vengono copiati da
`modules/special/`. **Il nostro target ha `special_modules: []`**, quindi `modules/special/` non
viene mai usato.

Conseguenze da tenere a mente:

- Le due cartelle `files/` (`modules/generic/files/` e `modules/raspberry/files/`) si **fondono**
  in un'unica `scripts/files/`, che nel chroot è raggiungibile come **`/files`**. Da qui i
  `cp /files/...` che si trovano ovunque nei moduli. Un file con lo stesso nome nelle due
  cartelle verrebbe sovrascritto silenziosamente (oggi non succede).
- `modules/armbian/` **non viene mai eseguito** con `type: raspberry`.
- Non esiste alcuna selezione per modulo: **tutto ciò che sta in `modules/generic/` finisce
  nell'immagine**. Per escludere un modulo lo si cancella, o si rinomina con suffisso
  `.disabled` (come fa oggi `11-armbian-net.disabled`).

## Il target di build

Definito in [`config.yml`](../config.yml). Oggi ne è attivo **uno solo**:

```yaml
- name: raspberry_pi-arm64-bookworm
  type: raspberry
  img_download_url: ".../2025-10-01-raspios-bookworm-arm64-lite.img.xz.torrent"
  img_hash_url:     ".../2025-10-01-raspios-bookworm-arm64-lite.img.xz.sha256"
  env:
    ARCH: arm64
    IMAGE_ENLARGEROOT: 7000     # MB di spazio aggiunti alla root (upstream usa 6000)
    MOUNT_PROC: 1
    MOUNT_SYS: 1
  special_modules: []
  rpi_json:
    init_format: "systemd"      # flusso firstrun.sh classico del RPi Imager
    devices: [pi3-64bit, pi4-64bit, pi5-64bit]
```

Le voci sotto `env` diventano il file `config.local`, con prefisso `EDITBASE_` aggiunto
automaticamente (`ARCH` → `EDITBASE_ARCH`), che CustoPiZer legge. È il canale con cui un target
parla ai moduli.

Punti rilevanti:

- L'URL dell'immagine base è **pinnato a una data precisa** (`2025-10-01`, Bookworm). Upstream
  è passato a `raspios_lite_arm64_latest` — rolling — e quindi oggi a Trixie.
  ✅ **Deciso il 03/08/2026** ([Q3](Q&A.md)): seguiamo upstream, `_latest` compreso. Il target
  diventerà `raspberry_pi-arm64-trixie`; la forma finale attesa è in
  [05-piano-rebase.md](05-piano-rebase.md#configyml).
- `devices` elenca ancora pi3/pi4/pi5 pur essendo la G2 solo un Pi 4 Model B. Si riduce a
  `pi4-64bit` ([Q1](Q&A.md)).
- Le altre cinque voci del file sono **commentate**, non cancellate: sono i target Armbian di
  upstream. Sono la ragione per cui `config.yml` è un punto di conflitto garantito a ogni merge.

## I moduli, in ordine di esecuzione

Ordine effettivo per il nostro target (fusione di `generic` e `raspberry`, ordinati
lessicograficamente). La colonna "Nostro?" distingue ciò che abbiamo scritto o modificato noi da
ciò che arriva da MainsailOS.

| # | Modulo | Nostro? | Cosa fa |
|---|---|---|---|
| 00 | [`00-upgrade`](../modules/generic/00-upgrade) | ereditato | `apt upgrade`, autoremove, aggiunge piwheels a `/etc/pip.conf` |
| 10 | [`10-config-raspberry`](../modules/raspberry/10-config-raspberry) | ereditato | Appende `boot-config.txt` a `config.txt`, **disabilita la console seriale** (per liberare la UART PL011), abilita i2c, disabilita Bluetooth, ingrandisce lo swap |
| 11 | [`11-fix-wifi`](../modules/raspberry/11-fix-wifi) | ereditato | `rfkill unblock wifi`, forza `WirelessEnabled=true` in NetworkManager |
| 30 | [`30-headless-nm`](../modules/generic/30-headless-nm) | ereditato | Servizio che legge `headless_nm.txt` dalla partizione di boot per configurare il Wi-Fi senza monitor |
| 31 | [`31-wifi-powersave-off`](../modules/generic/31-wifi-powersave-off) | ereditato | Regola udev che disattiva il power save Wi-Fi |
| 50 | [`50-kalico`](../modules/generic/50-kalico) | **nostro** | Clona `KalicoCrew/kalico` in `~/kalico`, crea `~/kalico-env`, installa `klippy-requirements.txt` + `numpy<1.26`, installa `kalico.service` |
| 51 | [`51-moonraker`](../modules/generic/51-moonraker) | ereditato | Clona Moonraker, lancia `install-moonraker.sh -s -z`, deposita il nostro `moonraker.conf` |
| 52 | [`52-gingerview`](../modules/generic/52-gingerview) | **nostro** | Installa e configura nginx, clona GingerView in **`~/mainsail`**, verifica che esista `build/` |
| 53 | [`53-crowsnest`](../modules/generic/53-crowsnest) | ereditato | Streaming webcam |
| 54 | [`54-timelapse`](../modules/generic/54-timelapse) | ereditato (patchato) | Componente Moonraker per i timelapse — **contiene un bug**, vedi [TODO.md](TODO.md) |
| 55 | [`55-sonar`](../modules/generic/55-sonar) | ereditato | Keepalive Wi-Fi |
| 57 | [`57-Kiauh`](../modules/generic/57-Kiauh) | **nostro** (da G1OS) | Clona KIAUH in `~/kiauh`. Non installa nulla, è solo un tool interattivo per l'utente |
| 58 | [`58-Obico`](../modules/generic/58-Obico) | **nostro** (da G1OS) | Installa `moonraker-obico` (monitoraggio cloud di terze parti) |
| 60 | [`60-Kamp`](../modules/generic/60-Kamp) | **nostro** (da G1OS) | Klipper Adaptive Meshing & Purging, macro per il bed leveling |
| 60 | [`60-mainsailos`](../modules/generic/60-mainsailos) | ereditato | Scrive `/etc/g2-os-release`, imposta hostname a `g2-os`, installa `python3-serial` e `python3-opencv` |
| 61 | [`61-EnableUSB`](../modules/generic/61-EnableUSB) | **nostro** (da G1OS) | Automount delle chiavette USB via `pmount` + udev, linkate in `printer_data/gcodes/` |
| 61 | [`61-postrename`](../modules/raspberry/61-postrename) | ereditato | Installa `/postrename` e lo aggancia a `rc.local` |
| 63 | [`63-SplashScreen`](../modules/raspberry/63-SplashScreen) | **nostro** (da G1OS) | Splash screen con `fbi` al boot — **contiene due problemi**, vedi [TODO.md](TODO.md) |
| 69 | [`69-G2-Service`](../modules/generic/69-G2-Service) | **nostro** | Clona il repo privato G2-Service e lancia `Scripts/install.sh` |
| 98 | [`98-remove-passwordless-sudo`](../modules/raspberry/98-remove-passwordless-sudo) | ereditato | Commenta la riga NOPASSWD di `pi` — chiude il buco aperto per la build |
| 99 | [`99-unhold-packages`](../modules/generic/99-unhold-packages) | ereditato | **Non fa nulla**: l'unica riga utile è commentata. Upstream l'ha cancellato in v3.0.0 |

> Due coppie di moduli condividono il prefisso numerico (`60-Kamp`/`60-mainsailos`,
> `61-EnableUSB`/`61-postrename`). L'ordine relativo dipende quindi dall'ordinamento del nome, e
> con la collation di sistema in C le maiuscole precedono le minuscole (`Kamp` prima di
> `mainsailos`). Nessuna delle due coppie ha oggi una dipendenza reale sull'ordine, ma è fragilità
> gratuita: meglio non aggiungerne altre.

## `files/00-config`: le variabili condivise

[`modules/generic/files/00-config`](../modules/generic/files/00-config) è sourceato da **ogni**
modulo. Definisce:

```bash
export BASE_USER=pi
export DIST_NAME=G2-OS
export PRINTER_DATA_DIRS="printer_data printer_data/{config,comms,logs,systemd}"
export MOONRAKER_CONFIG="/home/pi/printer_data/config/moonraker.conf"
create_user_directories()   # crea printer_data/* idempotentemente
is_raspbian()               # 1 se config.txt + /etc/rpi-issue esistono
```

`HOLD_PKGS` esiste ancora ma con quasi tutte le voci commentate, ed è comunque letto solo da
`00-upgrade`, dove pure la riga `apt-mark hold` è commentata. È codice morto: upstream ha rimosso
entrambi in v3.0.0.

Nota: upstream ha **aggiunto** `export BASE_PASSWORD=raspberry` in v3.0.0, perché su Trixie
il modulo raspberry ora imposta esplicitamente la password dell'utente. Noi non ce l'abbiamo:
è un punto da gestire durante il riallineamento ([05-piano-rebase.md](05-piano-rebase.md)).

## Il flusso di primo boot (`postrename`)

Il RPi Imager permette all'utente di scegliere username e password. Se l'utente non è `pi`,
tutto ciò che la build ha scritto con il percorso `/home/pi/...` va riscritto. È il compito di
[`modules/raspberry/files/postrename`](../modules/raspberry/files/postrename), lanciato da
`rc.local` al primo avvio: rinomina i servizi, sposta i venv, aggiusta i path nei file `.env`,
patcha le regole polkit.

Lo abbiamo modificato per Kalico (`SERVICES` contiene `kalico` invece di `klipper`, i venv
`kalico-env`), e per togliere le parti KlipperScreen.

**Questo file è il punto di conflitto più duro con upstream**: in v3.0.0 è stato quasi
interamente svuotato e la sua logica spostata in una libreria condivisa
(`/usr/local/lib/mainsailos/postrename-lib`) usata sia dal flusso `rc.local` sia dal nuovo flusso
cloud-init. Dopo il riallineamento le nostre modifiche Kalico **non vivranno più qui**, ma in
`postrename-lib`: mappatura riga per riga in
[05-piano-rebase.md](05-piano-rebase.md#postrename--postrename-lib).

E avendo deciso di adottare `INIT_FORMAT: cloudinit-rpi` ([Q8](Q&A.md)), **l'intero meccanismo
descritto qui sopra sparisce**: `61-postrename` esce subito per via del gate, `/postrename` e
l'aggancio a `rc.local` non finiscono più nell'immagine, e al loro posto ci sono due unit
systemd (`mainsailos-prerename.service`, `mainsailos-postrename.service`) installate da
`modules/generic/61-postrename-cloudinit`. Il file di logica è lo stesso, `postrename-lib`,
condiviso fra i due flussi.

## Come si prova una build

Non c'è modo di provarla su macOS. Le opzioni reali:

1. **CI** — push su `develop`, oppure `workflow_dispatch` sul workflow "Build Images".
   Attenzione: `69-G2-Service` clona un repository privato, quindi serve un secret con un token
   valido. È esattamente il motivo del commit `9eb8aab fix: secret`.
2. **Locale su Linux x86_64/arm64** — CustoPiZer si può eseguire a mano con Docker, replicando gli
   step della CI (creare `scripts/`, `config.local`, montare l'immagine). Nessuno script di
   convenienza esiste oggi in questo repo per farlo: sarebbe un'aggiunta utile
   ([TODO.md](TODO.md)).

## La cartella `patches/`

[`patches/`](../patches/) contiene due script (`patch101.sh`, `udev-fix.sh`) che **non fanno parte
della build**: sono patch una-tantum che gli utenti MainsailOS lanciavano via `curl | bash` su
immagini già installate, per versioni 1.0.x e per un bug di udev su Debian Bullseye. Riferiscono
URL di `mainsail-crew/MainsailOS`. Per G2-OS sono zavorra pura ([Q&A.md](Q&A.md), **Q5**).
