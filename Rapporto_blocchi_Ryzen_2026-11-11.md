# Blocchi completi Ryzen — analisi indipendente (prima lettura)

Data: 11 novembre 2026
Sistema: ACEMAGIC Warlord W1, Ryzen 7 8745HS, Radeon 780M, 32 GB, Ubuntu 24.04.4

## Limiti di questa analisi

- I file sul Ryzen **non sono stati letti**: la sessione gira in un container cloud senza accesso al server. Analisi basata solo sulla sintesi fornita.
- Riferimenti esterni: letta solo la issue omarchy #15452. Openwall e Launchpad erano bloccati dalla rete della sessione.

## 1. Due fenomeni distinti

### A. Oops con il kernel 7.0.0-42 (8–10 novembre)

- Comparsi nello stesso avvio, in processi diversi (rustdesk, poi runc), tutti nel percorso `execve` (avvio di un programma e preparazione della sua memoria).
- L'indirizzo `fffffffffffffed8` vale −0x128: un puntatore NULL a cui è stato sottratto uno spostamento. Di solito indica una struttura vuota o già liberata, quindi fa pensare a un **bug software (race) o a memoria corrotta**, non a un singolo programma difettoso.
- La patch Openwall tocca proprio `mm_update_next_owner/try_to_set_owner`: la somiglianza c'è. Dal titolo però parla di una data race KCSAN, non di un crash. Finché non la leggiamo resta un indizio, non una corrispondenza.

### B. Blocchi totali senza traccia (sera del 10 con -38, notte 10–11 con 6.8)

1. **Blocco con tre kernel diversi** (7.0-42, 7.0-38, 6.8-147). Un bug di una sola versione del kernel diventa poco probabile. Più probabile un elemento comune ai tre avvii: hardware, BIOS, firmware condiviso, gestione energetica.
2. **Nell'ultimo blocco c'era `nomodeset`.** Con questo parametro il driver `amdgpu` di norma non si attiva, quindi lo scenario omarchy (GPU/TTM) **non spiega bene l'ultimo episodio**. Da confermare nel journal.
3. **Blocco notturno con Docker fermo, probabilmente a macchina inattiva.** Sui mini PC AMD Phoenix/Hawk Point sono documentati blocchi durante il riposo profondo della CPU (C-state). Ipotesi, non prova.

### Punto mancante nelle analisi precedenti: tornare a -38 non era un vero ritorno indietro

Firmware GPU (`linux-firmware`) e microcodice CPU (`amd64-microcode`) sono **condivisi da tutti i kernel**. Se l'aggiornamento automatico del 3 novembre ha aggiornato anche quei pacchetti, avviando -38 si ripristinava il kernel ma non il firmware.

Il microcodice 0x0a705208 visto con -38 normale viene dal pacchetto. Lo 0x0a705205 visto in recovery è quello del BIOS, perché `dis_ucode_ldr` blocca il caricamento del pacchetto. **Da verificare se il 208 è arrivato il 3 novembre.**

## 2. Ipotesi e risultati che le distinguono

| Ipotesi | La sostiene | La rafforza | La indebolisce |
|---|---|---|---|
| H1: bug del kernel 7.0-42 (solo per gli Oops) | Oops in exec/mm, patch a monte sulle stesse funzioni | Tutti gli Oops in exec/mm e nessuno con -38 o 6.8 | Oops anche con altri kernel |
| H2: RAM o corruzione di memoria | Oops in processi diversi, blocchi con più kernel | Errori Memtest, Oops in punti sparsi del kernel | 4 passate pulite (non la esclude) |
| H3: blocco a riposo o di piattaforma (C-state, firmware, BIOS) | Blocco notturno, 3 kernel, nessuna traccia | Lungo silenzio del journal prima del blocco, a macchina inattiva | Blocco sotto carico |
| H4: firmware o microcodice aggiornati il 3 novembre | Unico cambiamento comune ai kernel | Pacchetti aggiornati il 3/11 e nessun blocco prima | Nessun aggiornamento firmware, o blocchi già prima del 3/11 |
| H5: GPU (amdgpu) | Issue omarchy sulla stessa CPU | Messaggi amdgpu/REG_WAIT/TTM nel journal | amdgpu non caricato durante il blocco notturno |

## 3. Verifiche in sola lettura (sul Ryzen, un comando alla volta)

Nessun comando modifica il sistema. Le password vanno digitate personalmente.

**V1. Quando si è bloccato di notte e cosa stava facendo?**

    journalctl --list-boots --no-pager | tail -8
    sudo journalctl -b -1 -n 60 --no-pager

- Lungo silenzio prima della fine, a macchina ferma: si rafforza H3.
- Interruzione durante un'attività notturna (apt, backup, timer): correlazione da verificare.

**V2. Durante il blocco notturno amdgpu era caricato?**

    sudo journalctl -b -1 -k --no-pager | grep -iE "amdgpu|nomodeset|command line" | head -20

- Se amdgpu non compare, H5 si indebolisce molto.

**V3. Cosa ha installato l'aggiornamento del 3 novembre?**

    grep " install \| upgrade " /var/log/dpkg.log | grep "2026-11-0[1-9]" | grep -iE "firmware|microcode|linux-|bios|fwupd"

- Se compaiono `linux-firmware` o `amd64-microcode`, H4 diventa concreta.

**V4. Qual è l'ultimo periodo stabile documentato?**

    journalctl --list-boots --no-pager | head -40

- Cercare il periodo più lungo senza spegnimenti forzati prima del 7 novembre.
- Versione del kernel di un avvio: `journalctl -b <ID> -k | grep -m1 "Linux version"`.

**V5. Quali aggiornamenti sono in attesa?**

    apt list --upgradable 2>/dev/null

**V6. Gli Oops del kernel -42 sono tutti dello stesso tipo?** (`<ID>` = avvio -42 dell'8 novembre)

    sudo journalctl -b <ID> -k --no-pager | grep -E "Oops|BUG:|RIP:|Tainted" | head -30

- Sempre lo stesso percorso exec/mm: si rafforza H1. Funzioni sparse: si rafforza H2.
- Controllare le lettere "Tainted" del **primo** Oops.

**V7. Cosa c'è in /var/crash?**

    ls -l --time-style=long-iso /var/crash/

**Verifica a costo zero dall'i9 (consigliata):** durante il prossimo uso, ping con orario:

    ping -D 192.168.178.150 | tee ping-ryzen.log

- Il ping continua durante il blocco: kernel vivo, bloccata la grafica o la parte utente.
- Il ping si ferma: bloccata tutta la macchina (CPU, piattaforma, firmware).

## 4. Proposte da valutare (non applicare senza decisione esplicita)

- **Backup:** prima di qualunque prova, confermare dove si trova un backup ripristinabile di progetti, database e configurazioni. Gli archivi diagnostici non lo sostituiscono.
- **netconsole:** invia all'i9 i messaggi del kernel via UDP direttamente dal kernel, anche quando i programmi normali sono fermi. Richiede un parametro di avvio, quindi è una modifica da decidere.
- **Memtest:** completare le 4 passate. Se pulite, H2 si indebolisce ma non è esclusa.
- **BIOS:** il BIOS è del 03/03/2025. Controllare soltanto se ACEMAGIC ne ha pubblicato uno più recente, senza installarlo.

**Passo con più informazioni per lo sforzo richiesto:** V1 + V2 + V3.

## Riferimento esterno letto: omarchy #15452

- Beelink SER8, stesso Ryzen 7 8745HS / Radeon 780M, kernel Arch 7.2.x.
- Blocco con `REG_WAIT timeout ... dcn31_program_compbuf_size`, avviso `ttm_resource_add_bulk_move`, poi soft lockup su più CPU.
- Causa del blocco non identificata. Problema separato: perdita di memoria GTT dovuta a Hyprland (corretto in Hyprland 0.57.0).
- Stato: aperta. Nei nostri log tracce amdgpu/TTM non ancora riscontrate.
