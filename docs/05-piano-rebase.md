# 05 — Piano di riallineamento a upstream

Presupposto: hai letto [04-upstream-e-divergenza.md](04-upstream-e-divergenza.md).
Stato di partenza: `develop` locale **identico** a `origin/develop` (`f994a31`), working tree
pulito, remote `upstream` già configurato.

## Le decisioni prese (03/08/2026)

| | Scelta | Dettagli |
|---|---|---|
| Strategia | **`git rebase`** | Storia lineare. Force-push su `develop` da concordare con Jack_up |
| Immagine base | **Trixie, `raspios_lite_arm64_latest`** | Massimo allineamento a upstream ([Q3](Q&A.md)) |
| Pulizia | `modules/armbian/`, `modules/special/`, `patches/`, `devices` → solo `pi4-64bit` | ([Q1](Q&A.md), [Q5](Q&A.md)) |
| cloud-init | **sì, `INIT_FORMAT: cloudinit-rpi`** | ([Q8](Q&A.md)) Il flusso legacy `rc.local` → `/postrename` sparisce |

Tutte le decisioni che bloccavano il riallineamento sono prese: **il piano è eseguibile così
com'è.**

> **Nota sulla strategia.** Avevo raccomandato il merge, per due ragioni: i conflitti su
> `50-kalico` e `postrename` si ripresentano a ogni commit replayato, e i riallineamenti futuri
> ripagano lo stesso prezzo. La scelta è stata il rebase e il piano qui sotto è scritto per il
> rebase — con `rerere` attivo il costo si riduce parecchio. La sezione
> [Se invece si sceglie il merge](#se-invece-si-sceglie-il-merge) resta in fondo.

## Perché la pulizia va in coda

Questo è il punto in cui il piano cambia rispetto a com'era scritto per il merge, e vale la pena
capirlo prima di eseguire.

Con un **merge**, cancellare prima riduce davvero i conflitti: un file cancellato da noi e
modificato da loro dà un conflitto `deleted by us`, banale da risolvere con `git rm`, una volta sola.

Con un **rebase** no. Un commit di pulizia creato adesso finisce comunque **in cima** alla
sequenza, quindi viene replayato **per ultimo**: tutti i nostri 14 commit precedenti continuano
ad applicarsi contro un albero in cui `modules/armbian/` e `modules/special/` ci sono ancora, e i
conflitti con le modifiche upstream a quei path si presentano identici. Pulire prima non evita
nulla e aggiunge un commit in più da replayare.

Verificato su questo repo:

```
upstream ha toccato dal fork:   patches/ → 0 commit
                                modules/armbian/ → 1 commit
                                modules/special/ → 3 commit
nostri commit che toccano quei path: 2ed1b3d, e173bb8, aa48c07
```

Quindi:

- **`patches/`** — nessuno dei due lati in conflitto con l'altro (upstream: 0 commit). Si può
  cancellare quando si vuole.
- **`modules/armbian/`, `modules/special/`** — toccati da entrambi. I conflitti durante il replay
  sono **inevitabili**, ma sono anche **gratis da risolvere**: quei file vengono cancellati alla
  fine, quindi a ogni stop basta prendere una versione qualsiasi e andare avanti (`git checkout
  --theirs .` o `git rm`, indifferente).

**Conclusione: un unico commit di pulizia, dopo il rebase, scritto contro l'albero finale.**

## Il piano

### Passo 0 — Rete di sicurezza

Con un rebase questo passo non è opzionale: stiamo per riscrivere storia già pubblicata.

```bash
git branch backup/pre-rebase-2026-08-03 develop
git push origin backup/pre-rebase-2026-08-03
```

### Passo 1 — Attivare rerere

```bash
git config rerere.enabled true
```

Git registra come risolvi ogni conflitto e riapplica la stessa risoluzione da solo quando lo
rivede. È **la** cosa che rende sopportabile un rebase con conflitti ripetuti su
`50-kalico` e `postrename`. Da fare prima di iniziare, non a metà.

### Passo 2 — Cosa comporta cloud-init (deciso, [Q8](Q&A.md))

Niente da decidere, ma da sapere prima di risolvere i conflitti, perché cambia quali moduli
finiscono nell'immagine:

- `modules/raspberry/61-postrename` (legacy) **esce subito** per via del gate su `INIT_FORMAT`.
  `/postrename` e l'aggancio a `rc.local` **non esistono più** nell'immagine.
- `modules/generic/61-postrename-cloudinit` diventa attivo e installa
  `/usr/local/sbin/mainsailos-{pre,post}rename`, `/usr/local/lib/mainsailos/postrename-lib`,
  `/var/lib/mainsailos/`, più le unit `mainsailos-prerename.service` e
  `mainsailos-postrename.service`, entrambe abilitate.
- Due pacchetti nuovi: **`cloud-init`** e `python3-yaml`.

🟢 **Non collide con la cancellazione di `modules/armbian/`** del passo 5: il modulo e i file che
ci servono stanno in `modules/generic/`. Quelli sotto `modules/armbian/files/cloudinit/` servono
solo alla variante `INIT_FORMAT: cloudinit` (Armbian), non a `cloudinit-rpi`.

⚠️ Apre però [T12](TODO.md): `rpi_json` non dichiara più nessun `init_format` al Raspberry Pi
Imager. Da verificare al passo 8, è il primo controllo sulla prima immagine.

### Passo 3 — Aggiornare upstream

```bash
git fetch upstream develop --tags
```

### Passo 4 — Il rebase

```bash
git checkout -b chore/align-upstream-v3 develop
git rebase upstream/develop
```

Regole di risoluzione, decise in anticipo per non doverci pensare a conflitto aperto:

| File | Come risolvere |
|---|---|
| `modules/armbian/**`, `modules/special/**` | **Indifferente** — si cancella tutto al passo 5. Prendere una versione qualsiasi e proseguire |
| `patches/**` | Idem |
| `CHANGELOG.md` | **Il nostro**. È generato da git-cliff sulla nostra storia, fonderlo non ha senso |
| `README.md` | **Il nostro** |
| `VERSION` | **Il nostro** (`2.0.5`). La numerazione del fork è indipendente |
| `.github/workflows/*.yml` | **Base il nostro** (rebranding, merge su master disabilitato), riportando a mano da upstream: runner `ubuntu-24.04-arm`, `checkout@v6`, `upload-artifact@v7`, `custopizer: main` |
| `modules/generic/files/00-config` | **Base il loro** (`HOLD_PKGS` via, `BASE_PASSWORD` dentro), rimettere `DIST_NAME=G2-OS` |
| `config.yml` | **Manuale** — vedi sotto |
| `50-kalico` vs `50-klipper` | **Manuale** — vedi sotto |
| `files/postrename` / `postrename-lib` | **Manuale, il caso più delicato** — vedi sotto |
| Tutto il resto | **Il loro** |

#### `config.yml`

Si parte dal target `raspberry_pi-arm64-trixie` di upstream e lo si adatta. Risultato atteso —
**un solo target**, tutto il resto cancellato:

```yaml
- name: raspberry_pi-arm64-trixie
  type: raspberry
  img_download_url: "https://downloads.raspberrypi.org/raspios_lite_arm64_latest.torrent"
  img_hash_url: "https://downloads.raspberrypi.org/raspios_lite_arm64_latest.sha256"
  env:
    ARCH: arm64
    IMAGE_ENLARGEROOT: 7000        # ← il NOSTRO valore, non i 6000 di upstream
    MOUNT_PROC: 1
    MOUNT_SYS: 1
    INIT_FORMAT: cloudinit-rpi     # ← Q8
  special_modules: []
  rpi_json:
    name: "G2-OS $VERSION$ - Ginger G2 (Raspberry Pi 4)"
    description: "Immagine di sistema della stampante Ginger G2 — Kalico, Moonraker, GingerView"
    icon: "https://os.mainsail.xyz/rpi-imager.png"     # ← da sostituire, vedi T11 in TODO.md
    devices:
      - "pi4-64bit"                # ← solo Pi 4 (Q1)
  # init_format: ← vedi T12, potrebbe servire "cloudinit" qui dentro
```

Tre dettagli da non perdere:

- `IMAGE_ENLARGEROOT` deve restare **7000**: installiamo più roba di upstream, che usa 6000.
- `init_format: "systemd"` sparisce da `rpi_json` — upstream l'ha spostato in `env` come
  `INIT_FORMAT`. **Ma non l'ha rimpiazzato con `"cloudinit"`**, ed è il dubbio di
  [T12](TODO.md): se al passo 8 si scopre che l'Imager scrive ancora `firstrun.sh`, va aggiunto
  `init_format: "cloudinit"` qui.
- Il nome del target cambia da `...-bookworm` a `...-trixie`, e finisce nel nome del file
  immagine prodotto (`<data>-G2-OS-raspberry_pi-arm64-trixie-<VERSION>.img.xz`).

#### `50-kalico`

Tenere il nostro file e portarci dentro le due modifiche Trixie di upstream:

```bash
# libatlas-base-dev è confluito in libopenblas-dev su Trixie
DEPS=(... python3-numpy python3-matplotlib libopenblas-dev)
if [ "$(is_in_apt libatlas-base-dev)" -eq 1 ]; then
  DEPS+=(libatlas-base-dev)
fi

# numpy<1.26 non supporta Python 3.13
sudo -u "${BASE_USER}" "${ENV_DIR}"/bin/pip install "numpy"
```

Se git vede delete+add invece di un rename:

```bash
git checkout --ours modules/generic/50-kalico
git show upstream/develop:modules/generic/50-klipper > /tmp/50-klipper-upstream
diff /tmp/50-klipper-upstream modules/generic/50-kalico
```

#### `postrename` → `postrename-lib`

Upstream ha svuotato `files/postrename` e spostato la logica in
`modules/generic/files/cloudinit/postrename-lib`, che `61-postrename` installa in
`/usr/local/lib/mainsailos/postrename-lib` **anche nel flusso legacy**. Quindi le nostre
personalizzazioni non vanno più in `files/postrename`: vanno **riportate in `postrename-lib`**.

Ho confrontato il nostro `postrename` con `postrename-lib` di upstream. Le nostre quattro
modifiche mappano così:

| Nostra modifica | Dove, in `postrename-lib` | Azione |
|---|---|---|
| `SERVICES`: `klipper` → `kalico` | riga ~12 | **Da riapplicare** |
| `venvs`: `klippy-env` → `kalico-env` | riga ~90 | **Da riapplicare.** Attenzione: upstream ha aggiunto `crowsnest-env` alla lista — va tenuto |
| Rimozione blocchi KlipperScreen | righe ~18-25 e ~119-126 | **Da riapplicare** |
| Guardia `if [[ -f … ]]` in `fix_mainsailcfg_links` | riga ~163 | **Già presente in upstream.** Non serve fare niente |

🟢 **Nota positiva:** `fix_timelapse_links` di upstream usa `klipper_macro/`, che è il nome
corretto della directory. Risolvendo il conflitto **verso upstream** su quella riga, il bug
[T3](TODO.md) si chiude da solo. **Non riapplicarci sopra il nostro `kalico_macro`** — è
esattamente l'errore da cui il bug è nato.

Prima di toccarlo, leggere `postrename-lib` per intero: è un file nuovo, con un wrapper
`run_step` e la sostituzione del placeholder `@BASE_USER@` a build time, non una versione
modificata di quello che conosciamo.

### Passo 5 — Il commit di pulizia

Un commit unico, subito dopo il rebase, contro l'albero finale:

```bash
git rm -r modules/armbian modules/special patches
git rm --cached .DS_Store
# aggiungere .DS_Store a .gitignore (upstream l'ha esteso in a0dc1ba)
git commit -m "chore: remove Armbian, Orange Pi and legacy patch files

Il target è solo il Raspberry Pi 4 Model B: modules/armbian/ non viene mai
eseguito con type: raspberry, modules/special/ non viene mai usato perché
special_modules è [], e patches/ contiene patch una-tantum per MainsailOS 1.0.x."
```

`devices` ridotto a `pi4-64bit` è già stato fatto risolvendo `config.yml` al passo 4.

### Passo 6 — Verifiche statiche

```bash
# 1. Sintassi di tutti gli script
for f in modules/generic/[0-9]* modules/raspberry/[0-9]*; do bash -n "$f" || echo "FAIL $f"; done

# 2. Nessun riferimento a Klipper rimasto dove serve Kalico
grep -rn "klippy-env\|klipper.service\|/klipper\b" modules/

# 3. …e nessun kalico_macro reintrodotto per sbaglio (bug T3)
grep -rn "kalico_macro" modules/ && echo "!! T3 reintrodotto"

# 4. config.yml valido e con un solo target
python3 -c "import yaml; c=yaml.safe_load(open('config.yml')); print(len(c), c[0]['name'], c[0]['rpi_json']['devices'])"

# 5. I nostri moduli sono sopravvissuti
ls modules/generic/52-gingerview modules/generic/69-G2-Service modules/generic/files/kalico.service
```

### Passo 7 — La build

```bash
git push origin chore/align-upstream-v3
gh workflow run "Build Images" --ref chore/align-upstream-v3
```

**La build deve girare su un branch, non su `develop`.** Serve il secret per il clone di
G2-Service (repo privato): se non è disponibile su branch diversi da `develop`, va sistemato
prima, altrimenti il fallimento di `69-G2-Service` maschera qualsiasi altro problema.

Nei log, guardare in particolare:

- `50-kalico`: quale versione di NumPy viene installata su Python 3.13, e se `klippy-requirements.txt`
  si installa pulito
- `61-postrename`: deve stampare `Skipping legacy rc.local post-rename hook
  (INIT_FORMAT=cloudinit-rpi)` ed uscire — se non lo fa, `INIT_FORMAT` non sta arrivando nel chroot
- `61-postrename-cloudinit`: deve invece stampare `Installing cloud-init pre/post-rename services`
- `63-SplashScreen`: se la `sed` su `cmdline.txt` passa (è il sospetto [T4](TODO.md))
- `69-G2-Service`: se `virtualenv` funziona su Trixie con PEP 668

I primi due sono l'unico modo, prima di avere hardware sotto mano, di verificare che
`config.yml` → `config.local` → `EDITBASE_INIT_FORMAT` funzioni end-to-end.

### Passo 8 — Prova sull'hardware

Un'immagine che compila non è un'immagine che funziona, e con il salto a Trixie questo passo pesa
più del solito: **nessuno dei nostri quattro componenti è mai stato provato su Python 3.13.**

- [ ] Boot completo, hostname `g2-os`
- [ ] `systemctl status kalico moonraker nginx crowsnest sonar g2-service` tutti attivi
- [ ] Kalico parte davvero su Python 3.13 (guardare `kalico.log`, non solo lo stato del servizio)
- [ ] GingerView risponde su `http://<ip>/` e il routing SPA funziona (ricaricare una sotto-pagina)
- [ ] WebSocket Moonraker connesso
- [ ] `/service/` — **oggi non è raggiungibile**, vedi [T1](TODO.md)
- [ ] UART: Kalico vede l'MCU
- [ ] Wi-Fi configurabile da GingerView
- [ ] **Primo boot con username diverso da `pi`** ← la verifica che conta di più. Copre due cose
      insieme: che il RPi Imager scriva la personalizzazione nel formato che l'immagine si
      aspetta ([T12](TODO.md) — guardare se sulla partizione di boot finisce `user-data` o
      `firstrun.sh`), e che le nostre personalizzazioni Kalico in `postrename-lib` funzionino.
      È anche la parte che il riallineamento riscrive interamente

### Passo 9 — Push su `develop`

```bash
git checkout develop
git reset --hard chore/align-upstream-v3
git push --force-with-lease origin develop
```

**Mai `--force` secco**: `--force-with-lease` rifiuta il push se nel frattempo qualcuno ha
pubblicato qualcosa, `--force` glielo cancella sopra.

Da concordare con Jack_up prima, non dopo: 13 dei 14 commit sono suoi e chiunque abbia il repo
clonato dovrà fare `git reset --hard origin/develop`.

Poi aggiornare [04-upstream-e-divergenza.md](04-upstream-e-divergenza.md) con il nuovo punto di
allineamento.

## Se invece si sceglie il merge

Cambia il passo 4 (`git merge upstream/develop`, conflitti una volta sola, push normale al passo
9) e conviene invertire il passo 5: con il merge, cancellare `modules/armbian/`,
`modules/special/` e `patches/` **prima** riduce davvero l'area di conflitto. Tutto il resto —
regole di risoluzione, `postrename-lib`, verifiche — resta identico.

## Cosa non fa questo piano

Non aggiorna la versione e **non corregge i difetti** elencati in [TODO.md](TODO.md) — con
l'eccezione di T3, che si chiude da sé prendendo la versione upstream di `fix_timelapse_links`.

L'obiettivo è riportare la base allo stato di upstream cambiando **solo** ciò che il salto a
Trixie impone. Ogni altra modifica mescolata qui dentro rende impossibile capire, se la prima
immagine non parte, se la colpa è del riallineamento, di Trixie, o nostra. I fix di T1, T2, T4,
T5, T6 vanno in commit separati, dopo che la build passa.
