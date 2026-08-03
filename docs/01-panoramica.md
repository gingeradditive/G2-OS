# 01 — Panoramica

## Cos'è G2-OS

G2-OS è il repository che **costruisce l'immagine SD della Ginger G2**. Non è un'applicazione: è
una collezione di script di personalizzazione che vengono eseguiti dentro il chroot di
un'immagine Raspberry Pi OS ufficiale, e che ne producono una versione con tutto il software
della macchina già installato e abilitato.

È un **fork di [MainsailOS](https://github.com/mainsail-crew/MainsailOS)** (a sua volta basato
su CustomPiOS / CustoPiZer), con tre sostituzioni fondamentali rispetto all'originale:

| MainsailOS | G2-OS | Perché |
|---|---|---|
| Klipper | **Kalico** (`KalicoCrew/kalico`) | Fork di Klipper, necessario per l'estrusione a pellet |
| Mainsail | **GingerView** (`gingeradditive/GingerView`) | Interfaccia operatore della G2 |
| — | **G2-Service** (`gingeradditive/G2-Service`) | Backend locale della macchina (Wi-Fi, timezone, config) |

Il resto dello stack MainsailOS (Moonraker, Crowsnest, Sonar, nginx) è ereditato quasi
inalterato.

## Hardware bersaglio

**Solo Raspberry Pi 4 Model B**, 64 bit.

Questo è il vincolo che guida ogni scelta di semplificazione in questo repo. Upstream distribuisce
un ventaglio di immagini che a noi non serve:

- Raspberry Pi 1/2 a 32 bit (armhf)
- Orange Pi 3 LTS, 4 LTS, Zero2, Zero3 (Armbian)
- BigTreeTech CB1 (Armbian, build-only)

**Stato attuale del repo rispetto a questo vincolo:** parzialmente allineato.
[`config.yml`](../config.yml) costruisce una sola immagine (`raspberry_pi-arm64-bookworm`), e le
varianti Armbian sono commentate — ma:

- il target dichiara ancora `pi3-64bit`, `pi4-64bit`, `pi5-64bit` in `rpi_json.devices`;
- [`modules/armbian/`](../modules/armbian/) e [`modules/special/`](../modules/special/) sono
  ancora nel repo pur non venendo mai eseguiti per `type: raspberry`;
- [`modules/raspberry/files/boot-config.txt`](../modules/raspberry/files/boot-config.txt)
  contiene ancora le sezioni `[pi0]`, `[pi2]`, `[pi3]`.

È zavorra innocua ma è anche superficie di conflitto gratuita a ogni riallineamento con upstream.

> ✅ **Deciso il 03/08/2026** ([Q1](Q&A.md)): si cancella tutto — `modules/armbian/`,
> `modules/special/`, i target commentati — e `devices` si riduce a `pi4-64bit`. Avviene nel
> commit di pulizia del [piano di riallineamento](05-piano-rebase.md), **dopo** il rebase.

## Cosa produce la build

Una CI GitHub Actions ([`build.yml`](../.github/workflows/build.yml)) che, a ogni push su
`develop`, scarica l'immagine Raspberry Pi OS Lite ufficiale, ci esegue dentro tutti i moduli, e
carica come artifact:

```
<data>-G2-OS-raspberry_pi-arm64-bookworm-<VERSION>.img.xz
<data>-G2-OS-raspberry_pi-arm64-bookworm-<VERSION>.img.xz.sha256
<data>-G2-OS-raspberry_pi-arm64-bookworm-<VERSION>.img.sha256
```

La release ([`release.yml`](../.github/workflows/release.yml)) fa la stessa cosa ma è manuale
(`workflow_dispatch`), scrive il numero di versione in [`VERSION`](../VERSION), crea il tag,
genera il changelog con `git-cliff` e pubblica anche un `rpi-imager.json` per il Raspberry Pi
Imager.

Versione attuale del fork: **2.0.5**. Numerazione **indipendente** da quella di upstream, che è
già a 3.0.0 — è una divergenza voluta, non un ritardo. Vedi
[04-upstream-e-divergenza.md](04-upstream-e-divergenza.md).

## Cosa G2-OS non è

- **Non contiene la configurazione della stampante.** `printer.cfg` e i config Kalico
  (`Configs/main.cfg`, `macro.cfg`) vivono in **G2-Service**, che li installa. G2-OS deposita
  solo `moonraker.conf`.
- **Non compila GingerView.** Il modulo `52-gingerview` clona il repo e si aspetta di trovare la
  cartella `build/` **già committata** su `main`. La build dell'SPA avviene in GingerView, non qui.
  Vedi [03-stack-sulla-macchina.md](03-stack-sulla-macchina.md).
- **Non è pubblicabile senza credenziali.** `G2-Service` è un repository privato: il modulo
  `69-G2-Service` richiede `GITHUB_TOKEN` o una chiave SSH nel chroot, altrimenti la build fallisce.

## Il posto di G2-OS nell'ecosistema

```
                     ┌────────────────────────────────┐
                     │            G2-OS               │
                     │   (questo repo — build time)   │
                     └───────────────┬────────────────┘
                                     │ clona e installa
        ┌────────────┬───────────────┼───────────────┬──────────────┐
        ▼            ▼               ▼               ▼              ▼
    ┌────────┐  ┌──────────┐   ┌──────────┐   ┌───────────┐  ┌───────────┐
    │ Kalico │  │ Moonraker│   │GingerView│   │G2-Service │  │ Crowsnest │
    │        │  │          │   │  (SPA)   │   │           │  │  + Sonar  │
    └────────┘  └──────────┘   └──────────┘   └───────────┘  └───────────┘
         ▲            │              │              │
         └── UDS ─────┘              └── nginx :80 ─┘
```

Dettaglio di porte, percorsi e contratti in
[03-stack-sulla-macchina.md](03-stack-sulla-macchina.md).
