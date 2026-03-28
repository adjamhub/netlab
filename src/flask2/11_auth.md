# Autenticazione utenti

In questo capitolo affronteremo i concetti relativi alle sessioni e all'autenticazione in una applicazione gestita con Flask.


## Sessioni

HTTP è un protocollo **stateless**: ogni richiesta è indipendente dalla precedente. Il server non ricorda chi sei tra una richiesta e l'altra.

Le **sessioni** risolvono questo problema: permettono di salvare informazioni associate a un utente per tutta la durata della sua visita. 
Flask salva i dati di sessione in un cookie firmato con la `secret_key`.

```python
from flask import Flask, session

app = Flask(__name__)
app.secret_key = 'chiave_super_segretissima'

@app.route('/set')
def set_sessione():
    session['utente'] = 'Marco'
    return 'Sessione impostata'

@app.route('/get')
def get_sessione():
    utente = session.get('utente', 'nessuno')
    return f'Utente in sessione: {utente}'

@app.route('/clear')
def clear_sessione():
    session.clear()
    return 'Sessione cancellata'
```

`session` si comporta come un dizionario Python — puoi leggere, scrivere e cancellare valori. 
I dati persistono tra una richiesta e l'altra finché il browser non chiude la sessione o finché non la cancelli esplicitamente.

---

### Esercizi

**Esercizio 601**

Crea una piccola app con tre route: `/imposta/<nome>` che salva il nome in sessione, 
`/chi-sono` che mostra il nome salvato (o "ospite" se non c'è), 
e `/esci` che cancella la sessione e reindirizza a `/chi-sono`.

---

## Flask-Login

Gestire l'autenticazione a mano con le sessioni è possibile, ma richiede di ripetere lo stesso codice in ogni route protetta:

```python
@app.route('/dashboard')
def dashboard():
    if 'utente' not in session:
        return redirect( '/login' )
    return render_template('dashboard.html')
```

Con molte route protette questo diventa noioso e soggetto a errori. **Flask-Login** automatizza tutto questo. E' un modulo ausiliario 
per Flask, che puoi installare come al solito:


```bash
pip install flask-login
```

Struttura il progetto in questo modo:

```
progetto/
├── app.py
├── utenti.json
└── templates/
    ├── base.html
    ├── login.html
    └── dashboard.html
```

---

### Il modello utente

Flask-Login richiede che il modello utente implementi quattro proprietà e un metodo. 
Il modo più semplice è ereditare da `UserMixin`, che le implementa tutte con valori di default ragionevoli:

```python
from flask_login import UserMixin

class Utente(UserMixin):
    def __init__(self, id, username, password):
        self.id = id
        self.username = username
        self.password = password
```

Le proprietà fornite da `UserMixin`:
- `is_authenticated` — `True` se l'utente è loggato
- `is_active` — `True` se l'account è attivo
- `is_anonymous` — `True` se non è loggato
- `get_id()` — restituisce l'id come stringa

---

### Setup di Flask-Login

```python
from flask import Flask, render_template, request, redirect, url_for, flash
from flask_login import LoginManager, UserMixin, login_user, logout_user, login_required, current_user
import json

app = Flask(__name__)
app.secret_key = 'chiave_segreta'

# Inizializza Flask-Login
login_manager = LoginManager(app)
login_manager.login_view = 'login'  # route a cui reindirizzare se non loggato
login_manager.login_message = 'Devi effettuare il login per accedere a questa pagina.'
login_manager.login_message_category = 'errore'
```

`login_view` è il nome della funzione (non l'URL) a cui Flask-Login reindirizza automaticamente l'utente quando tenta di accedere a una route protetta senza essere loggato.

---

### Gestione utenti su file JSON

Gli utenti sono salvati in `utenti.json`:

```json
[
    {"id": "1", "username": "admin", "password": "admin123"},
    {"id": "2", "username": "mario", "password": "mario456"}
]
```

> ⚠️ ATTENZIONE!
> 
> In questo primo esempio salviamo le password in chiaro per semplicità. 
> In una applicazione reale le password vanno sempre **hashate** — lo vedremo nella sezione dedicata.

Funzioni di utilità per leggere gli utenti e cercarli:

```python
def leggi_utenti():
    try:
        file = open('utenti.json', 'r')
        users = file.read()
        file.close()
        return json.loads(users)
    except (FileNotFoundError, json.JSONDecodeError):
        return []

def trova_utente_per_id(id):
    for u in leggi_utenti():
        if u['id'] == str(id):
            return Utente(u['id'], u['username'], u['password'])
    return None

def trova_utente_per_username(username):
    for u in leggi_utenti():
        if u['username'] == username:
            return Utente(u['id'], u['username'], u['password'])
    return None
```

---

### Il user_loader

Flask-Login ha bisogno di sapere come ricaricare un utente dalla sessione ad ogni richiesta. 
Si definisce con il decorator `@login_manager.user_loader`:

```python
@login_manager.user_loader
def carica_utente(id):
    return trova_utente_per_id(id)
```

Flask-Login chiama questa funzione automaticamente ad ogni richiesta, passando l'id salvato in sessione. 
Se la funzione restituisce `None`, l'utente viene considerato non loggato.

---

### Login e logout

```python
@app.route('/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('dashboard'))

    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        password = request.form.get('password', '').strip()

        utente = trova_utente_per_username(username)

        if not utente or utente.password != password:
            flash('Credenziali non valide.', 'errore')
            return redirect(url_for('login'))

        login_user(utente)
        flash(f'Benvenuto, {utente.username}!', 'successo')
        return redirect(url_for('dashboard'))

    return render_template('login.html')


@app.route('/logout')
@login_required
def logout():
    logout_user()
    flash('Hai effettuato il logout.', 'info')
    return redirect(url_for('login'))
```

`login_user(utente)` salva l'utente in sessione. `logout_user()` la cancella.

Il template `login.html`:

```html
{% extends 'base.html' %}

{% block titolo %}Login{% endblock %}

{% block contenuto %}
<h1>Login</h1>

<form action="/login" method="POST">
    <label>Username:</label><br>
    <input type="text" name="username"><br><br>

    <label>Password:</label><br>
    <input type="password" name="password"><br><br>

    <button type="submit">Accedi</button>
</form>
{% endblock %}
```

---

### 2.8 Proteggere le route

Il decorator `@login_required` protegge una route: se l'utente non è loggato, viene reindirizzato automaticamente alla `login_view` definita in precedenza.

```python
@app.route('/dashboard')
@login_required
def dashboard():
    return render_template('dashboard.html')
```

Nel template puoi accedere all'utente corrente con la variabile `current_user`, disponibile automaticamente in tutti i template:

```html
{% extends 'base.html' %}

{% block titolo %}Dashboard{% endblock %}

{% block contenuto %}
<h1>Ciao, {{ current_user.username }}!</h1>
<p>Sei loggato con successo.</p>
<a href="/logout">Esci</a>
{% endblock %}
```

---

### 2.9 Hashare le password

Salvare le password in chiaro è una pessima pratica. Con la libreria `werkzeug` — già inclusa in Flask — possiamo hashare le password facilmente:

```python
from werkzeug.security import generate_password_hash, check_password_hash

# Quando si registra un utente
password_hashata = generate_password_hash('mia_password')

# Quando si verifica il login
ok = check_password_hash(password_hashata, 'mia_password')  # True
ok = check_password_hash(password_hashata, 'password_sbagliata')  # False
```

Il file `utenti.json` con password hashate:

```json
[
    {
        "id": "1",
        "username": "admin",
        "password": "pbkdf2:sha256:..."
    }
]
```

La verifica nel login diventa:

```python
from werkzeug.security import check_password_hash

if not utente or not check_password_hash(utente.password, password):
    flash('Credenziali non valide.', 'errore')
    return redirect(url_for('login'))
```

---

### Esercizi

**Esercizio 621**

Completa l'app con Flask-Login: crea il file `utenti.json` con almeno due utenti, implementa login e logout, e proteggi la route `/dashboard`. Verifica che un utente non loggato venga reindirizzato al login.

--

**Esercizio 622**

Aggiungi una route `/profilo` protetta che mostri username e id dell'utente corrente usando `current_user`.

--

**Esercizio 623** *(più impegnativo)*

Aggiungi una route `/registrazione` che permetta di creare un nuovo utente. I dati vanno salvati in `utenti.json` con la password hashata usando `generate_password_hash`. Gestisci il caso in cui lo username scelto sia già presente.

<br>
<br>
<br>
