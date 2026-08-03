# TODO — Difetti e debito tecnico

Cose trovate leggendo il codice il **03/08/2026**, non ancora affrontate. Nessuna è stata
corretta: questo file esiste per non doverle riscoprire.

Priorità: 🔴 rompe una funzionalità · 🟡 rischio o incoerenza · 🟢 pulizia.

---

## 🔴 T1 — `/service/` non è raggiungibile via nginx

**Dove:** [`modules/generic/files/mainsail-nginx/mainsail`](../modules/generic/files/mainsail-nginx/mainsail)

G2-Service deposita `/etc/nginx/g2-locations.d/g2-service.conf` con la propria `location ^~
/service/`, contando su una riga di include nel site. **Quella riga non c'è**, quindi il servizio
risponde solo su `:8000` e non dalla porta 80.

`G2-Service/Scripts/install.sh` se ne accorge e stampa un warning (`nessun site nginx include
g2-locations.d/`), ma la build non fallisce: il problema si manifesta solo a runtime.

**Fix:** dentro il blocco `server { … }`, accanto alle altre `location`:

```nginx
include /etc/nginx/g2-locations.d/*.conf;
```

L'include con wildcard che non trova file non è un errore per nginx, quindi la modifica è sicura
anche sulle immagini senza G2-Service. Va anche creata la directory in `52-gingerview`, altrimenti
su una macchina senza G2-Service il site non ha nulla da includere e `nginx -t` resta comunque
valido — ma tanto vale essere espliciti.

> Nota: `G2-Service/docs/05-deploy-e-sviluppo.md` attribuisce il site nginx a *GingerView*. È
> impreciso, il file è di **G2-OS**: quando si sistema questo, correggere anche là.

---

## 🔴 T2 — G2-Service non è aggiornabile dalla macchina

**Dove:** [`modules/generic/files/moonraker.conf`](../modules/generic/files/moonraker.conf)

Manca il blocco `[update_manager G2-Service]`. Il suo `install.sh` è scritto apposta per essere
idempotente e usabile come `install_script` di Moonraker, ma nessuno lo registra: una G2 in campo
non può aggiornare il proprio backend.

Non è però un fix da un minuto: G2-Service è un repository **privato** e l'update_manager
dovrebbe autenticarsi per fare `git pull`. Vedi [Q&A.md](Q&A.md) **Q7** — è una decisione, non
solo del codice da scrivere.

---

## 🔴 T3 — `54-timelapse` crea un symlink rotto — *si chiude col riallineamento*

> 🟢 **Il rebase lo risolve gratis.** `fix_timelapse_links` in `postrename-lib` di upstream usa
> `klipper_macro/`, che è il nome corretto: basta risolvere il conflitto **verso upstream** su
> quella riga e non riapplicarci sopra il nostro `kalico_macro`. Resta invece da correggere a
> mano `modules/generic/54-timelapse`, che upstream non ha toccato.
> Verifica al passo 6 di [05-piano-rebase.md](05-piano-rebase.md):
> `grep -rn "kalico_macro" modules/` deve non trovare nulla.

**Dove:** [`modules/generic/54-timelapse`](../modules/generic/54-timelapse) riga ~60, e
[`modules/raspberry/files/postrename`](../modules/raspberry/files/postrename) in
`fix_timelapse_links`

```bash
ln -sf "${SRC_DIR}/kalico_macro/timelapse.cfg" "${PRINTER_CONFIG_PATH}/timelapse.cfg"
```

**In `mainsail-crew/moonraker-timelapse` quella directory si chiama `klipper_macro/`, non
`kalico_macro/`** (verificato sul repository remoto il 03/08/2026). È un danno collaterale della
sostituzione globale klipper→kalico del commit `aa48c07`, che ha rinominato anche path dentro
repository di terze parti.

`ln -sf` su una sorgente inesistente **riesce** — crea un symlink pendente — quindi la build non
fallisce e il problema si vede solo quando Kalico prova a includere `timelapse.cfg`.

**Fix:** ripristinare `klipper_macro` in entrambi i punti. Se invece si decide che il timelapse
non serve ([Q&A.md](Q&A.md) **Q2**), il fix è cancellare il modulo.

---

## 🟡 T4 — `63-SplashScreen`: due problemi

**Dove:** [`modules/raspberry/63-SplashScreen`](../modules/raspberry/63-SplashScreen)

**(a) L'immagine di splash non esiste.** Il servizio lancia:

```
ExecStart=/usr/bin/fbi --noverbose -a /home/pi/printer_data/config/splash.png
```

`splash.png` **non è depositato da nessun modulo di G2-OS**, e non esiste in G2-Service né in
GingerView (verificato con grep su tutti e tre i repo). Il servizio fallisce a ogni boot. Va
aggiunto il file — probabilmente in `modules/raspberry/files/` — o rimosso il modulo.

**(b) Percorso di `cmdline.txt` probabilmente sbagliato** — *da verificare*.

```bash
SPLASHSCREEN_BOOT_CONFIG="/boot/firmware/config.txt"    # nuovo layout
SPLASHSCREEN_CMDLINE_CONFIG="/boot/cmdline.txt"         # vecchio layout
```

Le due righe usano layout diversi nello stesso file. Da Bookworm in poi — quindi anche su Trixie,
che è la base scelta — la partizione di boot è montata in `/boot/firmware` e `/boot/cmdline.txt`
non esiste: la `sed -i` fallirebbe, e con `set -e` in cima al modulo farebbe fallire **l'intera
build**. Il problema di fondo è che il modulo usa path assoluti invece di `$BOOT_PATH`, che è la
variabile che CustoPiZer imposta apposta e che `10-config-raspberry` usa correttamente.

Non l'ho verificato contro una build reale — se la CI oggi passa, allora CustoPiZer monta la
partizione altrove e la mia lettura è sbagliata. **È la prima cosa da guardare nei log della
prossima build** (passo 7 di [05-piano-rebase.md](05-piano-rebase.md)). In ogni caso usare
`$BOOT_PATH` in entrambe le righe è la forma giusta, e va fatto a prescindere da come si risolve
il dubbio.

---

## 🟡 T5 — GingerView è clonato shallow ma registrato come `git_repo`

**Dove:** [`modules/generic/52-gingerview`](../modules/generic/52-gingerview) e
[`moonraker.conf`](../modules/generic/files/moonraker.conf)

```bash
git clone --depth 1 "${GINGERVIEW_REPO}" /home/pi/mainsail
```

mentre `moonraker.conf` dichiara:

```ini
[update_manager GingerView]
type: git_repo
path: ~/mainsail
```

L'update_manager `git_repo` di Moonraker ispeziona la storia del repository (tag, ref, commit
raggiungibili) e **un clone shallow gli risulta tipicamente non valido**: la voce comparirebbe
come "invalid" nell'interfaccia e non si aggiornerebbe. *Da verificare su una macchina reale.*

Da notare il contrasto con upstream: `52-mainsail` scarica uno **zip di release** e lo registra
come `type: web`. Noi facciamo git clone e lo registriamo come `git_repo` — è una combinazione
diversa, e la parte shallow è quella sospetta.

**Fix candidato:** togliere `--depth 1`. Costa qualche decina di MB nell'immagine, ma è ciò che
già fanno i moduli Kalico e Moonraker.

Collegato: `[update_manager GingerView]` non dichiara un `install_script`, quindi un aggiornamento
è un `git pull` e basta. Funziona **solo perché GingerView committa `build/`** su `main`
(verificato: 37 file tracciati sotto `build/`). È una dipendenza implicita fra due repository che
andrebbe scritta da qualche parte — idealmente nel README di GingerView.

---

## 🟡 T6 — `69-G2-Service` installa dipendenze sbagliate

**Dove:** [`modules/generic/69-G2-Service`](../modules/generic/69-G2-Service)

```bash
DEPS=(git python3-pip python3-flask)
```

G2-Service **non usa Flask**: `requirements.txt` è `fastapi` + `uvicorn[standard]`, installati in
un venv dedicato che `Scripts/install.sh` si crea da solo. `python3-flask` viene installato di
sistema e non lo usa nessuno.

`git` serve. `python3-pip` non serve al venv, che usa il proprio pip.

**Fix:** `DEPS=(git)`. Non tocca nulla di funzionante, toglie solo peso all'immagine.

> Nota correlata, **da non rompere**: `install.sh` ripiega su `virtualenv` perché `python3-venv`
> non è installato da nessun modulo di G2-OS, e `virtualenv` c'è solo perché lo installa
> `50-kalico`. È una dipendenza implicita fra due moduli: se un giorno Kalico smettesse di
> installare `virtualenv`, G2-Service smetterebbe di installarsi. Vale la pena aggiungere
> `python3-venv` alle DEPS di `69-G2-Service` e chiudere la questione.

---

## 🟢 T7 — Residui di Mainsail in `moonraker.conf`

```ini
cors_domains:
    https://my.mainsail.xyz      # ← non ci serve
    http://my.mainsail.xyz       # ← non ci serve
    http://*.local
    http://*.lan

[announcements]
subscriptions:
    mainsail                     # ← annunci di Mainsail dentro GingerView
```

Gli `[announcements]` in particolare fanno comparire in GingerView le notifiche del progetto
Mainsail. Da sostituire con nulla, o con una sottoscrizione nostra se un giorno ne avremo una.

---

## 🟢 T8 — `99-unhold-packages` non fa niente

L'unica riga operativa è commentata; il modulo esegue solo il boilerplate. È coerente con
`00-upgrade`, dove pure `apt-mark hold` è commentato, e con `HOLD_PKGS` in `00-config` che ha
quasi tutte le voci commentate.

Upstream ha **cancellato** modulo e variabile in v3.0.0. Il riallineamento lo risolve da sé: basta
non reintrodurli risolvendo i conflitti.

---

## 🟢 T9 — `.DS_Store` committato

`.DS_Store` (6 KB) è tracciato nella radice del repo (`git ls-files | grep DS_Store`).
Va rimosso dall'indice e aggiunto a `.gitignore`, che oggi ignora solo `src/workspace`,
`src/image`, `src/build.log` e `.vscode`. Upstream ha esteso il proprio `.gitignore` nel commit
`a0dc1ba`: conviene prendere quello e aggiungerci `.DS_Store`.

---

## 🔴 T12 — `rpi_json` non dichiara più nessun `init_format` — *da verificare sulla prima immagine*

**Dove:** [`config.yml`](../config.yml), dopo il riallineamento

Avendo deciso di adottare cloud-init ([Q8](Q&A.md)), il target avrà `INIT_FORMAT: cloudinit-rpi`
sotto `env` — è quello che decide quali moduli si installano nell'immagine.

Ma upstream, nello stesso commit, ha **rimosso** `init_format: "systemd"` da `rpi_json` **senza
sostituirlo** con `"cloudinit"`: nessuno dei sei target di upstream v3.0.0 dichiara più un
`init_format` (verificato con `git show upstream/develop:config.yml`).

`rpi_json` è il blocco che finisce nel `rpi-imager.json` pubblicato con la release, ed è come
l'immagine dice al **Raspberry Pi Imager** in che formato scrivere la personalizzazione (utente,
password, Wi-Fi, SSH). Il rischio: se in assenza di quella chiave l'Imager continua a scrivere un
`firstrun.sh` in stile systemd, l'immagine si aspetterebbe cloud-init e riceverebbe altro — e il
primo boot **non creerebbe l'utente**.

Tre spiegazioni possibili, e non posso distinguerle da qui:

1. L'Imager rileva da solo il formato e la chiave è superflua.
2. Upstream distribuisce l'immagine per altre vie e non se n'è accorto.
3. È una svista di upstream.

**Verifica:** flashare l'immagine con il RPi Imager impostando un username **diverso da `pi`** e
guardare cosa finisce sulla partizione di boot — `firstrun.sh` (systemd) o `user-data`
(cloud-init). Se è il primo, aggiungere `init_format: "cloudinit"` a `rpi_json`.

È lo stesso test del passo 8 di [05-piano-rebase.md](05-piano-rebase.md), che quindi copre due
cose insieme: questa e le personalizzazioni Kalico in `postrename-lib`.

---

## 🟢 T11 — `rpi_json` punta ad asset e testi di Mainsail

**Dove:** [`config.yml`](../config.yml)

```yaml
icon: "https://os.mainsail.xyz/rpi-imager.png"
description: "A port of Raspberry Pi OS with preinstalled Kalico/Moonraker/Mainsail for 3D printers"
```

Sono i metadati che il **Raspberry Pi Imager mostra all'utente**: l'icona è servita da un dominio
di terze parti che non controlliamo, e la descrizione nomina Mainsail, che non installiamo più.

Se l'immagine viene distribuita ai clienti va sistemato: icona ospitata da noi (o nelle release
GitHub del repo) e descrizione che nomina GingerView. Il momento naturale per farlo è mentre si
risolve il conflitto su `config.yml` durante il riallineamento.

---

## 🟢 T10 — Piccole cose

- **`61-EnableUSB`**: `echo "USB rule added to ${RULES_FILE}"` usa una variabile mai definita (si
  chiama `ENABLEUSB_RULE_FILE`). Solo cosmetico — il modulo non ha `set -u` — ma stampa una riga
  monca nei log di build.
- **`52-gingerview`**: `rm -f printer_data/config/mainsail.cfg` cancella un file che nessun modulo
  crea più (lo creava `52-mainsail` di upstream). Vestigiale, innocuo.
- **Prefissi numerici duplicati**: `60-Kamp`/`60-mainsailos` e `61-EnableUSB`/`61-postrename`.
  L'ordine dipende dall'ordinamento del nome. Nessuna dipendenza reale oggi, ma non aggiungerne
  altri.
- **`patches/`**: due script per MainsailOS 1.0.x che non c'entrano nulla con noi. Vedi
  [Q&A.md](Q&A.md) **Q5**.
- **Nessun modo di provare la build in locale**: manca uno script che replichi gli step della CI
  (creare `scripts/`, generare `config.local`, invocare CustoPiZer via Docker). Oggi l'unico
  modo di provare una modifica è pushare. Uno script `tools/build-local.sh` si ripagherebbe da solo.
