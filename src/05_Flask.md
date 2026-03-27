# Capitolo 5 — API REST

## Obiettivi
Al termine di questo capitolo sarai in grado di:
- capire cos'è un'API REST e come si differenzia da una web app classica
- restituire dati in formato JSON con `jsonify`
- ricevere dati JSON in ingresso con `request.get_json()`
- gestire i codici di stato HTTP in modo appropriato
- organizzare il codice con i Blueprint

---

## Parte 1 — Cos'è un'API REST

### 1.1 Web app vs API

Fino ad ora Flask ha restituito pagine HTML destinate al browser. Un'**API** (Application Programming Interface) funziona diversamente: invece di HTML, restituisce dati — tipicamente in formato JSON — che possono essere consumati da qualsiasi client: un'altra applicazione, un'app mobile, uno script Python, un dispositivo IoT.

Il confronto è semplice:

| | Web app | API REST |
|---|---|---|
| Risponde con | HTML | JSON |
| Destinatario | Browser umano | Altro software |
| Interazione | Form, click | Richieste HTTP programmatiche |

Le due cose non si escludono: una stessa applicazione Flask può avere sia pagine HTML per l'utente, sia endpoint API per altri client.

---

### 1.2 I principi REST

REST (Representational State Transfer) è uno stile architetturale per le API. I concetti chiave sono:

- Le **risorse** sono identificate da URL (es. `/api/libri`, `/api/libri/3`)
- Le **operazioni** sono espresse dai metodi HTTP:

| Metodo | Operazione | Esempio |
|---|---|---|
| GET | Leggi | `GET /api/libri` → lista tutti i libri |
| GET | Leggi uno | `GET /api/libri/3` → leggi il libro 3 |
| POST | Crea | `POST /api/libri` → aggiungi un libro |
| PUT | Sostituisci | `PUT /api/libri/3` → sostituisci il libro 3 |
| DELETE | Elimina | `DELETE /api/libri/3` → elimina il libro 3 |

---

## Parte 2 — Rispondere con JSON

### 2.1 `jsonify`

Flask mette a disposizione la funzione `jsonify` per restituire dati JSON in modo corretto — imposta automaticamente l'header `Content-Type: application/json`:

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/api/saluto')
def saluto():
    return jsonify({'messaggio': 'Ciao dal server!'})
```

Puoi restituire dizionari, liste, o qualsiasi struttura serializzabile in JSON:

```python
@app.route('/api/numeri')
def numeri():
    return jsonify([1, 2, 3, 4, 5])
```

---

### 2.2 Codici di stato

Con `jsonify` puoi specificare il codice di stato HTTP come secondo valore di ritorno:

```python
@app.route('/api/libri/<int:indice>')
def get_libro(indice):
    libri = leggi_libri()

    if indice < 0 or indice >= len(libri):
        return jsonify({'errore': 'Libro non trovato'}), 404

    return jsonify(libri[indice]), 200
```

Restituire il codice corretto è importante: i client lo usano per capire se la richiesta è andata a buon fine senza dover interpretare il contenuto della risposta.

---

### 2.3 API in sola lettura — GET

Costruiamo i primi due endpoint GET per la nostra lista libri:

```python
from flask import Flask, jsonify
import json

app = Flask(__name__)
FILE = 'libri.json'

def leggi_libri():
    try:
        with open(FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return []

# Lista tutti i libri
@app.route('/api/libri', methods=['GET'])
def get_libri():
    return jsonify(leggi_libri()), 200

# Legge un singolo libro per indice
@app.route('/api/libri/<int:indice>', methods=['GET'])
def get_libro(indice):
    libri = leggi_libri()
    if indice < 0 or indice >= len(libri):
        return jsonify({'errore': 'Libro non trovato'}), 404
    return jsonify(libri[indice]), 200
```

Puoi testare i endpoint GET direttamente dal browser, oppure con il modulo `requests` di Python:

```python
import requests

r = requests.get('http://127.0.0.1:5000/api/libri')
print(r.status_code)  # 200
print(r.json())       # lista dei libri

r = requests.get('http://127.0.0.1:5000/api/libri/0')
print(r.json())       # primo libro
```

---

### Esercizi — Parte 2

**Esercizio 2.1**
Aggiungi un endpoint `GET /api/libri/letti` che restituisca solo i libri con `letto: true`. Se non ce ne sono, restituisci una lista vuota con codice `200`.

**Esercizio 2.2**
Aggiungi un endpoint `GET /api/libri/cerca?autore=...` che filtri i libri per autore (ricerca case-insensitive). Se il parametro `autore` non è presente, restituisci un errore `400` con un messaggio appropriato.

---

## Parte 3 — Ricevere dati JSON

### 3.1 `request.get_json()`

Quando un client invia dati JSON nel corpo della richiesta POST, si leggono con `request.get_json()`:

```python
from flask import request

@app.route('/api/libri', methods=['POST'])
def add_libro():
    dati = request.get_json()

    if not dati:
        return jsonify({'errore': 'Dati JSON mancanti'}), 400

    titolo = dati.get('titolo', '').strip()
    autore = dati.get('autore', '').strip()

    if not titolo or not autore:
        return jsonify({'errore': 'Titolo e autore sono obbligatori'}), 400

    libri = leggi_libri()
    libri.append({'titolo': titolo, 'autore': autore, 'letto': False})
    salva_libri(libri)

    return jsonify({'messaggio': f'"{titolo}" aggiunto'}), 201
```

Il codice `201 Created` indica che una nuova risorsa è stata creata con successo.

Per testare il POST con `requests`:

```python
import requests

r = requests.post(
    'http://127.0.0.1:5000/api/libri',
    json={'titolo': 'Il nome della rosa', 'autore': 'Umberto Eco'}
)
print(r.status_code)  # 201
print(r.json())       # {'messaggio': '"Il nome della rosa" aggiunto'}
```

---

### 3.2 DELETE

```python
@app.route('/api/libri/<int:indice>', methods=['DELETE'])
def delete_libro(indice):
    libri = leggi_libri()

    if indice < 0 or indice >= len(libri):
        return jsonify({'errore': 'Libro non trovato'}), 404

    titolo = libri[indice]['titolo']
    libri.pop(indice)
    salva_libri(libri)

    return jsonify({'messaggio': f'"{titolo}" eliminato'}), 200
```

```python
import requests

r = requests.delete('http://127.0.0.1:5000/api/libri/0')
print(r.status_code)  # 200
print(r.json())       # {'messaggio': '... eliminato'}
```

---

### 3.3 PUT — aggiornamento completo

```python
@app.route('/api/libri/<int:indice>', methods=['PUT'])
def update_libro(indice):
    libri = leggi_libri()

    if indice < 0 or indice >= len(libri):
        return jsonify({'errore': 'Libro non trovato'}), 404

    dati = request.get_json()
    if not dati:
        return jsonify({'errore': 'Dati JSON mancanti'}), 400

    titolo = dati.get('titolo', '').strip()
    autore = dati.get('autore', '').strip()
    letto  = dati.get('letto', False)

    if not titolo or not autore:
        return jsonify({'errore': 'Titolo e autore sono obbligatori'}), 400

    libri[indice] = {'titolo': titolo, 'autore': autore, 'letto': letto}
    salva_libri(libri)

    return jsonify({'messaggio': f'"{titolo}" aggiornato'}), 200
```

```python
import requests

r = requests.put(
    'http://127.0.0.1:5000/api/libri/0',
    json={'titolo': 'Il nome della rosa', 'autore': 'Umberto Eco', 'letto': True}
)
print(r.status_code)  # 200
print(r.json())       # {'messaggio': '... aggiornato'}
```

---

### Esercizi — Parte 3

**Esercizio 3.1**
Testa tutti gli endpoint con `curl` o con un client REST a scelta (es. Bruno, Postman, o semplicemente il browser per i GET). Verifica che i codici di stato siano corretti in ogni caso.

**Esercizio 3.2**
Scrivi uno script Python separato `client.py` che usi il modulo `requests` per:
1. aggiungere tre libri tramite POST
2. leggere la lista completa tramite GET
3. eliminare il secondo libro tramite DELETE
4. rileggere la lista per verificare il risultato

> 💡 *Suggerimento:* installa `requests` con `pip install requests`. Per inviare JSON usa `requests.post(url, json={...})`.

---

## Parte 4 — Blueprint

### 4.1 Perché i Blueprint

Finora tutto il codice sta in `app.py`. Man mano che l'applicazione cresce, diventa difficile da leggere e mantenere. I **Blueprint** di Flask permettono di suddividere le route in moduli separati.

---

### 4.2 Creare un Blueprint

Creiamo un file separato `api.py` per tutte le route dell'API:

```python
# api.py
from flask import Blueprint, jsonify, request
import json

api = Blueprint('api', __name__)

FILE = 'libri.json'

def leggi_libri():
    try:
        with open(FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return []

def salva_libri(libri):
    with open(FILE, 'w', encoding='utf-8') as f:
        json.dump(libri, f, indent=2, ensure_ascii=False)

@api.route('/libri', methods=['GET'])
def get_libri():
    return jsonify(leggi_libri()), 200

@api.route('/libri/<int:indice>', methods=['GET'])
def get_libro(indice):
    libri = leggi_libri()
    if indice < 0 or indice >= len(libri):
        return jsonify({'errore': 'Libro non trovato'}), 404
    return jsonify(libri[indice]), 200

# ... altri endpoint
```

---

### 4.3 Registrare il Blueprint

In `app.py` si registra il Blueprint con un prefisso URL:

```python
# app.py
from flask import Flask
from api import api

app = Flask(__name__)
app.secret_key = 'chiave_segreta'

app.register_blueprint(api, url_prefix='/api')

if __name__ == '__main__':
    app.run(debug=True)
```

Con `url_prefix='/api'`, tutte le route definite nel Blueprint saranno automaticamente precedute da `/api`. La route `/libri` nel Blueprint diventa `/api/libri` nell'applicazione.

La struttura finale del progetto:

```
corso_flask/
├── app.py
├── api.py
├── libri.json
└── templates/
    ├── base.html
    ├── index.html
    └── modifica.html
```

---

### Esercizio — Parte 4

**Esercizio 4.1**
Riorganizza il progetto usando i Blueprint: sposta tutte le route API in `api.py` e tutte le route della web app in `views.py`. Il file `app.py` deve contenere solo la creazione dell'app e la registrazione dei due Blueprint.

---

## Riepilogo

| Concetto | Descrizione |
|---|---|
| `jsonify(dati)` | Restituisce una risposta JSON |
| `request.get_json()` | Legge il corpo JSON della richiesta |
| `GET` | Legge dati |
| `POST` | Crea una nuova risorsa |
| `PUT` | Aggiorna una risorsa esistente |
| `DELETE` | Elimina una risorsa |
| `200 OK` | Richiesta andata a buon fine |
| `201 Created` | Risorsa creata con successo |
| `400 Bad Request` | Dati mancanti o non validi |
| `404 Not Found` | Risorsa non trovata |
| `Blueprint` | Modulo per organizzare le route |
