# Progetti finali


I progetti si svolgono **a coppie**. Il codice va gestito con `git` e le dipendenze con `uv`. Al termine della consegna, il docente si occupa del deploy sul server del laboratorio.

### Requisiti comuni a tutti i progetti

**Struttura**
- Application factory (`create_app`) in `factory.py`
- Configurazione in `config.py` con `DevConfig` e `ProdConfig`
- Blueprint separati per web app e API
- Template con ereditarietà da `base.html`

**Codice**
- Dipendenze gestite con `uv` — file `pyproject.toml` aggiornato
- Nessuna secret key nel codice sorgente — va in `.env`
- `.env` presente ma non committato su git (nel `.gitignore`)
- `debug=False` in `ProdConfig`

**Consegna**
- Repository git con storia dei commit significativa — almeno un commit per funzionalità
- `README.md` con descrizione del progetto e istruzioni per avviarlo in locale
- Nessun file `__pycache__`, `.env`, o `uploads/` nel repository

---

### Progetto A — 🖼️ Galleria condivisa

**Descrizione**
Un'app dove gli utenti registrati caricano immagini che vengono mostrate in una galleria pubblica. Chiunque può vedere le immagini, solo gli utenti loggati possono caricarle ed eliminare le proprie.

**Funzionalità richieste**

*Web app*
- Registrazione e login con password hashata
- Upload di immagini (solo `jpg`, `png`, `gif`, max 2 MB)
- Galleria pubblica con nome dell'autore e data di caricamento
- Ogni utente può eliminare solo le proprie immagini
- Flash messages per tutte le operazioni

*API REST*
- `GET /api/immagini` — lista di tutte le immagini con paginazione
- `GET /api/immagini/<indice>` — dettaglio di una singola immagine
- `DELETE /api/immagini/<indice>` — elimina un'immagine (richiede token)

*Autenticazione API*
- `POST /api/login` — restituisce un token
- Le operazioni di modifica richiedono il token nell'header `Authorization: Bearer <token>`

**Struttura attesa**
```
galleria/
├── factory.py
├── config.py
├── app.py
├── .gitignore
├── README.md
├── pyproject.toml
├── auth/
│   ├── __init__.py
│   ├── routes.py
│   └── utils.py
├── galleria/
│   ├── __init__.py
│   ├── routes.py
│   └── utils.py
├── api/
│   ├── __init__.py
│   └── routes.py
└── templates/
    ├── base.html
    ├── auth/
    └── galleria/
```

---

### Progetto B — 📝 Mini blog

**Descrizione**
Un blog multiutente dove gli utenti registrati pubblicano post con titolo, testo e tag. La home mostra tutti i post in ordine cronologico. Chiunque può leggere, solo gli autori possono modificare o eliminare i propri post.

**Funzionalità richieste**

*Web app*
- Registrazione e login con password hashata
- Creazione di post con titolo, testo e tag (separati da virgola)
- Home con tutti i post in ordine dal più recente, con autore e data
- Pagina di dettaglio del singolo post
- Ogni utente può modificare ed eliminare solo i propri post
- Filtro per tag: cliccando su un tag si vedono solo i post con quel tag
- Flash messages per tutte le operazioni

*API REST*
- `GET /api/post` — lista dei post con paginazione
- `GET /api/post/<indice>` — dettaglio di un post
- `GET /api/post?tag=flask` — filtra i post per tag
- `POST /api/post` — crea un nuovo post (richiede token)
- `DELETE /api/post/<indice>` — elimina un post (richiede token)

*Autenticazione API*
- `POST /api/login` — restituisce un token
- Le operazioni di modifica richiedono il token nell'header `Authorization: Bearer <token>`

**Struttura attesa**
```
miniblog/
├── factory.py
├── config.py
├── app.py
├── .gitignore
├── README.md
├── pyproject.toml
├── auth/
│   ├── __init__.py
│   ├── routes.py
│   └── utils.py
├── blog/
│   ├── __init__.py
│   ├── routes.py
│   └── utils.py
├── api/
│   ├── __init__.py
│   └── routes.py
└── templates/
    ├── base.html
    ├── auth/
    └── blog/
```

---

### Progetto C — 🗳️ Sondaggi

**Descrizione**
Un'app dove gli utenti registrati creano sondaggi con una domanda e più opzioni di risposta. Chiunque può votare — ma solo una volta per sondaggio, controllato tramite sessione. I risultati sono visibili in tempo reale dopo il voto.

**Funzionalità richieste**

*Web app*
- Registrazione e login con password hashata
- Creazione di un sondaggio con domanda e almeno due opzioni
- Pagina di voto: mostra la domanda e le opzioni come radio button
- Dopo il voto, reindirizza alla pagina dei risultati con le percentuali
- Un utente non loggato può votare, ma non creare sondaggi
- Ogni utente può eliminare solo i propri sondaggi
- Flash messages per tutte le operazioni

*Controllo voto unico*
- Il controllo che un utente abbia già votato va fatto tramite sessione
- Se un utente tenta di votare due volte, viene reindirizzato ai risultati con un messaggio

*API REST*
- `GET /api/sondaggi` — lista di tutti i sondaggi
- `GET /api/sondaggi/<indice>` — dettaglio con domanda, opzioni e risultati
- `POST /api/sondaggi` — crea un nuovo sondaggio (richiede token)
- `DELETE /api/sondaggi/<indice>` — elimina un sondaggio (richiede token)

*Autenticazione API*
- `POST /api/login` — restituisce un token
- Le operazioni di modifica richiedono il token nell'header `Authorization: Bearer <token>`

**Struttura attesa**
```
sondaggi/
├── factory.py
├── config.py
├── app.py
├── .gitignore
├── README.md
├── pyproject.toml
├── auth/
│   ├── __init__.py
│   ├── routes.py
│   └── utils.py
├── sondaggi/
│   ├── __init__.py
│   ├── routes.py
│   └── utils.py
├── api/
│   ├── __init__.py
│   └── routes.py
└── templates/
    ├── base.html
    ├── auth/
    └── sondaggi/
```

---

### Progetto D — 📅 Agenda condivisa

**Descrizione**
Un'agenda dove gli utenti registrati aggiungono eventi con titolo, data, ora e descrizione. Gli eventi sono visibili a tutti in ordine cronologico. Ogni utente gestisce solo i propri eventi.

**Funzionalità richieste**

*Web app*
- Registrazione e login con password hashata
- Aggiunta di eventi con titolo, data (`date`), ora (`time`) e descrizione
- Home con tutti gli eventi futuri in ordine cronologico, con autore
- Evidenziazione visiva degli eventi del giorno corrente
- Ogni utente può modificare ed eliminare solo i propri eventi
- Flash messages per tutte le operazioni

*API REST*
- `GET /api/eventi` — lista di tutti gli eventi futuri con paginazione
- `GET /api/eventi/<indice>` — dettaglio di un evento
- `GET /api/eventi?autore=mario` — filtra gli eventi per autore
- `POST /api/eventi` — crea un nuovo evento (richiede token)
- `DELETE /api/eventi/<indice>` — elimina un evento (richiede token)

*Autenticazione API*
- `POST /api/login` — restituisce un token
- Le operazioni di modifica richiedono il token nell'header `Authorization: Bearer <token>`

**Struttura attesa**
```
agenda/
├── factory.py
├── config.py
├── app.py
├── .gitignore
├── README.md
├── pyproject.toml
├── auth/
│   ├── __init__.py
│   ├── routes.py
│   └── utils.py
├── agenda/
│   ├── __init__.py
│   ├── routes.py
│   └── utils.py
├── api/
│   ├── __init__.py
│   └── routes.py
└── templates/
    ├── base.html
    ├── auth/
    └── agenda/
```


Buon lavoro!
