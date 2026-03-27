# Capitolo 4 — File JSON come storage

## Obiettivi
Al termine di questo capitolo sarai in grado di:
- leggere e scrivere file JSON con Python in un contesto Flask
- applicare il pattern load → modifica → save
- gestire gli errori più comuni (file mancante, JSON malformato)
- costruire una piccola web app CRUD con Flask e file JSON

---

## Parte 1 — JSON con Flask

### 1.1 Lettura e scrittura su file

Già conosci il formato JSON e il modulo `json` di Python. In Flask lo usiamo per mantenere i dati tra una richiesta e l'altra, salvandoli su file.

Le due funzioni che useremo sono `json.load` e `json.dump`:

```python
import json

# Legge da file
with open('dati.json', 'r', encoding='utf-8') as f:
    dati = json.load(f)

# Scrive su file
with open('dati.json', 'w', encoding='utf-8') as f:
    json.dump(dati, f, indent=2, ensure_ascii=False)
```

Il parametro `indent=2` formatta il JSON in modo leggibile. Il parametro `ensure_ascii=False` permette di salvare correttamente caratteri accentati.

---

### 1.2 Il pattern load → modifica → save

Ogni volta che vogliamo aggiornare i dati su file, seguiamo sempre questo schema:

```python
# 1. Leggi il file
with open('dati.json', 'r', encoding='utf-8') as f:
    dati = json.load(f)

# 2. Modifica i dati in memoria
dati.append({'nome': 'nuovo elemento'})

# 3. Riscrivi il file
with open('dati.json', 'w', encoding='utf-8') as f:
    json.dump(dati, f, indent=2, ensure_ascii=False)
```

---

### 1.3 Gestione degli errori

Due errori sono comuni quando si lavora con file JSON in Flask:

- **`FileNotFoundError`** — il file non esiste ancora (prima esecuzione dell'app)
- **`json.JSONDecodeError`** — il file esiste ma il contenuto non è JSON valido

È buona pratica gestirli sempre insieme:

```python
try:
    with open('dati.json', 'r', encoding='utf-8') as f:
        dati = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    dati = []  # valore di default
```

---

### 1.4 Funzioni di utilità

Quando un'applicazione legge e scrive spesso lo stesso file, conviene creare due funzioni di utilità per non ripetere il codice:

```python
FILE = 'dati.json'

def leggi_dati():
    try:
        with open(FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return []

def salva_dati(dati):
    with open(FILE, 'w', encoding='utf-8') as f:
        json.dump(dati, f, indent=2, ensure_ascii=False)
```

D'ora in poi nel codice useremo solo `leggi_dati()` e `salva_dati()`.

---

### Esercizio — Parte 1

**Esercizio 1.1**
Scrivi uno script Python (senza Flask) che:
1. legga una lista di nomi da `nomi.json` gestendo il caso in cui il file non esista
2. aggiunga un nome passato da tastiera con `input()`
3. salvi la lista aggiornata su file

Eseguilo più volte e verifica che i nomi si accumulino correttamente.

---

## Parte 2 — CRUD con Flask e JSON

### 2.1 Cos'è il CRUD

CRUD è l'acronimo delle quattro operazioni fondamentali sui dati:

| Operazione | Significato | HTTP |
|---|---|---|
| **C**reate | Crea un nuovo elemento | POST |
| **R**ead | Leggi / visualizza | GET |
| **U**pdate | Modifica un elemento esistente | POST |
| **D**elete | Elimina un elemento | POST / GET |

Costruiremo una piccola app per gestire una lista di libri da leggere.

---

### 2.2 Struttura del progetto

```
corso_flask/
├── app.py
├── libri.json          ← creato automaticamente alla prima aggiunta
└── templates/
    ├── base.html
    ├── index.html
    └── modifica.html
```

---

### 2.3 Setup iniziale

```python
from flask import Flask, render_template, request, flash, redirect, url_for
import json

app = Flask(__name__)
app.secret_key = 'chiave_segreta'

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
```

---

### 2.4 Read e Create

La pagina principale mostra la lista dei libri e un form per aggiungerne uno nuovo:

```python
@app.route('/', methods=['GET', 'POST'])
def index():
    if request.method == 'POST':
        titolo = request.form.get('titolo', '').strip()
        autore = request.form.get('autore', '').strip()

        if not titolo or not autore:
            flash('Titolo e autore sono obbligatori.', 'errore')
            return redirect(url_for('index'))

        libri = leggi_libri()

        # Controlla duplicati
        if any(l['titolo'].lower() == titolo.lower() for l in libri):
            flash('Questo libro è già nella lista.', 'errore')
            return redirect(url_for('index'))

        libri.append({'titolo': titolo, 'autore': autore, 'letto': False})
        salva_libri(libri)

        flash(f'"{titolo}" aggiunto alla lista.', 'successo')
        return redirect(url_for('index'))

    libri = leggi_libri()
    return render_template('index.html', libri=libri)
```

Il template `index.html`:

```html
{% extends 'base.html' %}

{% block titolo %}Lista libri{% endblock %}

{% block contenuto %}
<h1>Lista libri</h1>

<form action="/" method="POST">
    <input type="text" name="titolo" placeholder="Titolo">
    <input type="text" name="autore" placeholder="Autore">
    <button type="submit">Aggiungi</button>
</form>

<ul>
{% for libro in libri %}
    <li>
        <strong>{{ libro.titolo }}</strong> — {{ libro.autore }}
        {% if libro.letto %}✅{% else %}📖{% endif %}
        <a href="/elimina/{{ loop.index0 }}">Elimina</a>
        <a href="/modifica/{{ loop.index0 }}">Modifica</a>
    </li>
{% endfor %}
</ul>
{% endblock %}
```

---

### 2.5 Delete

```python
@app.route('/elimina/<int:indice>')
def elimina(indice):
    libri = leggi_libri()

    if indice < 0 or indice >= len(libri):
        flash('Libro non trovato.', 'errore')
        return redirect(url_for('index'))

    titolo = libri[indice]['titolo']
    libri.pop(indice)
    salva_libri(libri)

    flash(f'"{titolo}" eliminato.', 'successo')
    return redirect(url_for('index'))
```

---

### 2.6 Update

Per la modifica usiamo un indice nell'URL e una pagina dedicata:

```python
@app.route('/modifica/<int:indice>', methods=['GET', 'POST'])
def modifica(indice):
    libri = leggi_libri()

    if indice < 0 or indice >= len(libri):
        flash('Libro non trovato.', 'errore')
        return redirect(url_for('index'))

    if request.method == 'POST':
        titolo = request.form.get('titolo', '').strip()
        autore = request.form.get('autore', '').strip()
        letto  = request.form.get('letto') == 'on'

        if not titolo or not autore:
            flash('Titolo e autore sono obbligatori.', 'errore')
            return redirect(url_for('modifica', indice=indice))

        libri[indice] = {'titolo': titolo, 'autore': autore, 'letto': letto}
        salva_libri(libri)

        flash(f'"{titolo}" aggiornato.', 'successo')
        return redirect(url_for('index'))

    return render_template('modifica.html', libro=libri[indice], indice=indice)
```

Il template `modifica.html`:

```html
{% extends 'base.html' %}

{% block titolo %}Modifica libro{% endblock %}

{% block contenuto %}
<h1>Modifica libro</h1>

<form action="/modifica/{{ indice }}" method="POST">
    <label>Titolo:</label><br>
    <input type="text" name="titolo" value="{{ libro.titolo }}"><br><br>

    <label>Autore:</label><br>
    <input type="text" name="autore" value="{{ libro.autore }}"><br><br>

    <label>
        <input type="checkbox" name="letto" {% if libro.letto %}checked{% endif %}>
        Già letto
    </label><br><br>

    <button type="submit">Salva</button>
    <a href="/">Annulla</a>
</form>
{% endblock %}
```

---

### Esercizi — Parte 2

**Esercizio 2.1**
Esegui l'app completa e verifica che tutte le operazioni CRUD funzionino correttamente. Apri `libri.json` con un editor di testo dopo ogni operazione e osserva come cambia il contenuto.

**Esercizio 2.2**
Aggiungi all'app un campo `anno` (anno di pubblicazione). Aggiorna il form di aggiunta, il form di modifica e la visualizzazione nella lista. Valida che l'anno sia un numero intero di 4 cifre.

**Esercizio 2.3** *(più impegnativo)*
Aggiungi una pagina `/cerca` con un form che permetta di cercare libri per autore. Il risultato deve essere una lista filtrata dei libri che contengono il termine cercato nel campo autore (ricerca case-insensitive). Se non viene trovato nessun risultato, mostra un messaggio appropriato.

---

## Riepilogo

| Concetto | Descrizione |
|---|---|
| `json.load(f)` | Legge JSON da file |
| `json.dump(dati, f)` | Scrive JSON su file |
| `indent=2` | Formatta il JSON in modo leggibile |
| `ensure_ascii=False` | Permette caratteri accentati |
| `FileNotFoundError` | Eccezione se il file non esiste |
| `json.JSONDecodeError` | Eccezione se il JSON è malformato |
| Pattern load → modifica → save | Schema base per aggiornare i dati |
| CRUD | Create, Read, Update, Delete |
