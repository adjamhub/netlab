# Capitolo 1 — Setup, primi passi e routing

## Obiettivi
Al termine di questo capitolo sarai in grado di:
- installare Flask in un ambiente virtuale
- creare una prima applicazione funzionante
- definire route con parametri statici e dinamici
- gestire le richieste GET e POST

---

## Parte 1 — Setup e primi passi

### Cos'è Flask

Flask è un **framework web** scritto in Python. Serve a costruire applicazioni web e API in modo semplice e veloce, senza dover gestire da zero tutta la complessità di un server HTTP.

Esistono framework più grandi e completi, come Django, ma Flask è volutamente minimale: ti dà gli strumenti essenziali e ti lascia libero di aggiungere solo quello che ti serve.

È molto usato per:
- applicazioni web di piccole e medie dimensioni
- API REST
- prototipi e progetti didattici

---

### 1.1 Installazione

#### Python e pip
Prima di tutto, assicurati di avere Python 3.8 o superiore installato. Verifica dal terminale:

```bash
python --version
```

#### Virtual environment
È buona pratica lavorare in un **ambiente virtuale**: uno spazio isolato dove installare i pacchetti del progetto senza interferire con il resto del sistema.

Crea una cartella per il progetto ed entra al suo interno:

```bash
mkdir corso_flask
cd corso_flask
```

Crea l'ambiente virtuale:

```bash
python -m venv venv
```

Attivalo:

```bash
# Windows
venv\Scripts\activate

# macOS / Linux
source venv/bin/activate
```

Quando l'ambiente è attivo, vedrai `(venv)` all'inizio del prompt.

#### Installazione di Flask

```bash
pip install flask
```

Verifica che tutto sia andato a buon fine:

```bash
flask --version
```

---

### 1.2 La prima applicazione

Crea un file chiamato `app.py` nella cartella del progetto e scrivi questo codice:

```python
from flask import Flask

# Crea l'applicazione Flask
app = Flask(__name__)

# Definisce la route principale
@app.route('/')
def home():
    return 'Ciao, Flask!'

# Avvia il server
if __name__ == '__main__':
    app.run(debug=True)
```

Avvia l'applicazione dal terminale:

```bash
python app.py
```

Dovresti vedere un output simile a questo:

```
 * Running on http://127.0.0.1:5000
 * Debug mode: on
```

Apri il browser e vai all'indirizzo `http://127.0.0.1:5000`. Vedrai scritto: **Ciao, Flask!**

---

### 1.3 Come funziona

Analizziamo il codice riga per riga:

```python
from flask import Flask
```
Importa la classe `Flask` dalla libreria.

```python
app = Flask(__name__)
```
Crea un'istanza dell'applicazione. `__name__` è una variabile Python che contiene il nome del modulo corrente — Flask la usa per trovare le risorse del progetto.

```python
@app.route('/')
def home():
    return 'Ciao, Flask!'
```
Il **decorator** `@app.route('/')` dice a Flask: *"quando il browser richiede la pagina `/`, esegui la funzione `home`"*. La funzione restituisce il testo che il browser mostrerà.

```python
app.run(debug=True)
```
Avvia il server. Il parametro `debug=True` è utile durante lo sviluppo: riavvia automaticamente il server ad ogni modifica del codice e mostra errori dettagliati nel browser.

> ⚠️ **Attenzione:** `debug=True` non va mai usato in produzione.

---

### Esercizi — Parte 1

**Esercizio 1.1**
Modifica `app.py` in modo che la pagina principale mostri il tuo nome e cognome.

**Esercizio 1.2**
Aggiungi una seconda funzione che risponda all'indirizzo `/saluto` e restituisca il testo `"Benvenuto nel corso di Flask!"`.

> 💡 *Suggerimento:* segui lo stesso schema della funzione `home`, cambiando il path nel decorator e il nome della funzione.

**Esercizio 1.3**
Cosa succede se provi ad andare su un indirizzo che non esiste, come `http://127.0.0.1:5000/pippo`? Prova e descrivi quello che vedi.

---

## Parte 2 — Routing e views

### 2.1 Route statiche e dinamiche

Fino ad ora abbiamo definito route con URL fissi, come `/` e `/saluto`. Flask permette però di definire **route dinamiche**, dove parte dell'URL è una variabile.

```python
@app.route('/utente/<nome>')
def utente(nome):
    return f'Ciao, {nome}!'
```

Se vai su `http://127.0.0.1:5000/utente/Marco`, il browser mostrerà: **Ciao, Marco!**

La parte `<nome>` nell'URL viene catturata e passata come parametro alla funzione.

#### Tipi di parametro

Per default il parametro è una stringa, ma puoi specificare un tipo:

```python
@app.route('/prodotto/<int:id>')
def prodotto(id):
    return f'Hai richiesto il prodotto numero {id}'
```

I tipi disponibili sono:

| Tipo | Esempio URL | Valore ricevuto |
|---|---|---|
| `string` (default) | `/utente/Marco` | `'Marco'` |
| `int` | `/prodotto/42` | `42` |
| `float` | `/prezzo/3.99` | `3.99` |

---

### 2.2 Metodi HTTP: GET e POST

Ogni richiesta HTTP ha un **metodo** che indica l'intenzione del client:
- **GET** — chiede di ricevere dati (es. aprire una pagina)
- **POST** — invia dati al server (es. compilare un form)

Per default, Flask risponde solo alle richieste GET. Per accettare anche il POST, bisogna specificarlo:

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/dati', methods=['GET', 'POST'])
def dati():
    if request.method == 'POST':
        return 'Hai inviato un POST'
    return 'Hai fatto una richiesta GET'
```

L'oggetto `request` contiene tutte le informazioni sulla richiesta in arrivo: il metodo, i dati del form, i parametri URL, e altro ancora.

---

### 2.3 Parametri GET nell'URL

Oltre ai parametri dinamici nel path, puoi leggere i **parametri query string**, quelli che appaiono dopo il `?` nell'URL:

```
http://127.0.0.1:5000/cerca?q=flask
```

```python
@app.route('/cerca')
def cerca():
    termine = request.args.get('q', '')
    return f'Stai cercando: {termine}'
```

`request.args.get('q', '')` legge il parametro `q` dall'URL. Il secondo argomento (`''`) è il valore di default se il parametro non è presente.

---

### 2.4 Dati inviati con POST

Quando un form HTML invia dati con il metodo POST, questi arrivano nel corpo della richiesta e si leggono con `request.form`:

```python
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        utente = request.form.get('utente')
        password = request.form.get('password')
        return f'Accesso come: {utente}'
    return 'Pagina di login (GET)'
```

> 📝 I template HTML per i form li vedremo nel prossimo capitolo. Per ora basta sapere come Flask riceve i dati.

---

### 2.5 Restituire codici di stato HTTP

Ogni risposta HTTP ha un **codice di stato**. Flask restituisce `200 OK` per default, ma puoi specificarne uno diverso:

```python
@app.route('/non-trovato')
def non_trovato():
    return 'Pagina non trovata', 404
```

I codici più comuni che incontrerai:

| Codice | Significato |
|---|---|
| `200` | OK — tutto bene |
| `201` | Created — risorsa creata |
| `400` | Bad Request — richiesta malformata |
| `404` | Not Found — risorsa non trovata |
| `500` | Internal Server Error — errore del server |

---

### Esercizi — Parte 2

**Esercizio 2.1**
Crea una route `/saluta/<nome>` che restituisca `"Ciao, [nome]! Benvenuto nel corso."`.

**Esercizio 2.2**
Crea una route `/somma/<int:a>/<int:b>` che restituisca il risultato della somma dei due numeri.

> 💡 *Suggerimento:* ricorda che puoi usare f-string per costruire la risposta.

**Esercizio 2.3**
Crea una route `/cerca` che legga un parametro `q` dalla query string e restituisca `"Risultati per: [q]"`. Se il parametro non è presente, restituisca `"Nessun termine di ricerca"` con codice di stato `400`.

---

## Riepilogo

| Concetto | Descrizione |
|---|---|
| Virtual environment | Ambiente isolato per i pacchetti del progetto |
| `@app.route('/path')` | Associa un URL a una funzione |
| `<variabile>` nel path | Parametro dinamico nell'URL |
| `request.method` | Metodo HTTP della richiesta (GET, POST...) |
| `request.args.get()` | Legge parametri dalla query string |
| `request.form.get()` | Legge dati inviati con POST |
| Codici di stato | Numero che indica l'esito della risposta |
