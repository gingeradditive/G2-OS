# Q&A — Decisioni aperte

Domande a cui serve una risposta **prima** di procedere con il riallineamento a upstream, o che
cambiano cosa costruiamo. Stessa convenzione di GingerView: numerate e stabili, si aggiunge in
fondo e non si rinumera.

Stato al 03/08/2026: **Q1, Q3, Q5, Q8 decise** (Giacomo) — sono tutte quelle che bloccavano il
riallineamento. Restano aperte Q2, Q4, Q6, Q7, nessuna delle quali lo blocca.

---

## Q1 — Il repo deve costruire solo il Raspberry Pi 4 Model B? — ✅ **DECISA (03/08/2026)**

> **Decisione: opzione (a) piena.** Si cancellano `modules/armbian/`, `modules/special/` e i
> target commentati; `devices` si riduce a `pi4-64bit`.
>
> ⚠️ **Con il rebase la cancellazione va fatta DOPO, non prima** — vedi
> [05-piano-rebase.md](05-piano-rebase.md), sezione "Perché la pulizia va in coda".

Il vincolo dichiarato è "solo Pi 4 Model B". Il repo però porta ancora dietro:

- `rpi_json.devices: [pi3-64bit, pi4-64bit, pi5-64bit]` in `config.yml`
- cinque target Armbian/Orange Pi commentati in `config.yml`
- `modules/armbian/` (7 file) e `modules/special/` (5 file), mai eseguiti con `type: raspberry`
- sezioni `[pi0]`, `[pi2]`, `[pi3]` in `boot-config.txt`

**Cosa cambia in base alla risposta:** se cancelliamo tutto prima del merge, sparisce una fetta
consistente dell'area di conflitto con upstream — quattro dei sedici commit che ci mancano
riguardano solo Orange Pi. Se lo teniamo, i loro commit si applicano lì sopra senza attrito ma
continuiamo a portarci dietro codice morto a ogni giro.

**Opzioni:**
- **(a)** Cancellare `modules/armbian/`, `modules/special/`, i target commentati; ridurre `devices`
  a `pi4-64bit` e `boot-config.txt` a `[pi4]` + `[all]`.
- **(b)** Lasciare `devices` con pi3/pi4/pi5 (l'immagine ci gira davvero) ma cancellare Armbian.
- **(c)** Non toccare niente, minimizzare le differenze con upstream.

Propendo per **(b)**: Armbian è chiaramente morto per noi, mentre restringere `devices` a
`pi4-64bit` ha l'unico effetto di limitare quali schede il RPi Imager offre — utile se
l'immagine viene distribuita ai clienti, inutile se resta interna.

---

## Q2 — Kiauh, Obico, KAMP e timelapse servono sulla G2?

Quattro moduli sono arrivati con l'importazione da G1OS e nessuno rientra nel perimetro dichiarato
("GingerView, Kalico, G2-Service e servizi accessori"):

| Modulo | Cosa installa | Osservazioni |
|---|---|---|
| `57-Kiauh` | Clona KIAUH in `~/kiauh` | Installer interattivo per Klipper. Su una macchina di produzione è un modo per farsi disinstallare Kalico da sotto i piedi |
| `58-Obico` | `moonraker-obico` + servizio | Monitoraggio **cloud di terze parti**. Attivo di default su una macchina venduta a un cliente: da valutare anche lato privacy |
| `60-Kamp` | Klipper Adaptive Meshing & Purging | Macro per bed leveling. Ha senso su un piano a zone? |
| `54-timelapse` | Componente Moonraker per timelapse | **Oggi rotto**, vedi [TODO.md](TODO.md) **T3**. GingerView espone i timelapse? |

Ognuno aggiunge tempo di build, superficie di attacco e un repository esterno da cui la macchina
dipende al primo boot.

**Serve una risposta modulo per modulo.** Sospetto che almeno Kiauh e Obico vadano via, ma è una
decisione di prodotto, non tecnica.

---

## Q3 — Bookworm o Trixie? — ✅ **DECISA (03/08/2026)**

> **Decisione: opzione (c) — Trixie con `raspios_lite_arm64_latest`, come upstream.**
> Massimo allineamento. Prendiamo anche la modifica NumPy senza vincolo (sotto-domanda a1
> decade: su Python 3.13 `numpy<1.26` non è installabile, quindi non c'è scelta).
>
> **Conseguenze accettate, da tenere presenti durante la verifica:**
> - Kalico, Moonraker, G2-Service e GingerView girano per la prima volta su **Python 3.13**.
>   Nessuno dei quattro è mai stato provato lì: la prova su hardware del passo 6 non è
>   formalità.
> - `..._latest` è **rolling**: la stessa `VERSION` di G2-OS può produrre immagini diverse in
>   momenti diversi. Se un domani serve tracciabilità su un'immagine consegnata a un cliente,
>   la strada è pinnare l'URL a una data al momento della release, non ripensare il resto.
> - G2-Service crea il venv con `virtualenv`; su Trixie vale la pena verificare che
>   **PEP 668** non interferisca (vedi T6 in [TODO.md](TODO.md): conviene aggiungere
>   `python3-venv` alle sue DEPS).
> - Apre **Q8** (cloud-init), che prima non si poneva.

<details>
<summary>Analisi originale (mantenuta per riferimento)</summary>

Upstream v3.0.0 è passato a Raspberry Pi OS Trixie e a un URL `..._latest` (rolling). Noi siamo
su Bookworm con URL pinnato al 01/10/2025.

Come spiegato in [04](04-upstream-e-divergenza.md), **il merge non ci obbliga a seguirli**:
l'immagine base la sceglie `config.yml`, che è nostro, e gli script di upstream fanno detection
per funzionare su entrambe.

**Opzioni:**
- **(a) Restare su Bookworm pinnato.** Prendiamo i miglioramenti dei moduli senza cambiare SO.
  Rischio quasi nullo. Costo: fra un anno Bookworm sarà vecchio e il salto sarà più duro.
- **(b) Passare a Trixie con URL pinnato a una data.** Prendiamo tutto, ma su una base
  riproducibile invece che rolling.
- **(c) Passare a Trixie con `..._latest` come upstream.** Massimo allineamento, ma la stessa
  identica `VERSION` di G2-OS può produrre immagini diverse in momenti diversi. Su un prodotto
  che va da un cliente è un problema di tracciabilità serio.

Propendo per **(a) ora, (b) come lavoro successivo e separato**. Trixie porta Python 3.13 e
PEP 668, e **nessuno dei nostri quattro componenti — Kalico, Moonraker, G2-Service, GingerView —
è mai stato provato lì**. Mescolarlo al riallineamento significa che, se la prima immagine non
parte, non sappiamo di chi è la colpa.

**Sotto-domanda (a1), da rispondere comunque:** se restiamo su Bookworm, accettiamo la modifica
NumPy di upstream (`numpy` senza vincolo, invece di `numpy<1.26`)? Quella riga **non è
condizionale**, quindi ci arriva addosso comunque. Su Python 3.11 installerebbe NumPy 2.x, che
Kalico potrebbe non gradire. La via prudente è tenere `numpy<1.26` finché restiamo su Bookworm,
e far scattare la modifica insieme al passaggio a Trixie.

</details>

---

## Q4 — Rinominare i path `mainsailos` che arrivano da upstream?

Il flusso cloud-init di upstream introduce path hardcoded: `/usr/local/lib/mainsailos/`,
`mainsailos-prerename.service`, `mainsailos-postrename.service`, `99_mainsailos.cfg`.

Uno di questi ci riguarda **anche restando sul flusso legacy**:
`/usr/local/lib/mainsailos/postrename-lib` è installato pure da `61-postrename`.

**Opzioni:**
- **(a)** Lasciare `mainsailos` nei path interni. Zero attrito nei merge futuri, ma dentro
  un'immagine G2-OS compare il nome di un altro progetto.
- **(b)** Rinominare in `g2-os`. Coerente col branding, ma crea un conflitto permanente su ogni
  file di quel flusso, per sempre.

Propendo per **(a)**: sono path interni che nessun utente vede, e il costo di (b) si paga a ogni
riallineamento. Il branding visibile (hostname, `/etc/g2-os-release`, nome dell'immagine) è già
nostro e resta tale.

---

## Q5 — Cosa fare della cartella `patches/`? — ✅ **DECISA (03/08/2026)**

> **Decisione: cancellare.** Upstream non l'ha toccata dal punto di fork (0 commit), quindi la
> cancellazione non genera conflitti in nessun ordine. Va nello stesso commit di pulizia di Q1.

`patches/patch101.sh` e `patches/udev-fix.sh` sono patch una-tantum che gli utenti MainsailOS
lanciavano via `curl | bash` su immagini già installate: una per MainsailOS 1.0.0→1.0.1, una per
un bug di udev su Debian Bullseye. Non fanno parte della build, e il loro `Readme.md` rimanda a
URL di `mainsail-crew/MainsailOS`.

Direi **cancellarli**. Sono documentazione fuorviante per chiunque apra il repo: sembrano
strumenti nostri e non lo sono.

---

## Q6 — Rinominare `~/mainsail` in `~/gingerview`?

GingerView è clonato in `/home/pi/mainsail`. Il nome è referenziato da tre posti (site nginx,
`[update_manager GingerView]` in `moonraker.conf`, symlink dei log).

Rinominarlo è fattibile, ma **ha un costo di migrazione sulle macchine già in campo**:
l'update_manager di Moonraker punterebbe a una directory che lì non esiste, e va gestito a mano.

Da fare solo se sappiamo quante G2 sono già installate e se accettiamo una procedura di
migrazione. Non è lavoro da mescolare al riallineamento.

---

## Q7 — Quale sistema di aggiornamento per G2-Service in campo?

`moonraker.conf` non ha un blocco `[update_manager G2-Service]`, quindi **la macchina non può
aggiornare G2-Service**. Al tempo stesso `G2-Service/Scripts/install.sh` è scritto esplicitamente
per essere idempotente e usabile come `install_script` di Moonraker — la funzionalità è pronta ma
non collegata.

L'ostacolo vero è che **G2-Service è un repository privato**: l'update_manager di Moonraker
dovrebbe autenticarsi per fare `git pull`, e mettere un token dentro l'immagine è una scelta con
implicazioni sue.

**Opzioni:**
- **(a)** Rendere pubblico G2-Service, poi aggiungere il blocco `[update_manager]`. Semplice, ma è
  una decisione aziendale.
- **(b)** Tenerlo privato e usare un deploy key read-only nell'immagine. Funziona, ma la chiave è
  estraibile da chiunque abbia la SD in mano.
- **(c)** Distribuire gli aggiornamenti reimmaginando la SD. Nessun rischio, massimo attrito.

Aperta anche in `G2-Service/docs/`, ma la parte che tocca G2-OS è questa.

---

## Q8 — Adottiamo anche il flusso cloud-init? — ✅ **DECISA (03/08/2026)**

> **Decisione: opzione (a) — sì, `INIT_FORMAT: cloudinit-rpi`.** Coerente con Q3: se prendiamo
> Trixie per allinearci, prenderlo con un flusso di primo boot che upstream esercita meno
> sarebbe il peggio dei due mondi.
>
> **Cosa comporta concretamente**, verificato leggendo `61-postrename-cloudinit`:
> - `modules/raspberry/61-postrename` (legacy) **esce subito** per via del gate su
>   `INIT_FORMAT`: `/postrename` e l'aggancio a `rc.local` non esistono più.
> - `modules/generic/61-postrename-cloudinit` diventa attivo e installa:
>   `/usr/local/sbin/mainsailos-{pre,post}rename`, `/usr/local/lib/mainsailos/postrename-lib`,
>   `/var/lib/mainsailos/`, e due unit `mainsailos-prerename.service` /
>   `mainsailos-postrename.service`, entrambe abilitate.
> - Due pacchetti nuovi nell'immagine: **`cloud-init`** e `python3-yaml`.
> - I placeholder `@BASE_USER@` e `@BOOT_REQUIRE@` sono sostituiti a build time.
>
> 🟢 **Non è in conflitto con la cancellazione di `modules/armbian/`** (Q1): il modulo e i file
> del flusso cloud-init che ci riguardano stanno in `modules/generic/`. Quelli sotto
> `modules/armbian/files/cloudinit/` servono solo a `INIT_FORMAT: cloudinit` (variante Armbian),
> non a `cloudinit-rpi`. Verificato sull'albero di upstream.
>
> ⚠️ **Da verificare per primo:** upstream ha **rimosso** `init_format: "systemd"` da `rpi_json`
> senza sostituirlo con `"cloudinit"` — nessuno dei sei target di upstream dichiara più un
> `init_format` al Raspberry Pi Imager. Se l'Imager, in assenza di quella chiave, continua a
> scrivere un `firstrun.sh` in stile systemd, l'immagine si aspetterebbe cloud-init e
> riceverebbe altro: il primo boot non creerebbe l'utente. Non ho modo di verificarlo da qui —
> è il primo controllo da fare sulla prima immagine, vedi [T12](TODO.md).

<details>
<summary>Analisi originale (mantenuta per riferimento)</summary>

Prima di decidere Q3 questa domanda non si poneva: restando su Bookworm si restava anche sul
flusso legacy. Avendo scelto il massimo allineamento, va decisa.

Il target `raspberry_pi-arm64-trixie` di upstream imposta:

```yaml
env:
  INIT_FORMAT: cloudinit-rpi
rpi_json:
  # init_format: "systemd"   ← rimosso
```

Con quella riga, il primo boot cambia completamente:

| | Legacy (oggi) | cloud-init (upstream v3) |
|---|---|---|
| Chi crea l'utente | `firstrun.sh` scritto dal RPi Imager | `cloud-init` da `user-data` sulla partizione di boot |
| Chi ripara i path | `rc.local` → `/postrename` | `mainsailos-prerename.service` + `mainsailos-postrename.service` |
| Dipendenze nuove | — | pacchetti `cloud-init` e `python3-yaml` |

**Opzioni:**
- **(a) Adottarlo** (`INIT_FORMAT: cloudinit-rpi`). È il percorso che upstream ha effettivamente
  provato su Trixie; il flusso legacy su Trixie è supportato dal codice ma non è la loro strada
  principale. Coerente con la scelta "massimo allineamento" di Q3.
- **(b) Restare sul legacy**, non impostando `INIT_FORMAT` (il default di `61-postrename` è
  `systemd`). Cambia una variabile in meno rispetto a oggi, ma ci mette su un percorso che
  upstream esercita meno.

Propendo per **(a)**, per coerenza con Q3: se prendiamo Trixie per allinearci, prendere Trixie
con un flusso di primo boot diverso da quello che upstream prova è il peggio dei due mondi.

**Ma è, insieme a Python 3.13, la superficie non provata più grande di tutto il riallineamento**,
ed è esattamente dove vivono le nostre personalizzazioni Kalico. Il test "primo boot con username
diverso da `pi`" del passo 6 di [05-piano-rebase.md](05-piano-rebase.md) smette di essere
opzionale: è **la** verifica che dice se il riallineamento ha funzionato.

In entrambi i casi le personalizzazioni Kalico vanno in `postrename-lib`, che è condiviso fra i
due flussi: la scelta non cambia *cosa* modifichiamo, cambia *quale strada* viene esercitata al
primo boot.

</details>
