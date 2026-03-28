# File JSON come storage



Già conosci il formato JSON e il modulo `json` di Python. In Flask lo usiamo per mantenere i dati tra una richiesta e l'altra, salvandoli su file.
Useremo codice tipo il seguente:

```python
import json

# caricare i dati da file
file = open('dati.json', 'r')
contenuto = file.read()
file.close()
dati = json.loads(contenuto)

# ... cose ...

# Salvare i dati su file
file = open('dati.json', 'w')
dati_json = json.dumps(dizionario_dati, indent=2, ensure_ascii=False)
file.write(dati_json)
file.close()
```

Il parametro `indent=2` formatta il JSON in modo leggibile. Il parametro `ensure_ascii=False` permette di salvare correttamente caratteri accentati.

---

### Il pattern load → modifica → save

Ogni volta che vogliamo aggiornare i dati su file, seguiamo sempre questo schema:

```python
import json

# 1. Leggi il file
file = open('dati.json', 'r')
contenuto = file.read()
file.close()
dizionario_dati = json.loads(contenuto)

# 2. Modifica i dati in memoria
dizionario_dati.append({'nome': 'nuovo elemento'})

# 3. Riscrivi il file
file = open('dati.json', 'w')
dati_json = json.dumps(dizionario_dati, indent=2, ensure_ascii=False)
file.write(dati_json)
file.close()
```

Due errori sono comuni quando si lavora con file JSON in Flask:

- **`FileNotFoundError`** — il file non esiste ancora (prima esecuzione dell'app)
- **`json.JSONDecodeError`** — il file esiste ma il contenuto non è JSON valido

È buona pratica gestirli sempre insieme.

```python
try:
    file = open('dati.json', 'r')
    contenuto = file.read()
    file.close()
    dizionario_dati = json.loads(contenuto)
except (FileNotFoundError, json.JSONDecodeError):
    dizionario_dati = []  # valore di default
```

---

### Funzioni di utilità

Quando un'applicazione legge e scrive spesso lo stesso file, conviene creare due funzioni di utilità per non ripetere il codice:

```python
FILE = 'dati.json'

def leggi_dati():
    try:
        file = open(FILE, 'r')
        contenuto = file.read()
        file.close()
        dizionario_dati = json.loads(contenuto)
    except (FileNotFoundError, json.JSONDecodeError):
        dizionario_dati = []  # valore di default
        return dizionario_dati
        
def salva_dati(dati):
    file = open(FILE, 'w')
    dati_json = json.dumps(dati, indent=2, ensure_ascii=False)
    file.write(dati_json)
    file.close()
```

D'ora in poi nel codice useremo solo `leggi_dati()` e `salva_dati()`.

---

### Esercizio

**Esercizio 401**

Scrivi uno script Python (senza Flask) che:
1. legga una lista di nomi da `nomi.json` gestendo il caso in cui il file non esista
2. aggiunga un nome passato da tastiera con `input()`
3. salvi la lista aggiornata su file

Eseguilo più volte e verifica che i nomi si accumulino correttamente.

---

## CRUD con Flask e JSON

***CRUD*** è l'acronimo delle quattro operazioni fondamentali sui dati:

| Operazione | Significato                    | HTTP        |
|------------|--------------------------------|-------------|
| **C**reate | Crea un nuovo elemento         | POST        |
| **R**ead   | Leggi / visualizza             | GET         |
| **U**pdate | Modifica un elemento esistente | POST        |
| **D**elete | Elimina un elemento            | POST / GET  |

Costruiremo una piccola app per gestire una lista di libri da leggere.

---

### Struttura del progetto

```
libreria_flask/
├── app.py
├── libri.json          ← creato automaticamente alla prima aggiunta
└── templates/
    ├── base.html
    ├── index.html
    └── modifica.html
```

---

### Setup iniziale

```python
from flask import Flask, render_template, request, flash, redirect, url_for
import json

app = Flask(__name__)
app.secret_key = 'chiave_super_segretissima'

FILE = 'libri.json'

def leggi_dati():
    try:
        file = open(FILE, 'r')
        contenuto = file.read()
        file.close()
        dizionario_dati = json.loads(contenuto)
    except (FileNotFoundError, json.JSONDecodeError):
        dizionario_dati = []  # valore di default
        return dizionario_dati
        
def salva_dati(dati):
    file = open(FILE, 'w')
    dati_json = json.dumps(dati, indent=2, ensure_ascii=False)
    file.write(dati_json)
    file.close()
```

---

### Read e Create

La pagina principale mostra la lista dei libri e un form per aggiungerne uno nuovo:

```python
@app.route('/', methods=['GET', 'POST'])
def index():
    if request.method == 'POST':
        titolo = request.form.get('titolo', '').strip()
        autore = request.form.get('autore', '').strip()

        if not titolo or not autore:
            flash('Titolo e autore sono obbligatori.', 'errore')
            return redirect( '/' )

        libri = leggi_dati()

        # Controlla duplicati
        for l in libri:
            if l['titolo'].lower() == titolo.lower():
                flash('Questo libro è già nella lista.', 'errore')
                return redirect( '/' )

        libri.append({'titolo': titolo, 'autore': autore, 'letto': False})
        salva_dati(libri)

        flash(f'"{titolo}" aggiunto alla lista.', 'successo')
        return redirect( '/' )

    libri = leggi_dati()
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

### Delete

```python
@app.route('/elimina/<int:indice>')
def elimina(indice):
    libri = leggi_dati()

    if indice < 0 or indice >= len(libri):
        flash('Libro non trovato.', 'errore')
        return redirect( '/' )
        
    titolo = libri[indice]['titolo']
    libri.pop(indice)
    salva_dati(libri)

    flash(f'"{titolo}" eliminato.', 'successo')
    return redirect( '/' )
```

---

### Update

Per la modifica usiamo un indice nell'URL e una pagina dedicata:

```python
@app.route('/modifica/<int:indice>', methods=['GET', 'POST'])
def modifica(indice):
    libri = leggi_dati()

    if indice < 0 or indice >= len(libri):
        flash('Libro non trovato.', 'errore')
        return redirect( '/' )
        
    if request.method == 'POST':
        titolo = request.form.get('titolo', '').strip()
        autore = request.form.get('autore', '').strip()
        letto  = request.form.get('letto') == 'on'

        if not titolo or not autore:
            flash('Titolo e autore sono obbligatori.', 'errore')
            return redirect( f'/modifica/{indice}' )
            
        libri[indice] = {'titolo': titolo, 'autore': autore, 'letto': letto}
        salva_dati(libri)

        flash(f'"{titolo}" aggiornato.', 'successo')
        return redirect( '/' )
        
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

### Esercizi

**Esercizio 421**

Esegui l'app completa e verifica che tutte le operazioni CRUD funzionino correttamente. Apri `libri.json` con un editor di testo dopo ogni operazione e osserva come cambia il contenuto.

--

**Esercizio 422**

Aggiungi all'app un campo `anno` (anno di pubblicazione). Aggiorna il form di aggiunta, il form di modifica e la visualizzazione nella lista. Valida che l'anno sia un numero intero di 4 cifre.

--

**Esercizio 423** *(più impegnativo)*

Aggiungi una pagina `/cerca` con un form che permetta di cercare libri per autore. Il risultato deve essere una lista filtrata dei libri che contengono il termine cercato nel campo autore (ricerca case-insensitive). Se non viene trovato nessun risultato, mostra un messaggio appropriato.

<br>
<br>
<br>
