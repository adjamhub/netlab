# API REST avanzata

In questo capitolo vedremo come:

- gestire gli errori HTTP in modo centralizzato
- implementare la paginazione dei risultati
- proteggere le API con token di autenticazione
- documentare gli endpoint con una pagina di riferimento

---

## Gestione centralizzata degli errori


Finora quando qualcosa andava storto restituivamo un errore direttamente nella route:

```python
if not dati:
    return jsonify({'errore': 'Dati mancanti'}), 400
```

In un'API con molti endpoint questo approccio ha due problemi: il formato degli errori rischia di essere inconsistente tra una route e l'altra, e il codice si riempie di controlli ripetuti.

La soluzione è gestire gli errori in modo **centralizzato**.

---

### Handler di errori globali

Flask permette di registrare handler per i codici di errore HTTP. Questi handler intercettano automaticamente gli errori sollevati ovunque nell'applicazione:

```python
from flask import jsonify

@app.errorhandler(400)
def bad_request(e):
    return jsonify({'errore': 'Richiesta non valida', 'dettaglio': str(e)}), 400

@app.errorhandler(404)
def not_found(e):
    return jsonify({'errore': 'Risorsa non trovata'}), 404

@app.errorhandler(405)
def method_not_allowed(e):
    return jsonify({'errore': 'Metodo non consentito'}), 405

@app.errorhandler(500)
def internal_error(e):
    return jsonify({'errore': 'Errore interno del server'}), 500
```

Ora tutte le risposte di errore hanno lo stesso formato JSON, indipendentemente da dove viene generato l'errore.

---

### Sollevare errori con `abort`

Nelle route puoi sollevare un errore HTTP con `abort`, che verrà intercettato dall'handler corrispondente:

```python
from flask import abort

@app.route('/api/libri/<int:indice>')
def get_libro(indice):
    libri = leggi_dati()
    if indice < 0 or indice >= len(libri):
        abort(404)
    return jsonify(libri[indice]), 200
```

`abort(404)` interrompe immediatamente l'esecuzione della route e passa il controllo all'handler del 404. 
Il codice diventa più leggibile — niente più `return jsonify({'errore': ...})` ripetuti ovunque.

---

### Errori personalizzati

Per errori più specifici puoi creare eccezioni personalizzate:

```python
from werkzeug.exceptions import HTTPException

class DatiMancanti(HTTPException):
    code = 400
    description = 'I dati JSON sono mancanti o malformati.'

class RisorsaNonTrovata(HTTPException):
    code = 404
    description = 'La risorsa richiesta non esiste.'

@app.errorhandler(DatiMancanti)
def dati_mancanti(e):
    return jsonify({'errore': e.description}), e.code
```

---

### Esercizi

**Esercizio 741**

Aggiungi al progetto gli handler per i codici `400`, `404` e `500`. Sostituisci tutti i `return jsonify({'errore': ...})` nelle route API con `abort(codice)`. Verifica che le risposte di errore abbiano tutte lo stesso formato.

--

**Esercizio 742**

Aggiungi un handler per il codice `429 Too Many Requests`. Per ora non serve implementare un vero rate limiting — basta che l'handler restituisca un JSON con un messaggio appropriato.

---

## Paginazione

Un endpoint che restituisce tutti i dati in una volta sola funziona finché i dati sono pochi. Con centinaia o migliaia di elementi diventa lento e pesante. 
La **paginazione** divide i risultati in pagine di dimensione fissa.

---

### Paginazione con query string

Il modo più semplice è accettare i parametri `pagina` e `per_pagina` nella query string:

```
GET /api/libri?pagina=2&per_pagina=5
```

```python
@app.route('/api/libri')
def get_libri():
    libri = leggi_libri()

    # Legge i parametri con valori di default
    try:
        pagina = int(request.args.get('pagina', 1))
        per_pagina = int(request.args.get('per_pagina', 5))
    except ValueError:
        abort(400)

    if pagina < 1 or per_pagina < 1:
        abort(400)

    # Calcola gli indici di inizio e fine
    inizio = (pagina - 1) * per_pagina
    fine = inizio + per_pagina

    risultati = libri[inizio:fine]
    totale = len(libri)
    pagine_totali = (totale + per_pagina - 1) // per_pagina

    return jsonify({
        'dati': risultati,
        'paginazione': {
            'pagina': pagina,
            'per_pagina': per_pagina,
            'totale': totale,
            'pagine_totali': pagine_totali,
            'ha_precedente': pagina > 1,
            'ha_successiva': pagina < pagine_totali
        }
    }), 200
```

La risposta include sia i dati che le informazioni di paginazione — il client sa quante pagine esistono e se può navigare avanti o indietro.

---

### Usare la paginazione con requests

```python
import requests

# Prima pagina
r = requests.get('http://127.0.0.1:5000/api/libri?pagina=1&per_pagina=3')
dati = r.json()

print(f"Pagina {dati['paginazione']['pagina']} di {dati['paginazione']['pagine_totali']}")
for libro in dati['dati']:
    print(f"  - {libro['titolo']}")

# Pagina successiva se esiste
if dati['paginazione']['ha_successiva']:
    r2 = requests.get('http://127.0.0.1:5000/api/libri?pagina=2&per_pagina=3')
    print(r2.json())
```

---

### Esercizi

**Esercizio 751**

Aggiungi la paginazione all'endpoint `GET /api/libri`. Popola `libri.json` con almeno 15 elementi e verifica il funzionamento 
con valori diversi di `pagina` e `per_pagina`.

--

**Esercizio 752**

Scrivi uno script `pagina_tutti.py` che usi `requests` per scorrere automaticamente tutte le pagine e stampare tutti i titoli, una pagina alla volta.

---

## Autenticazione con token

Flask-Login funziona bene per le web app, dove il browser gestisce i cookie di sessione automaticamente. Per le API però i cookie non sono pratici — i client come script Python o app mobile preferiscono un sistema più semplice: il **token**.

Il flusso è:

1. Il client fa login inviando username e password
2. Il server verifica le credenziali e restituisce un **token** — una stringa univoca
3. Il client include il token in ogni richiesta successiva nell'header `Authorization`
4. Il server verifica il token e risponde

---

### Generare e salvare i token

Per semplicità generiamo token casuali con `secrets` e li salviamo in un file JSON:

```python
import secrets
import json

TOKEN_FILE = 'token.json'

def leggi_token():
    try:
        with open(TOKEN_FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return {}

def salva_token(token_db):
    with open(TOKEN_FILE, 'w', encoding='utf-8') as f:
        json.dump(token_db, f, indent=2)

def genera_token():
    return secrets.token_hex(32)
```

Il file `token.json` mappa ogni token al relativo username:

```json
{
    "a3f8c2...": "admin",
    "9d1e4b...": "mario"
}
```

---

### Endpoint di login

```python
from werkzeug.security import check_password_hash

@app.route('/api/login', methods=['POST'])
def api_login():
    dati = request.get_json()
    if not dati:
        abort(400)

    username = dati.get('username', '').strip()
    password = dati.get('password', '').strip()

    utente = trova_utente_per_username(username)
    if not utente or not check_password_hash(utente.password, password):
        return jsonify({'errore': 'Credenziali non valide'}), 401

    # Genera e salva il token
    token = genera_token()
    token_db = leggi_token()
    token_db[token] = username
    salva_token(token_db)

    return jsonify({'token': token, 'utente': username}), 200
```

---

### Verificare il token

Creiamo una funzione che estrae e verifica il token dall'header della richiesta:

```python
def verifica_token():
    auth_header = request.headers.get('Authorization', '')

    # Il formato atteso è: "Bearer <token>"
    if not auth_header.startswith('Bearer '):
        abort(401)

    token = auth_header[7:]  # Rimuove "Bearer "
    token_db = leggi_token()

    if token not in token_db:
        abort(401)

    return token_db[token]  # Restituisce lo username
```

Nelle route protette basta chiamare `verifica_token()` all'inizio:

```python
@app.route('/api/libri', methods=['POST'])
def add_libro():
    username = verifica_token()  # Solleva 401 se non valido

    dati = request.get_json()
    if not dati:
        abort(400)

    # ... resto della logica
```

---

### Usare il token con requests

```python
import requests

# Login
r = requests.post('http://127.0.0.1:5000/api/login',
    json={'username': 'admin', 'password': 'admin123'})
token = r.json()['token']

# Richiesta autenticata
headers = {'Authorization': f'Bearer {token}'}

r = requests.post('http://127.0.0.1:5000/api/libri',
    json={'titolo': 'Il nome della rosa', 'autore': 'Umberto Eco'},
    headers=headers)
print(r.status_code, r.json())
```

---

### Logout — invalidare il token

```python
@app.route('/api/logout', methods=['POST'])
def api_logout():
    auth_header = request.headers.get('Authorization', '')
    if auth_header.startswith('Bearer '):
        token = auth_header[7:]
        token_db = leggi_token()
        token_db.pop(token, None)
        salva_token(token_db)
    return jsonify({'messaggio': 'Logout effettuato'}), 200
```

---

### Esercizi

**Esercizio 761**

Implementa il sistema di autenticazione con token. Proteggi gli endpoint `POST`, `PUT` e `DELETE` della tua API — l'endpoint `GET` rimane pubblico. Verifica il funzionamento con uno script `requests`.

--

**Esercizio 762**

Aggiungi un endpoint `GET /api/me` che restituisce le informazioni dell'utente corrente (username) basandosi sul token. Deve restituire `401` se il token non è presente o non è valido.

--

**Esercizio 763** *(più impegnativo)*

Aggiungi una scadenza ai token: ogni token ha una data di creazione salvata in `token.json`. Se il token ha più di 24 ore, viene considerato scaduto e la richiesta viene rifiutata con `401`. La funzione `verifica_token` deve controllare la scadenza.

> 💡 *Suggerimento:* 
> 
> Usa `datetime.datetime.now().isoformat()` per salvare la data di creazione e `datetime.datetime.fromisoformat()` per rileggerla.

<br>
<br>
<br>
