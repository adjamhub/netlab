# Struttura avanzata del progetto

In questo capitolo vedremo come:

- organizzare un progetto Flask in moduli separati
- usare l'application factory per creare l'app
- gestire configurazioni diverse per sviluppo e produzione
- strutturare Blueprint multipli in modo professionale

---

## Il problema della crescita

Fino ad ora tutto il codice stava in `app.py`. Per progetti piccoli va bene, 
ma man mano che l'applicazione cresce — login, upload, API, decine di route — un solo file diventa difficile da leggere e mantenere.

Considera un progetto reale con autenticazione e upload:

```
app.py   ← 300+ righe, tutto mescolato
```

L'obiettivo è arrivare a qualcosa di simile:

```
progetto/
├── app.py          ← solo 10 righe: crea e avvia l'app
├── config.py       ← configurazione
├── auth/           ← modulo autenticazione
│   ├── __init__.py
│   └── routes.py
├── galleria/       ← modulo galleria
│   ├── __init__.py
│   └── routes.py
├── static/
└── templates/
    ├── base.html
    ├── auth/
    │   └── login.html
    └── galleria/
        └── galleria.html
```

Ogni modulo ha le sue route, i suoi template, la sua logica — e `app.py` si limita ad assemblarli.

---

## Application Factory

L'**application factory** è una funzione che crea e configura l'app Flask. 
Invece di creare l'app a livello di modulo — come abbiamo fatto finora — la creiamo dentro una funzione chiamata `create_app`:

**Prima (approccio semplice):**
```python
# app.py
app = Flask(__name__)
app.secret_key = 'chiave'
# ... tutto il resto
```

**Dopo (application factory):**
```python
# app.py
from factory import create_app

app = create_app()

if __name__ == '__main__':
    app.run(debug=True)
```

```python
# factory.py
from flask import Flask

def create_app():
    app = Flask(__name__)
    app.secret_key = 'chiave_segreta'
    # registra Blueprint, estensioni, ecc.
    return app
```

Il vantaggio principale è che possiamo creare istanze diverse dell'app con configurazioni diverse — ad esempio una per lo sviluppo e una per la produzione.

---

### 2.2 Configurazione separata

Creiamo un file `config.py` con classi di configurazione:

```python
# config.py
import os

class Config:
    SECRET_KEY = 'chiave_segreta_di_default'
    UPLOAD_FOLDER = 'uploads'
    MAX_CONTENT_LENGTH = 2 * 1024 * 1024  # 2 MB

class DevConfig(Config):
    DEBUG = True

class ProdConfig(Config):
    DEBUG = False
    SECRET_KEY = os.environ.get('SECRET_KEY', 'cambia-questa-chiave')
```

`os.environ.get('SECRET_KEY', ...)` legge la chiave da una variabile d'ambiente — in produzione non vogliamo mai avere segreti nel codice sorgente.

La factory sceglie la configurazione giusta:

```python
# factory.py
from flask import Flask
from config import DevConfig, ProdConfig
import os

def create_app():
    app = Flask(__name__)

    # Sceglie la configurazione in base all'ambiente
    if os.environ.get('FLASK_ENV') == 'production':
        app.config.from_object(ProdConfig)
    else:
        app.config.from_object(DevConfig)

    return app
```

---

### Esercizi

**Esercizio 701**

Prendi uno dei progetti dei capitoli precedenti e riscrivilo usando l'application factory. Crea `factory.py` con la funzione `create_app` e `config.py` con almeno due classi di configurazione (`DevConfig` e `ProdConfig`). Verifica che l'app funzioni esattamente come prima.

---

## Blueprint multipli


Ogni modulo dell'applicazione diventa un **pacchetto Python** — una cartella con un file `__init__.py`. Il Blueprint viene definito nel file `routes.py` del pacchetto.

Creiamo il modulo `auth`:

```
auth/
├── __init__.py   ← vuoto o con importazioni
└── routes.py     ← Blueprint con le route
```

```python
# auth/routes.py
from flask import Blueprint, render_template, request, redirect, url_for, flash
from flask_login import login_user, logout_user, login_required

auth = Blueprint('auth', __name__, template_folder='templates')

@auth.route('/login', methods=['GET', 'POST'])
def login():
    # ... logica di login
    return render_template('auth/login.html')

@auth.route('/logout')
@login_required
def logout():
    logout_user()
    return redirect(url_for('auth.login'))
```

Nota `url_for('auth.login')` — quando si usano Blueprint, il nome della route va prefissato con il nome del Blueprint.

---

### Template per Blueprint

Ogni Blueprint può avere la propria sottocartella di template:

```
templates/
├── base.html
├── auth/
│   ├── login.html
│   └── registrazione.html
└── galleria/
    └── galleria.html
```

Nei template, i link tra moduli diversi usano il prefisso del Blueprint:

```html
<a href="{{ url_for('auth.login') }}">Login</a>
<a href="{{ url_for('galleria.index') }}">Galleria</a>
```

---

### Registrare i Blueprint nella factory

```python
# factory.py
from flask import Flask
from flask_login import LoginManager
from config import DevConfig, ProdConfig
import os

def create_app():
    app = Flask(__name__)

    if os.environ.get('FLASK_ENV') == 'production':
        app.config.from_object(ProdConfig)
    else:
        app.config.from_object(DevConfig)

    # Inizializza Flask-Login
    login_manager = LoginManager(app)
    login_manager.login_view = 'auth.login'

    @login_manager.user_loader
    def carica_utente(id):
        from auth.utils import trova_utente_per_id
        return trova_utente_per_id(id)

    # Registra i Blueprint
    from auth.routes import auth
    from galleria.routes import galleria

    app.register_blueprint(auth, url_prefix='/auth')
    app.register_blueprint(galleria, url_prefix='/')

    return app
```

---

### Struttura finale completa

```
progetto/
├── app.py
├── factory.py
├── config.py
├── utenti.json
├── uploads.json
├── uploads/
├── auth/
│   ├── __init__.py
│   ├── routes.py
│   └── utils.py        ← funzioni di utilità (leggi_utenti, trova_utente...)
├── galleria/
│   ├── __init__.py
│   ├── routes.py
│   └── utils.py        ← funzioni di utilità (leggi_registro, salva_registro...)
└── templates/
    ├── base.html
    ├── auth/
    │   ├── login.html
    │   └── registrazione.html
    └── galleria/
        └── galleria.html
```

```python
# app.py
from factory import create_app

app = create_app()

if __name__ == '__main__':
    app.run()
```

`app.py` è ridotto a tre righe. Tutta la logica di configurazione e registrazione è nella factory.

---

### Esercizi

**Esercizio 711**

Riorganizza il progetto completo (autenticazione + galleria) nella struttura a pacchetti descritta sopra. I moduli `auth` e `galleria` devono essere Blueprint separati, ognuno con le proprie route, utility e template.

--

**Esercizio 712**

Aggiungi un terzo Blueprint `api` con un prefisso `/api`. Per ora basta un endpoint `GET /api/immagini` che restituisca la lista delle immagini caricate in formato JSON. Registralo nella factory insieme agli altri due.

--

**Esercizio 713** *(più impegnativo)*

Aggiungi alla factory la gestione delle cartelle necessarie all'avvio: se la cartella `uploads/` non esiste, deve essere creata automaticamente. Fai lo stesso per i file JSON (`utenti.json`, `uploads.json`) — se non esistono, inizializzali con una lista vuota `[]`.

> 💡 *Suggerimento:* 
> 
> aggiungi questa logica in fondo a `create_app`, prima del `return app`.

<br>
<br>
<br>
