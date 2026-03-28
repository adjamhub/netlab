# Upload di file

In questo capitolo parleremo di:
- gestire l'upload di file tramite form HTML
- validare il tipo e la dimensione del file
- salvare i file su disco in modo sicuro
- servire i file caricati agli utenti

---

## Upload di base

Per permettere l'upload di file, il form HTML deve avere due accorgimenti rispetto a un form normale:
- `enctype="multipart/form-data"` — indica al browser di inviare il file come dati binari
- `<input type="file">` — il campo per selezionare il file

```html
<form action="/upload" method="POST" enctype="multipart/form-data">
    <label>Scegli un file:</label><br>
    <input type="file" name="file"><br><br>
    <button type="submit">Carica</button>
</form>
```

Senza `enctype="multipart/form-data"` il file non viene inviato correttamente — è l'errore più comune.

---

### Ricevere il file in Flask

Il file caricato si legge con `request.files`, in modo analogo a `request.form` per i campi di testo:

```python
from flask import Flask, request, flash, redirect, url_for

app = Flask(__name__)
app.secret_key = 'chiave_segreta'

@app.route('/upload', methods=['GET', 'POST'])
def upload():
    if request.method == 'POST':
        file = request.files.get('file')

        if not file or file.filename == '':
            flash('Nessun file selezionato.', 'errore')
            return redirect(url_for('upload'))

        file.save('uploads/' + file.filename)
        flash(f'File "{file.filename}" caricato con successo.', 'successo')
        return redirect(url_for('upload'))

    return render_template('upload.html')
```

`file.filename` contiene il nome originale del file. `file.save(percorso)` lo salva su disco.

---

### Creare la cartella uploads

La cartella `uploads/` deve esistere prima di salvare i file. Puoi crearla automaticamente all'avvio dell'app:

```python
import os

UPLOAD_FOLDER = 'uploads'
os.makedirs(UPLOAD_FOLDER, exist_ok=True)

app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
```

`os.makedirs(..., exist_ok=True)` crea la cartella se non esiste, senza errori se esiste già.

La route di upload diventa:

```python
file.save(os.path.join(app.config['UPLOAD_FOLDER'], file.filename))
```

---

### Esercizi

**Esercizio 641**

Crea una app con una route `/upload` che permetta di caricare un file qualsiasi. 
Dopo il caricamento, mostra un flash message di conferma con il nome del file. Verifica che il file sia presente nella cartella `uploads/`.

---

## Validazione

Accettare qualsiasi file è pericoloso. È buona pratica limitare i tipi di file consentiti verificando l'estensione:

```python
ESTENSIONI_CONSENTITE = {'png', 'jpg', 'jpeg', 'gif', 'pdf'}

def estensione_consentita(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ESTENSIONI_CONSENTITE
```

`rsplit('.', 1)` divide il nome del file sull'ultimo punto, restituendo nome ed estensione. Il controllo `'.' in filename` evita file senza estensione.

La route aggiornata:

```python
if not estensione_consentita(file.filename):
    flash('Tipo di file non consentito.', 'errore')
    return redirect(url_for('upload'))
```

---

### Nomi sicuri con `secure_filename`

Il nome del file fornito dall'utente non è affidabile — potrebbe contenere caratteri speciali o percorsi come `../../etc/passwd` che potrebbero sovrascrivere file di sistema.

Werkzeug mette a disposizione `secure_filename` che pulisce il nome del file:

```python
from werkzeug.utils import secure_filename

filename = secure_filename(file.filename)
# 'My File (1).jpg'  →  'My_File_1_.jpg'
# '../../etc/passwd' →  'etc_passwd'
```

Il salvataggio sicuro diventa:

```python
filename = secure_filename(file.filename)
file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
```

---

### Evitare sovrascritture

Se due utenti caricano un file con lo stesso nome, il secondo sovrascrive il primo. Una soluzione semplice è aggiungere un prefisso univoco al nome del file usando `uuid`:

```python
import uuid

def nome_univoco(filename):
    estensione = filename.rsplit('.', 1)[1].lower()
    return f"{uuid.uuid4().hex}.{estensione}"
```

`uuid.uuid4().hex` genera una stringa esadecimale casuale di 32 caratteri — praticamente impossibile da indovinare o ripetere.

```python
filename = nome_univoco(secure_filename(file.filename))
file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
```

---

### Limitare la dimensione del file

Flask permette di impostare una dimensione massima per le richieste in ingresso:

```python
app.config['MAX_CONTENT_LENGTH'] = 2 * 1024 * 1024  # 2 MB
```

Se il file supera il limite, Flask solleva automaticamente un errore `413 Request Entity Too Large`. Puoi intercettarlo con un handler personalizzato:

```python
from flask import abort
from werkzeug.exceptions import RequestEntityTooLarge

@app.errorhandler(RequestEntityTooLarge)
def file_troppo_grande(e):
    flash('Il file supera la dimensione massima consentita (2 MB).', 'errore')
    return redirect(url_for('upload'))
```

---

### Esercizi

**Esercizio 661**

Aggiorna l'app dell'esercizio precedente aggiungendo:
- validazione dell'estensione (solo immagini: `png`, `jpg`, `jpeg`, `gif`)
- nome del file reso sicuro con `secure_filename`
- dimensione massima di 1 MB

--

**Esercizio 662**

Modifica l'app in modo che ogni file venga salvato con un nome univoco generato con `uuid`. 
Salva in `uploads.json` un registro dei file caricati con il nome originale, il nome salvato su disco e la data di caricamento.

---

## Servire i file e galleria

I file nella cartella `uploads/` non sono automaticamente accessibili dal browser. Per servirli bisogna creare una route apposita con `send_from_directory`:

```python
from flask import send_from_directory

@app.route('/uploads/<filename>')
def file_caricato(filename):
    return send_from_directory(app.config['UPLOAD_FOLDER'], filename)
```

Ora il file `uploads/immagine.jpg` è raggiungibile all'URL `/uploads/immagine.jpg`.

---

### Una galleria immagini completa

Mettiamo tutto insieme: un'app che permette di caricare immagini e le mostra in una galleria.

```python
import os
import uuid
import json
from datetime import date
from flask import Flask, render_template, request, flash, redirect, url_for, send_from_directory
from werkzeug.utils import secure_filename
from werkzeug.exceptions import RequestEntityTooLarge

app = Flask(__name__)
app.secret_key = 'chiave_segreta'
app.config['UPLOAD_FOLDER'] = 'uploads'
app.config['MAX_CONTENT_LENGTH'] = 2 * 1024 * 1024

ESTENSIONI_CONSENTITE = {'png', 'jpg', 'jpeg', 'gif'}
REGISTRO = 'uploads.json'

os.makedirs(app.config['UPLOAD_FOLDER'], exist_ok=True)

def estensione_consentita(filename):
    return '.' in filename and \
           filename.rsplit('.', 1)[1].lower() in ESTENSIONI_CONSENTITE

def nome_univoco(filename):
    estensione = filename.rsplit('.', 1)[1].lower()
    return f"{uuid.uuid4().hex}.{estensione}"

def leggi_registro():
    try:
        with open(REGISTRO, 'r', encoding='utf-8') as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return []

def salva_registro(dati):
    with open(REGISTRO, 'w', encoding='utf-8') as f:
        json.dump(dati, f, indent=2, ensure_ascii=False)

@app.errorhandler(RequestEntityTooLarge)
def file_troppo_grande(e):
    flash('Il file supera la dimensione massima consentita (2 MB).', 'errore')
    return redirect(url_for('galleria'))

@app.route('/', methods=['GET', 'POST'])
def galleria():
    if request.method == 'POST':
        file = request.files.get('file')

        if not file or file.filename == '':
            flash('Nessun file selezionato.', 'errore')
            return redirect(url_for('galleria'))

        if not estensione_consentita(file.filename):
            flash('Tipo di file non consentito.', 'errore')
            return redirect(url_for('galleria'))

        filename_sicuro = secure_filename(file.filename)
        filename_salvato = nome_univoco(filename_sicuro)
        file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename_salvato))

        registro = leggi_registro()
        registro.append({
            'nome_originale': filename_sicuro,
            'nome_salvato': filename_salvato,
            'data': date.today().isoformat()
        })
        salva_registro(registro)

        flash(f'"{filename_sicuro}" caricato con successo.', 'successo')
        return redirect(url_for('galleria'))

    immagini = leggi_registro()
    return render_template('galleria.html', immagini=immagini)

@app.route('/uploads/<filename>')
def file_caricato(filename):
    return send_from_directory(app.config['UPLOAD_FOLDER'], filename)
```

Il template `galleria.html`:

```html
{% extends 'base.html' %}

{% block titolo %}Galleria{% endblock %}

{% block contenuto %}
<h1>Galleria immagini</h1>

<form action="/" method="POST" enctype="multipart/form-data">
    <input type="file" name="file">
    <button type="submit">Carica</button>
</form>

<div style="display: flex; flex-wrap: wrap; gap: 10px; margin-top: 20px;">
{% for img in immagini %}
    <div>
        <img src="/uploads/{{ img.nome_salvato }}" style="width: 200px; height: 150px; object-fit: cover;">
        <p style="font-size: 0.85em;">{{ img.nome_originale }}<br>{{ img.data }}</p>
    </div>
{% endfor %}
</div>
{% endblock %}
```

---

### Esercizi

**Esercizio 681**

Aggiungi alla galleria la possibilità di eliminare un'immagine. 
La cancellazione deve rimuovere sia il file dalla cartella `uploads/` che la voce corrispondente in `uploads.json`.

> 💡 *Suggerimento:* 
> 
> usa `os.remove(percorso)` per eliminare un file dal disco.

--

**Esercizio 682** *(più impegnativo)*

Proteggi la route di upload con `@login_required` usando Flask-Login (vedi Capitolo 1). 
Solo gli utenti loggati possono caricare immagini, ma tutti possono vedere la galleria.

<br>
<br>
<br>
