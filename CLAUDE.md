# Segnapunti Yahtzee

Web app in **un solo file** (`index.html`) per giocare a Yahtzee in famiglia: segnapunti per dadi veri, dadi virtuali, modalità storia con avversari virtuali e minigiochi. Interfaccia e testi in **italiano**. Nessuna dipendenza esterna, nessun build: HTML + CSS + JS vanilla inline. Pubblicata con GitHub Pages.

## File

- `index.html` — tutta l'app (stile, markup, script). È l'unico file che va pubblicato.
- `README.md` — descrizione per GitHub.
- `.claude/launch.json` + `.claude/serve.ps1` — mini server PowerShell (`http://localhost:8765/`) per l'anteprima nel browser di Claude Code. Su questo PC Python e Node non ci sono.

## Come provare le modifiche

1. Avvia l'anteprima con la configurazione `yahtzee` di `.claude/launch.json`.
2. Prova sia in desktop sia in mobile (375×812): la cartella e la barra di navigazione cambiano molto sotto i 640px.
3. Controlla la console: deve essere pulita.
4. Alla fine **cancella i dati di prova** (`localStorage.clear(); indexedDB.deleteDatabase('yahtzee')`), così l'utente non trova partite finte.

## Struttura dello script (in ordine)

1. **Dati e regole**: `AIS` (avversari), `MINIS` (minigiochi), `STORY` (percorso), `POWERS` (aiuti), `PRESET` (giocatori precaricati), `defaults()`, `load()`/`save()`, `UP`/`LO`/`CATS` (13 caselle), `calc()`, `diceScore()`, `guidedOptions()`.
2. **Utility UI**: `die()` (dado SVG), `av()` (avatar), `toast()`, `ask()` (conferma su `<dialog id="dlg">`, unico per tutta l'app).
3. **Navigazione**: `render()` ridisegna la vista del tab attivo con `innerHTML`; `afterGameRender()` gestisce cambio turno, scorrimento automatico e avvio del turno dell'avversario.
4. **Preferenze**: tema, caratteri grandi, schermo acceso (Wake Lock), suoni (WebAudio sintetizzato), vibrazione, lancio scuotendo il telefono (`devicemotion`), finestra Impostazioni.
5. **Partita**: `viewGame()`, `viewNewGame()`, `startGame()`, annulla (`pushUndo`/`undo`), dadi virtuali (`d6()` crittografico ed equo, `rollDice()`, `jokerState()`, `scoreVirtual()`).
6. **Avversari virtuali**: `aiRoll()` (lancio con fortuna), `naivePlan()`/`naiveBox()` (gioco "a istinto"), `aiTurnGen()` (generatore del turno, usato sia dal gioco sia dalle simulazioni), `runAI()` (turno animato), `simAI()`.
7. **Consiglio**: `suggest()` calcola il valore atteso esatto di ogni combinazione di dadi da tenere, guardando avanti fino a 2 lanci. Le caselle si valutano come punti − `PAR` (media di una partita ben giocata), con una spinta verso il bonus di 35.
8. **Modale punteggio** (modalità segnapunti) con tre modi di inserimento: scelta guidata, dadi, numero.
9. **Storia**: `viewStory()`, `openNode()`, `startStory()`, `showStoryFinal()`, `usePower()`, minigioco (`viewMini()`, `miniRoll()`, `finishMini()`).
10. **Giocatori, Statistiche, Storico, Record, Backup, Regole**, poi gestione eventi.

Eventi: un unico listener `click` su `document` che smista su `data-act`. Per aggiungere un pulsante usa `data-act="nome"` e aggiungi un `case`.

## Modello dati

Tutto lo stato sta in `S`, salvato in `localStorage['yahtzee-segnapunti-v1']`. I campi:

- `players` — giocatori: `{id, name, nick, emoji, color}`. Quelli precaricati hanno id `p-<nome>`.
- `current` — partita in corso o `null`: `{playerIds, scores:{pid:{casella:valore}}, yb:{pid:n}, undo, virtual, roll, story?}`.
  - `roll` è `{dice, held, n, max, lucky?, extraUsed?, fixUsed?}`.
  - `story` è `{aiId, pid}`.
- `games` — partite finite, dalla più recente. Contengono i risultati copiati (nome ed emoji) perché i giocatori possono essere eliminati.
- `story[pid]` — progressi nella storia: `{step, beaten, best, coins, mini}`.
- `mini` — minigioco in corso.
- Altri campi: `storyHero`, `prefs`, `diceStats` (solo dadi virtuali umani), `lastBackup`.

Il backup è il JSON di `S`. Al caricamento si mantengono le `prefs` del dispositivo. L'ultimo file usato viene ricordato in IndexedDB (`yahtzee`/`kv`/`handle`), così la finestra di salvataggio riparte da lì.

## Regole implementate (Yahtzee classico)

- **Sezione superiore**: Uno…Sei, con bonus di 35 se la somma arriva a 63.
- **Sezione inferiore**: Tris e Poker valgono la somma dei dadi, Full 25, Scala piccola 30, Scala grande 40, Yahtzee 50, Chance la somma.
- **Bonus Yahtzee**: +100 per ogni Yahtzee in più, se nella casella Yahtzee ci sono già 50 punti.
- **Regola del jolly ufficiale**: prima la casella superiore del numero uscito; se è già usata, una casella inferiore libera a punteggio pieno; altrimenti si sbarra una superiore.
  - Con i dadi virtuali la regola è **imposta**.
  - In modalità segnapunti è **solo un avviso**.

## Modalità storia: taratura

I parametri degli avversari sono: `skill` (probabilità di giocare la mossa migliore), `stop` (pigrizia), `dumb` (tiene solo i tris), `bias` (pesi delle facce), `extraRoll`, `match` (un dado rilanciato copia quelli uguali tenuti), `straight` (arriva il numero mancante alla scala).

I valori sono stati tarati con `simAI()`. Medie indicative, su campioni da 50 a 200 partite:

| Avversario | Media | Partite con Yahtzee |
|---|---|---|
| Sergio | ~120 | |
| Pippo | ~150 | |
| Gianni | ~175 | |
| Mario | ~205 | |
| Lucia | ~220 | |
| Gianna | ~235 | |
| Conte | ~250 | |
| Mastro Chiappone | ~270 | circa 50% (richiesta esplicita dell'utente) |

Riferimento: con `skill:1` e nessuna fortuna si fanno circa 243 punti e uno Yahtzee nel 32% delle partite.

**Attenzione**: `match` è potentissimo. Con 0,17 la media sale a circa 340, perché arrivano tanti bonus Yahtzee. Anche i dadi sbilanciati verso una faccia fanno salire molto gli Yahtzee. Se cambi un parametro, **risimula**. Una partita di un avversario con `skill` alto richiede circa 1,5 s. Fai girare le simulazioni a blocchi con `setTimeout` per non bloccare la pagina.

I risultati degli avversari (id `ai-…`) sono esclusi da record e statistiche. Nella storia "Annulla" è nascosto.

## Insidie note

- **Emoji**: Windows 10 non ha 🪙, per questo la moneta è 💰. Prima di usare emoji recenti (Unicode 13 o successive) verifica che si vedano.
- **Barra di navigazione su telefono**: non mettere `backdrop-filter` sull'`header` sotto i 640px, altrimenti la barra `position:fixed` in basso si aggancia all'header.
- **Vibrazione**: `navigator.vibrate` funziona solo dopo un tocco dell'utente; il controllo `userActivation` evita errori in console.
- **Schermo acceso, scelta cartella del backup, sensore di movimento**: richiedono un contesto sicuro. Su GitHub Pages (https) funzionano. Aprendo il file dal disco funzionano solo in parte.
- **Classi dei pulsanti**: le etichette dei pulsanti che si nascondono sul telefono usano `.lbl`. Nella barra di partita la regola è `.gamebar .btn .lbl`, per non nascondere anche il testo "Turno X di 13".

## Preferenze dell'utente

- Testi e risposte in italiano, tono semplice: l'app la usa tutta la famiglia.
- Giocatori precaricati: Ale, Laura, Roby, Jo, Sere, Fra, Steu, Monica.
- Il backup si carica all'inizio e si scarica a fine partita: tieni sempre visibili i due pulsanti.
- Il lancio scuotendo il telefono resta facoltativo (spento di default) e il pulsante "Lancia" resta sempre.
- I dati restano separati per dispositivo. Il salvataggio online è stato proposto e rifiutato.
