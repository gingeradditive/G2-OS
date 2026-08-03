# Documentazione G2-OS

Documentazione interna del repository che costruisce l'immagine di sistema della stampante
**Ginger G2**. Convenzione dell'ecosistema: codice e UI in inglese, documentazione in italiano.

Scritta il **03/08/2026** contro `develop` a `f994a31` e `upstream/develop` a `a607941`
(MainsailOS v3.0.0).

| Documento | Cosa contiene |
|---|---|
| [01-panoramica.md](01-panoramica.md) | Cos'è G2-OS, hardware bersaglio, cosa produce, cosa **non** deve produrre |
| [02-build-e-moduli.md](02-build-e-moduli.md) | Come funziona la build (CustoPiZer), `config.yml`, i moduli in ordine di esecuzione |
| [03-stack-sulla-macchina.md](03-stack-sulla-macchina.md) | Cosa gira sulla macchina finita: servizi, porte, percorsi, nginx, contratti con gli altri repo |
| [04-upstream-e-divergenza.md](04-upstream-e-divergenza.md) | Punto di fork, i nostri 14 commit, i 16 di upstream, mappa dei conflitti attesi |
| [05-piano-rebase.md](05-piano-rebase.md) | Il piano di riallineamento, passo per passo |
| [Q&A.md](Q&A.md) | Decisioni: prese (Q1, Q3, Q5, Q8) e ancora aperte (Q2, Q4, Q6, Q7) |
| [TODO.md](TODO.md) | Difetti e incongruenze trovate nel codice attuale, con priorità |

## Stato del riallineamento

Deciso il **03/08/2026**:

- **`git rebase`** su `upstream/develop` (non merge)
- **Debian Trixie** con `raspios_lite_arm64_latest`, come upstream
- **cloud-init** (`INIT_FORMAT: cloudinit-rpi`): il flusso `rc.local` → `/postrename` sparisce
- Si cancellano `modules/armbian/`, `modules/special/`, `patches/`; `devices` → solo `pi4-64bit`

Tutte le decisioni bloccanti sono prese: **il [piano](05-piano-rebase.md) è eseguibile così
com'è.** Le domande ancora aperte (Q2, Q4, Q6, Q7) non lo bloccano.

Il rebase **non è ancora stato eseguito**: `develop` è a `f994a31`, allineato a `origin`.

## Da dove partire

- **Devo capire cosa fa questo repo** → [01](01-panoramica.md), poi [02](02-build-e-moduli.md).
- **Devo toccare l'immagine** → [02](02-build-e-moduli.md) e [03](03-stack-sulla-macchina.md).
- **Devo eseguire il riallineamento** → [05](05-piano-rebase.md), con
  [04](04-upstream-e-divergenza.md) aperto di fianco per la mappa dei conflitti.

## Convenzioni di questi documenti

Ogni affermazione è di uno di questi tre tipi, e quando non è ovvio è marcata:

- **Verificato** — letto nel codice o nella cronologia git di questo checkout, o verificato
  contro il repository remoto (default: se non c'è marcatura, è verificato).
- **Da verificare** — dedotto ma non provato su hardware o in una build reale. Marcato esplicitamente.
- **Decisione aperta** — sta in [Q&A.md](Q&A.md), non va data per acquisita.

Non c'è nessuna build reale eseguita a supporto di questi documenti: tutto ciò che riguarda il
comportamento a runtime dell'immagine è dedotto dagli script.
