# Progetto finale e verifica

In questo capitolo metterai insieme tutto quello che hai imparato:
- strutturare un progetto Flask completo
- usare template Jinja2 con ereditarietà
- gestire form e flash messages
- salvare e leggere dati su file JSON
- esporre un'API REST

---

## Progetto finale

Il progetto finale consiste nel costruire una piccola applicazione Flask completa, con interfaccia web e API REST. 
Puoi scegliere uno dei quattro temi proposti di seguito — la struttura richiesta è la stessa per tutti.

Realizza un'applicazione Flask che soddisfi i seguenti punti. Ogni punto è indipendente: se non riesci a completarne uno, passa al successivo.

---

### Requisiti comuni

Ogni progetto deve includere:

**Web app**
- Un template `base.html` con navbar e footer
- Una pagina principale che mostra tutti gli elementi
- Un form per aggiungere un nuovo elemento (con validazione e flash messages)
- La possibilità di modificare ed eliminare un elemento

**Storage**
- I dati devono essere salvati su file JSON
- Usare le funzioni `leggi_dati()` e `salva_dati()` come visto nel Capitolo 4

**API REST**
- `GET /api/<risorsa>` — restituisce tutti gli elementi
- `GET /api/<risorsa>/<indice>` — restituisce un singolo elemento
- `POST /api/<risorsa>` — aggiunge un nuovo elemento
- `DELETE /api/<risorsa>/<indice>` — elimina un elemento

**Organizzazione**
- Le route della web app e le route API devono essere in Blueprint separati

---

### Proposta A — 🎬 Watchlist film

Gestisci una lista personale di film.

Ogni film ha:
- `titolo` (stringa, obbligatorio)
- `regista` (stringa, obbligatorio)
- `anno` (intero, obbligatorio)
- `visto` (booleano, default `false`)

Funzionalità aggiuntive suggerite:
- Filtra la lista per mostrare solo i film visti o solo quelli da vedere
- Endpoint API `GET /api/film/da-vedere` che restituisce i film con `visto: false`

---

### Proposta B — 📋 Bacheca annunci

Pubblica e gestisci annunci.

Ogni annuncio ha:
- `titolo` (stringa, obbligatorio)
- `testo` (stringa, obbligatorio)
- `autore` (stringa, obbligatorio)
- `data` (stringa ISO, generata automaticamente con `datetime.date.today().isoformat()`)

Funzionalità aggiuntive suggerite:
- Gli annunci sono mostrati in ordine dal più recente al più vecchio
- Endpoint API `GET /api/annunci/cerca?q=...` che filtra per parola chiave nel titolo o nel testo

---

### Proposta C — 🏆 Classifica torneo

Gestisci la classifica di un torneo.

Ogni giocatore ha:
- `nome` (stringa, obbligatorio)
- `punteggio` (intero, default `0`)
- `partite` (intero, default `0`)

Funzionalità aggiuntive suggerite:
- La classifica è sempre mostrata ordinata per punteggio decrescente
- Endpoint API `GET /api/classifica/top?n=3` che restituisce i primi `n` giocatori

---

### Proposta D — 🍕 Registro recensioni

Tieni traccia di locali visitati.

Ogni recensione ha:
- `nome` (stringa, obbligatorio)
- `tipo` (stringa, es. `"ristorante"`, `"bar"`, `"pizzeria"`)
- `voto` (intero da 1 a 10, obbligatorio)
- `commento` (stringa, opzionale)

Funzionalità aggiuntive suggerite:
- Mostra la media dei voti in cima alla pagina
- Endpoint API `GET /api/locali/top?voto_min=8` che filtra per voto minimo

---

### Struttura consigliata del progetto

```
progetto/
├── app.py
├── views.py        ← Blueprint web app
├── api.py          ← Blueprint API
├── dati.json       ← creato automaticamente
└── templates/
    ├── base.html
    ├── index.html
    └── modifica.html
```

---

### Tempo per la prova

100 minuti


---

### Valutazione prova pratica


**[20 punti] Web app**

1. Crea il template `base.html` con navbar e almeno due link di navigazione
2. Crea la pagina principale che mostra tutti gli elementi letti dal file JSON
3. Aggiungi un form per inserire un nuovo elemento, con validazione e flash messages
4. Implementa la funzione di eliminazione di un elemento

**[20 punti] Storage JSON**

5. I dati sono salvati e letti correttamente da file JSON
6. L'applicazione gestisce correttamente il caso in cui il file non esista ancora
7. Implementa le funzioni `leggi_dati()` e `salva_dati()`

**[20 punti] API REST**

8. Endpoint `GET /api/<risorsa>` che restituisce tutti gli elementi in JSON
9. Endpoint `GET /api/<risorsa>/<indice>` che restituisce un singolo elemento, con `404` se non trovato
10. Endpoint `POST /api/<risorsa>` che aggiunge un elemento ricevuto come JSON, con validazione
11. Endpoint `DELETE /api/<risorsa>/<indice>` che elimina un elemento

**[10 punti] Organizzazione**

12. Le route web e le route API sono in Blueprint separati

**[10 punti] Funzionalità aggiuntiva**

13. Implementa la funzionalità aggiuntiva suggerita per il tema scelto

<br>
<br>
<br>
