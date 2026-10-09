# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Cos'è

Prototipi HTML usa-e-getta per la "Banca Dati" di Mediaddress (staging: https://staging.mediaddress.com/it/rolodex). Per ora un solo prototipo: la ricerca per testo libero, che mette a confronto l'interfaccia attuale (2 input) con quella proposta (1 input con suggerimenti e chip per campo). Non c'è git, build tool né test.

## File e build

- `ricerca-proto.template.html`: il codice da modificare (HTML, CSS e JS inline). Contiene il segnaposto `/*DATA*/`.
- `ricerca-proto-data.json`: campione di dati di staging, `{o: testate[], p: giornalisti[]}`.
- `ricerca-proto-v2.template.html` → `ricerca-proto-v2.html`: copia del prototipo con la sola modalità "1 input" (nessun interruttore di modalità, nessuna tendina, etichetta "Filtra per termine"). Ha lo stesso `/*DATA*/` e si rigenera allo stesso modo, sostituendo il nome del template e dell'output. Le due pagine sono indipendenti: una modifica a una non si propaga all'altra.
- `ricerca-proto.html`: **generato**, non modificarlo a mano. Dopo ogni modifica al template o ai dati va rigenerato con:

```sh
python3 -c "
t=open('ricerca-proto.template.html').read(); d=open('ricerca-proto-data.json').read().replace('</','<\\\\/')
open('ricerca-proto.html','w').write(t.replace('/*DATA*/',d))"
```

I dati vengono incorporati nella pagina, così il file si apre anche con `file://`. Claude in Chrome però non può navigare su `file://`: per vederlo nel browser serve un server locale, per esempio `python3 -m http.server 8811 --bind 127.0.0.1`, e poi `http://127.0.0.1:8811/ricerca-proto.html`.

## Screenshot e gallery delle iterazioni

Dopo ogni modifica alla UI del prototipo:

1. Fai gli screenshot degli stati rilevanti in Chrome, con `save_to_disk: true`. Puoi pilotare lo stato via `javascript_tool`, per esempio impostando il valore di `#q` e lanciando un evento `input`.
2. Copiali in `screenshots/` con il nome `itNN-K-descrizione.jpg`, dove `NN` è il numero dell'iterazione.
3. In `iterazioni.html` aggiungi una nuova `<section class="iter">` **in cima**, sopra le precedenti. Mettici numero, data, cosa è cambiato e una `figure` con didascalia per ogni screenshot. Sposta il badge "ultima" sulla nuova sezione.

## Struttura del JS del prototipo

- **Dati**: `O` (Map id → testata) e `P` (array di giornalisti) vengono collegati al caricamento: `p.os` sono le testate del giornalista, `o.ps` i giornalisti della testata. I campi `_n`, `_e`, `_x` sono nome, email e note normalizzati (minuscolo, senza accenti).
- **I 5 campi di ricerca** (`FIELDS`): `gn`/`ge`/`gx` sono nome, email e note del giornalista; `tn`/`tx` sono nome e note della testata. `matchPerson` e `matchOutlet` cercano anche nei campi dell'entità collegata e salvano in `via` il nome su cui c'è stata la corrispondenza.
- **Rilevanza**: `W.g` e `W.t` sono i pesi dei campi per il tab Giornalisti e per il tab Testate. Ogni peso viene moltiplicato per `quality()`. Se il testo, tolti gli articoli (`STOP`), coincide con la query vale 2; altrimenti vale 1.3 per una parola intera, 1.2 per un inizio parola, 1 per una sottostringa, più fino a 0.5 in base a quanta parte del testo copre la query. Il punteggio di un risultato è il campo migliore più l'1% della somma di tutti i campi trovati. A parità di punteggio (`byRank`) si ordina per numero di giornalisti della testata, poi in ordine alfabetico.
- **Flusso**: `state` (mode, tab, query, filtro entità `ent`, campi attivi `on`) → `compute()` → `render()`.
- **Modalità**: `due` ("2 input") riproduce la logica attuale: input A sui campi del giornalista, input B su quelli della testata, combinati in AND. `secco` ("1 input") ha una sola casella senza tendina e filtra mentre si scrive. Invio fissa il termine in `state.pins`, mostrati come chip; tutti i termini attivi (`activeTerms()`) sono in AND e il punteggio è la somma dei punteggi per termine. Questa modalità mostra anche la colonna `#criteria` con la spiegazione del ranking. `uno` ("1 input + tendina") aggiunge i suggerimenti. `secco` e `uno` condividono la casella `#q`.

## Rigenerare i dati da staging

Le API richiedono il token Bearer della sessione già loggata nel browser. Lo si legge intercettando le richieste della pagina rolodex, senza mai stamparlo.

- `POST /api/persons/search` con body `{per_page, n, db:"1", sort:"score_za"}`: ricerca giornalisti.
- `GET /api/outlets?n=…&db=1&per_page=…`: ricerca testate. `GET /api/outlets/{id}` restituisce il dettaglio, compreso `media_type_id`.
- `GET /api/outlets/{id}/persons`: i giornalisti di una testata.
- `GET /api/staff-positions` e `GET /api/media-types`: le etichette, nel campo `value`.

Le uscite di `javascript_tool` vengono troncate a circa 1000 caratteri, e la pagina di staging blocca fetch e form verso localhost. Per portare il JSON sul disco funziona così: dalla pagina si chiama `navigator.clipboard.writeText(...)`, poi `pbpaste > ricerca-proto-data.json`. Sovrascrive gli appunti dell'utente: diglielo.

## Vincoli

- In `ricerca-proto-data.json` i nomi di battesimo dei giornalisti, la parte prima della @ delle loro email (e delle email personali delle testate) e i numeri di telefono nelle note sono inventati; cognomi e testate sono reali. Se rigeneri i dati da staging, anonimizzali di nuovo prima di fare commit. Gli screenshot in `screenshots/` mostrano ancora i dati reali. Non pubblicare il prototipo come Artifact.
- Lo stile segue il brand definito nel CLAUDE.md globale. I token sono copiati nel `:root` del template su tre livelli: scale, palette, semantiche.
