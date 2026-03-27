# Capitolo 2 — Jinja2 e template

## Obiettivi
Al termine di questo capitolo sarai in grado di:
- creare template HTML con Jinja2
- passare dati da Flask al template
- usare variabili, condizioni e cicli nei template
- strutturare le pagine con l'ereditarietà dei template

---

## Parte 1 — I template

### 1.1 Perché i template?

Finora le nostre funzioni restituivano semplici stringhe di testo. In una vera applicazione web vogliamo restituire pagine HTML complete. Potremmo costruire l'HTML direttamente in Python:

```python
@app.route('/')
def home():
    return '<html><body><h1>Ciao!</h1></body></html>'
```

Funziona, ma diventa presto ingestibile. I **template** risolvono questo problema: sono file HTML separati, con la possibilità di inserire dati dinamici al loro interno.

Flask usa il motore di template **Jinja2**, che è già incluso nell'installazione di Flask.

---

### 1.2 Struttura delle cartelle

Flask si aspetta che i template siano nella cartella `templates`, nella stessa directory di `app.py`:

```
corso_flask/
├── app.py
└── templates/
    └── index.html
```

Crea la cartella `templates` e al suo interno il file `index.html`:

```html
<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <title>La mia prima pagina</title>
</head>
<body>
    <h1>Ciao, Flask!</h1>
</body>
</html>
```

Per restituire questo template da Flask si usa la funzione `render_template`:

```python
from flask import Flask, render_template

app = Flask(__name__)

@app.route('/')
def home():
    return render_template('index.html')
```

Flask troverà automaticamente il file nella cartella `templates`.

---

### 1.3 Passare dati al template

Il vantaggio dei template è che possiamo passare dati da Python all'HTML. Si usa il secondo argomento di `render_template`:

```python
@app.route('/')
def home():
    return render_template('index.html', nome='Marco', eta=17)
```

Nel template, le variabili si usano con la sintassi `{{ variabile }}`:

```html
<body>
    <h1>Ciao, {{ nome }}!</h1>
    <p>Hai {{ eta }} anni.</p>
</body>
```

Possiamo passare anche strutture dati più complesse, come liste e dizionari:

```python
@app.route('/')
def home():
    studenti = ['Alice', 'Bruno', 'Carla']
    return render_template('index.html', studenti=studenti)
```

---

### 1.4 Variabili e filtri

Jinja2 permette di applicare **filtri** alle variabili, trasformandone il valore prima di visualizzarlo:

```html
<p>{{ nome | upper }}</p>       <!-- MARCO -->
<p>{{ testo | truncate(20) }}</p> <!-- Tronca a 20 caratteri -->
<p>{{ lista | length }}</p>     <!-- Numero di elementi -->
```

I filtri più utili:

| Filtro | Effetto |
|---|---|
| `upper` | Converte in maiuscolo |
| `lower` | Converte in minuscolo |
| `capitalize` | Prima lettera maiuscola |
| `length` | Lunghezza di una stringa o lista |
| `default('valore')` | Valore di default se la variabile è vuota |

---

### Esercizi — Parte 1

**Esercizio 1.1**
Crea un template `profilo.html` e una route `/profilo` che passi al template il tuo nome, la tua città e un numero a piacere. Mostra queste informazioni in una pagina HTML ben formattata.

**Esercizio 1.2**
Modifica l'esercizio precedente applicando il filtro `upper` al nome e `capitalize` alla città.

---

## Parte 2 — Logica nei template

### 2.1 Condizioni

Jinja2 permette di inserire istruzioni condizionali nel template con la sintassi `{% %}`:

```html
{% if eta >= 18 %}
    <p>Sei maggiorenne.</p>
{% else %}
    <p>Sei minorenne.</p>
{% endif %}
```

Nota i due tipi di delimitatori:
- `{{ }}` — per **visualizzare** il valore di una variabile
- `{% %}` — per **istruzioni** come if, for, extends...

---

### 2.2 Cicli

Per scorrere una lista si usa `for`:

```html
<ul>
{% for studente in studenti %}
    <li>{{ studente }}</li>
{% endfor %}
</ul>
```

Jinja2 mette a disposizione alcune variabili utili all'interno del ciclo:

```html
{% for studente in studenti %}
    <p>{{ loop.index }} - {{ studente }}</p>
{% endfor %}
```

| Variabile | Valore |
|---|---|
| `loop.index` | Indice corrente (parte da 1) |
| `loop.index0` | Indice corrente (parte da 0) |
| `loop.first` | `True` se è il primo elemento |
| `loop.last` | `True` se è l'ultimo elemento |

---

### 2.3 Iterare su dizionari

Possiamo passare anche dizionari e iterare sulle loro chiavi e valori:

```python
@app.route('/prodotto')
def prodotto():
    info = {
        'nome': 'Zaino',
        'prezzo': 29.90,
        'disponibile': True
    }
    return render_template('prodotto.html', info=info)
```

```html
<dl>
{% for chiave, valore in info.items() %}
    <dt>{{ chiave }}</dt>
    <dd>{{ valore }}</dd>
{% endfor %}
</dl>
```

Oppure accedendo direttamente ai campi:

```html
<h2>{{ info.nome }}</h2>
<p>Prezzo: {{ info.prezzo }} €</p>
{% if info.disponibile %}
    <p>Disponibile</p>
{% else %}
    <p>Non disponibile</p>
{% endif %}
```

---

### Esercizi — Parte 2

**Esercizio 2.1**
Crea una route `/classe` che passi al template una lista di almeno 5 nomi di studenti. Il template deve mostrarli in una lista numerata usando `loop.index`.

**Esercizio 2.2**
Modifica l'esercizio precedente: evidenzia in grassetto il primo e l'ultimo elemento della lista usando `loop.first` e `loop.last`.

> 💡 *Suggerimento:* usa `{% if loop.first or loop.last %}` per applicare un tag `<strong>`.

**Esercizio 2.3**
Crea una route `/voti` che passi al template un dizionario con almeno 4 materie e il relativo voto (es. `{'Matematica': 8, 'Italiano': 7, ...}`). Il template deve mostrare le materie e i voti in una tabella HTML, colorando in rosso i voti inferiori a 6.

> 💡 *Suggerimento:* per il colore puoi usare uno stile inline: `style="color: red"` dentro un `{% if %}`.

---

## Parte 3 — Ereditarietà dei template

### 3.1 Il problema

Immagina di avere dieci pagine, ognuna con la stessa navbar e lo stesso footer. Se vuoi modificare la navbar, devi aprire dieci file. L'**ereditarietà dei template** risolve questo problema.

### 3.2 Il template base

Si crea un file `base.html` che contiene la struttura comune a tutte le pagine. Le parti che cambieranno da pagina a pagina vengono definite come **blocchi**:

```html
<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <title>{% block titolo %}Il mio sito{% endblock %}</title>
</head>
<body>

    <nav>
        <a href="/">Home</a> |
        <a href="/about">Chi siamo</a>
    </nav>

    <main>
        {% block contenuto %}{% endblock %}
    </main>

    <footer>
        <p>© 2025 Corso Flask</p>
    </footer>

</body>
</html>
```

Il blocco `{% block nome %}{% endblock %}` è un segnaposto: ogni pagina figlia potrà riempirlo con il proprio contenuto.

---

### 3.3 Le pagine figlie

Una pagina figlia **estende** il template base e sovrascrive solo i blocchi che le interessano:

```html
{% extends 'base.html' %}

{% block titolo %}Home{% endblock %}

{% block contenuto %}
    <h1>Benvenuto!</h1>
    <p>Questa è la pagina principale.</p>
{% endblock %}
```

Tutto il resto — navbar, footer, struttura HTML — viene ereditato automaticamente da `base.html`.

Un altro esempio per la pagina "Chi siamo":

```html
{% extends 'base.html' %}

{% block titolo %}Chi siamo{% endblock %}

{% block contenuto %}
    <h1>Chi siamo</h1>
    <p>Questo è un corso di Flask per il liceo.</p>
{% endblock %}
```

---

### 3.4 Struttura finale delle cartelle

```
corso_flask/
├── app.py
└── templates/
    ├── base.html
    ├── index.html
    ├── about.html
    └── ...
```

---

### Esercizi — Parte 3

**Esercizio 3.1**
Crea un template `base.html` con una navbar che contenga i link a `/` e `/about`. Crea poi due pagine figlie `index.html` e `about.html` che estendano il base e abbiano contenuti diversi. Collega tutto in `app.py` con le relative route.

**Esercizio 3.2**
Aggiungi al template base un terzo blocco chiamato `{% block sottotitolo %}` posizionato sotto il titolo principale. Nella pagina `index.html` inserisci un sottotitolo, nella pagina `about.html` lascialo vuoto (non sovrascrivere il blocco) e verifica che non compaia nulla.

---

## Riepilogo

| Concetto | Descrizione |
|---|---|
| `render_template('file.html')` | Restituisce un template HTML |
| `{{ variabile }}` | Visualizza il valore di una variabile |
| `{% if %} ... {% endif %}` | Condizione nel template |
| `{% for %} ... {% endfor %}` | Ciclo nel template |
| `filtro` | Trasforma una variabile (es. `upper`, `length`) |
| `{% extends 'base.html' %}` | Eredita la struttura dal template base |
| `{% block nome %} ... {% endblock %}` | Definisce una sezione sostituibile |
