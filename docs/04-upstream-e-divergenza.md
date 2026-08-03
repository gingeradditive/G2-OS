# 04 — Upstream e divergenza

Fotografia al **03/08/2026**.

## Le coordinate

| | |
|---|---|
| Nostro `origin` | `git@github.com:gingeradditive/G2-OS.git`, branch `develop` |
| Nostro HEAD | `f994a31` — 30/07/2026 — *Remove unused 32-canbus and 62-PowerButton modules* |
| Upstream | `https://github.com/mainsail-crew/MainsailOS.git`, branch `develop` |
| Upstream HEAD | `a607941` — 14/05/2026 — *refactor: remove commented code and unused file (#364)* |
| **Punto di fork** | **`671218f`** — 30/10/2025 — *fix: armbian-motd (#353)* |
| Versione nostra / upstream | **2.0.5** / **3.0.0** |

Il remote `upstream` è già configurato in questo checkout. Per riprodurre le analisi qui sotto:

```bash
git fetch upstream develop --tags
git log --oneline 671218f..upstream/develop     # cosa ci manca
git log --oneline 671218f..develop              # cosa abbiamo aggiunto
```

Sono **14 nostri commit** contro **16 loro**. Divergenza contenuta in numero, ma non in impatto:
in mezzo ai loro 16 c'è un cambio di sistema operativo di base.

## I nostri 14 commit

```
f994a31  2026-07-30  Remove unused 32-canbus and 62-PowerButton modules
d525f0d  2026-04-23  replace Mainsail with GingerView + nginx serve build/ statica
5f33b94  2026-02-16  fix path eseguibile Kalico → kalico/klippy/klippy.py; socket moonraker
53ff5a2  2026-02-16  G2-Service installer → ./Scripts/install.sh
9a208bd  2026-02-16  requirements → klippy-requirements.txt
f24dc7f  2026-02-16  rimosso modulo KlipperScreen4pellet
9234cb5  2026-02-16  riferimenti Kalico → KalicoCrew, docs → docs.kalico.gg
aa48c07  2026-02-16  da Klipper a Kalico in tutto il progetto
9eb8aab  2026-02-16  fix: secret
a50fafe  2026-02-16  modulo G1Config → G2-Service
d3d6eff  2026-02-16  rimossa build armhf a 32 bit
bb890f5  2026-02-16  rebranding nei workflow GitHub, disabilitato il merge su master
e173bb8  2026-02-16  rebranding G1OS → G2-OS
2ed1b3d  2026-02-16  clonata la personalizzazione G1OS dentro G2OS
```

Da leggere così: **un blocco unico del 16/02/2026** che importa in un colpo solo tutta la
personalizzazione ereditata da G1OS e la ribattezza, seguito da tre interventi puntuali
(aprile: GingerView; luglio: pulizia).

Il fatto che i commit `9a208bd`, `5f33b94`, `53ff5a2` esistano è significativo: sono correzioni
di errori introdotti dalla sostituzione globale klipper→kalico di `aa48c07`, che ha rinominato
anche path di repository di terze parti che non andavano toccati. **Non tutte quelle correzioni
sono state fatte** — vedi il bug di `54-timelapse` in [TODO.md](TODO.md).

## I 16 commit di upstream che ci mancano

```
a607941  2026-05-14  refactor: remove commented code and unused file (#364)
f2a2003  2026-05-06  docs(changelog): update changelog
77ff5c1  2026-05-06  chore: bump version to v3.0.0
307573c  2026-05-02  fix(build): update architecture to armv7l
a0971d2  2026-05-02  ci(build): update runner to ubuntu-24.04-arm
7cc5ccc  2026-04-19  feat(cloudinit): add cloud-init init_format with pre/post-rename flow
426a216  2026-04-19  fix(raspberry): configure user, ssh, getty and passwordless sudo for Trixie
89e92cb  2026-04-19  feat(crowsnest): switch to v5 branch and drop legacy pkglist installer
1609ebe  2026-04-19  fix(klipper): adapt numpy and libatlas-base-dev for Trixie
457c5df  2026-04-19  feat(trixie)!: upgrade base images to Debian Trixie      ← BREAKING
4ab763d  2026-04-18  chore(workflow): update build & release workflow actions (#362)
a0dc1ba  2026-04-17  chore: update .gitignore (JetBrains, Clankers)
5f5c131  2026-02-28  fix(opi4lts): overlay_prefix RK3399                       ← non ci serve
76d059e  2026-02-28  fix(opi3lts): i2c e UART insieme a SPI                    ← non ci serve
71573e1  2026-02-28  fix: build wheels su immagini armhf                       ← non ci serve
d240cb6  2026-02-26  fix(opi-zero2): overlay SPI, I2C, UART                    ← non ci serve
```

Quattro dei sedici riguardano hardware che non usiamo. Il valore reale sta negli altri.

## Il nodo: v3.0.0 è il passaggio a Debian Trixie

`457c5df` è marcato `!` (breaking) e cambia la natura dell'immagine:

| | Fork point (v2.2.2) | Upstream oggi (v3.0.0) |
|---|---|---|
| Base | Raspberry Pi OS **Bookworm**, URL pinnato a una data | Raspberry Pi OS **Trixie**, URL `..._latest` (rolling) |
| Python di sistema | 3.11 | **3.13** |
| NumPy in Klipper | `numpy<1.26` | `numpy` senza vincolo (1.26 non supporta 3.13) |
| BLAS | `libatlas-base-dev` | assorbito in `libopenblas-dev`, installato solo se ancora in apt |
| Swap | `dphys-swapfile` | `rpi-swap` (drop-in in `/etc/rpi/swap.conf.d/`) |
| sudo/ssh/getty | impliciti nell'immagine base | configurati esplicitamente (`raspi-config nonint`, `userconf-pi`) |
| Primo boot | `firstrun.sh` + `rc.local` → `/postrename` | **cloud-init**, con servizi pre/post-rename |
| Runner CI | `ubuntu-22.04-arm` | `ubuntu-24.04-arm`, `checkout@v6`, `upload-artifact@v7`, CustoPiZer pinnato a `main` |

Le implicazioni per noi:

- **Il passaggio a Trixie non è obbligato dal riallineamento.** L'immagine base la sceglie
  `config.yml`, che è un **nostro** file. Possiamo assorbire tutti i miglioramenti dei moduli e
  restare su Bookworm pinnato. Gli script di upstream sono scritti per funzionare su entrambi:
  fanno detection (`if [[ -f /etc/dphys-swapfile ]] … elif dpkg-query -s rpi-swap`,
  `is_in_apt libatlas-base-dev`, `INIT_FORMAT="${EDITBASE_INIT_FORMAT:-systemd}"`).
- **Un'eccezione**: la modifica NumPy in `50-klipper` **non è condizionale**. Upstream installa
  `numpy` senza vincolo di versione anche su Bookworm. Se restiamo su Bookworm, la scelta se
  accettare NumPy 2.x con Kalico va presa consapevolmente ([Q&A.md](Q&A.md), **Q3**).
- **Il flusso cloud-init è opt-in**: `61-postrename` di upstream esce subito se
  `EDITBASE_INIT_FORMAT` vale `cloudinit`/`cloudinit-rpi`, e il default è `systemd`. Non
  impostando quella env, restiamo sul flusso `rc.local` che già conosciamo.

Rimane comunque un rischio reale, che nessuno può liquidare a tavolino: Trixie porta Python 3.13,
PEP 668 e nuove versioni delle librerie di sistema, e **Kalico, Moonraker, G2-Service e la SPA di
GingerView non sono mai stati provati lì**.

> **Decisione presa il 03/08/2026** ([Q3](Q&A.md)): si adotta **Trixie con
> `raspios_lite_arm64_latest`**, come upstream — massimo allineamento. Le due conseguenze
> accettate sono che l'immagine base diventa rolling (stessa `VERSION`, immagini potenzialmente
> diverse) e che i quattro componenti girano per la prima volta su Python 3.13. È il motivo per
> cui la prova su hardware del piano non è una formalità.

## Mappa dei conflitti attesi

File toccati da **entrambe** le parti dopo il punto di fork — ordinati per difficoltà.

### 🔴 Duri

| File | Il nostro lato | Il loro lato |
|---|---|---|
| `modules/generic/50-klipper` → **`50-kalico`** | Rinominato + repo/venv/requirements/service cambiati | +12 righe: numpy e libatlas per Trixie |
| `modules/raspberry/files/postrename` | 32 righe: Kalico, via KlipperScreen, guardia su `mainsail.cfg` | **313 righe**: logica estratta in `postrename-lib` condivisa con cloud-init |
| `config.yml` | Ridotto a 1 target, Bookworm pinnato, resto commentato | 6 target riscritti per Trixie + `INIT_FORMAT` |

Il rename di `50-klipper` è il caso classico in cui git può perdere la tracciabilità: con un
rebase la rilevazione del rename va rifatta a ogni commit replayato.

### 🟡 Medi

| File | Il nostro lato | Il loro lato |
|---|---|---|
| `modules/generic/files/00-config` | `DIST_NAME=G2-OS`, `HOLD_PKGS` quasi tutto commentato | `HOLD_PKGS` **cancellato**, aggiunto `BASE_PASSWORD` |
| `modules/generic/00-upgrade` | 2 righe (commento) | Rimosso il blocco `apt-mark hold` |
| `modules/raspberry/10-config-raspberry` | non toccato | **+64 righe**: swap moderno, `userconf-pi`, ssh, getty, sudo |
| `.github/workflows/build.yml` | 2 righe (rebranding) | runner, versioni action, `custopizer: main` |
| `.github/workflows/release.yml` | 40 righe (rebranding, merge su master disabilitato) | versioni action |
| `VERSION` | `2.0.5` | `3.0.0` |
| `CHANGELOG.md` | rigenerato per G2-OS (419 righe di diff) | 25 righe nuove |
| `README.md` | riscritto per G2-OS | non toccato |

`CHANGELOG.md` va risolto sempre a favore del nostro: è generato da `git-cliff` sulla nostra
storia, non ha senso fonderlo.

### 🟢 Facili o assenti

- `modules/generic/52-mainsail` → **`52-gingerview`**: upstream **non l'ha toccato** dopo il fork.
  Il rename passa senza attrito.
- `51-moonraker`, `53-crowsnest`: upstream ha cambiato, noi no → si prendono così come sono.
- `61-postrename`, `98-remove-passwordless-sudo`: idem.
- Tutti i moduli **solo nostri** (`57-Kiauh`, `58-Obico`, `60-Kamp`, `61-EnableUSB`,
  `63-SplashScreen`, `69-G2-Service`, `files/kalico.*`): nessun corrispettivo upstream, nessun
  conflitto.
- Tutta la roba Armbian / Orange Pi: conflitti solo se decidiamo di **cancellarla**. Se la
  lasciamo dov'è, i loro commit ci si applicano sopra senza attrito.

### File nuovi che arriverebbero da upstream

```
modules/generic/61-postrename-cloudinit                              ← ATTIVO (Q8)
modules/generic/files/cloudinit/{postrename,postrename-lib,prerename} ← ATTIVI (Q8)
modules/generic/files/cloudinit/mainsailos-{pre,post}rename.service   ← ATTIVI (Q8)

modules/armbian/13-armbian-cloudinit                                  ← si cancella (Q1)
modules/armbian/files/cloudinit/{99_mainsailos.cfg,meta-data,…}       ← si cancella (Q1)
modules/special/20-opi-3lts   (rinominato da 20-opi-3lts-spi)         ← si cancella (Q1)
modules/special/20-opi-4lts   (rinominato da 20-opi-4lts-spi)         ← si cancella (Q1)
```

Avendo deciso di adottare cloud-init ([Q8](Q&A.md)), i file sotto `modules/generic/` **non sono
inerti**: sono il flusso di primo boot dell'immagine. Quelli sotto `modules/armbian/` servono
alla variante `INIT_FORMAT: cloudinit` (Armbian) e non a `cloudinit-rpi`, quindi cancellarli non
rompe niente — le due decisioni Q1 e Q8 non collidono.

⚠️ **Il punto singolo che richiede più attenzione di tutto il riallineamento**:
`modules/generic/files/cloudinit/postrename-lib` è la libreria dove è finita la logica di
`postrename`, condivisa fra flusso legacy e cloud-init. Le nostre modifiche Kalico **vanno
riportate lì**, non lasciate in `modules/raspberry/files/postrename` — che con
`INIT_FORMAT: cloudinit-rpi` non viene nemmeno più installato, perché `61-postrename` esce
subito per via del gate.

Confrontando il nostro `postrename` con `postrename-lib`, le nostre quattro personalizzazioni
mappano così — mappatura completa e istruzioni operative in
[05-piano-rebase.md](05-piano-rebase.md#postrename--postrename-lib):

- `SERVICES` e `venvs`: da riapplicare (attenzione, upstream ha aggiunto `crowsnest-env`)
- blocchi KlipperScreen: da rimuovere
- guardia su `mainsail.cfg`: **già presente in upstream**, non serve fare niente
- 🟢 e `fix_timelapse_links` di upstream usa il corretto `klipper_macro/`: risolvendo verso
  upstream, il bug [T3](TODO.md) si chiude da solo

## Una nota sulla nomenclatura

Upstream usa ovunque `mainsailos` come identificatore interno (`/usr/local/lib/mainsailos/`,
`mainsailos-prerename.service`, `99_mainsailos.cfg`). Il nostro `60-mainsailos` deriva già
`DIST_NAME` da `00-config`, ma questi nuovi path sono **hardcoded**. Prendendoli così come sono ci
ritroveremmo `mainsailos` dentro un'immagine G2-OS. Non è un problema funzionale; è una scelta
tra coerenza del branding e minor attrito nei merge futuri ([Q&A.md](Q&A.md), **Q4**).
