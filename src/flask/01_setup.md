# Setup, primi passi e routing


Flask è un pacchetto disponibile su PyPi come tanti altri prima d'ora. Se vuoi semplicemente aggiungere il pacchetto `flask` alla tua installazione, da Thonny,
apri il terminale da `Strumenti` ---> `Apri shell di sistema` e digita:

```bash
pip install flask
```

Se invece vuoi creare un progetto con `uv`, allora crea una applicazione semplice e aggiungi flask come dipendenza! Da un normale terminale (non quello di Thonny)
digita:

```bash
uv init progettoFlask
cd progettoFlask
uv add flask
```

Il primo comando crea un progetto (una cartella) chiamata `progettoFlask`, con il secondo ci entri dentro, con il terzo aggiungi la dipendenza flask al progetto.
Da quello stesso terminale, in quella stessa cartella, per eseguire il progetto, ti basterà eseguire:

```bash
uv run
```


### La prima applicazione

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

### Come funziona

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
Il **decorator** `@app.route('/')` dice a Flask: *"quando il browser richiede la pagina `/`, esegui la funzione `home`"*. 
La funzione restituisce il testo che il browser mostrerà.

```python
app.run(debug=True)
```
Avvia il server. Il parametro `debug=True` è utile durante lo sviluppo: riavvia automaticamente il server ad ogni modifica del codice e mostra errori dettagliati nel browser.

> ⚠️ **Attenzione:** 
> 
> `debug=True` non va mai usato in produzione.

---

### Esercizi semplici su Flask

**Esercizio 1**

Modifica `app.py` in modo che la pagina principale mostri il tuo nome e cognome.

-- 

**Esercizio 2**

Aggiungi una seconda funzione che risponda all'indirizzo `/saluto` e restituisca il testo `"Benvenuto nel corso di Flask!"`.

> 💡 *Suggerimento:* 
> 
> Segui lo stesso schema della funzione `home`, cambiando il path nel decorator e il nome della funzione.

--

**Esercizio 3**

Cosa succede se provi ad andare su un indirizzo che non esiste, come `http://127.0.0.1:5000/pippo`? Prova e descrivi quello che vedi.

---

## Routing e views

Fino ad ora abbiamo definito route **statiche** (con URL fissi), come `/` e `/saluto`. 
Flask permette però di definire route **dinamiche**, dove parte dell'URL è una variabile.

```python
@app.route('/utente/<nome>')
def utente(nome):
    return f'Ciao, {nome}!'
```

Se vai su `http://127.0.0.1:5000/utente/Marco`, il browser mostrerà: **Ciao, Marco!**

La parte `<nome>` nell'URL viene catturata e passata come parametro alla funzione.


### Tipi di parametro

Per default il parametro è una stringa, ma puoi specificare un tipo:

```python
@app.route('/prodotto/<int:id>')
def prodotto(id):
    return f'Hai richiesto il prodotto numero {id}'
```

I tipi disponibili sono:

|               Tipo | Esempio URL     | Valore ricevuto |
|--------------------|-----------------|-----------------|
| `string` (default) | `/utente/Marco` | `'Marco'`       |
| `int`              | `/prodotto/42`  | `42`            |
| `float`            | `/prezzo/3.99`  | `3.99`          |

---

### Metodi HTTP: GET e POST

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

### Parametri GET nell'URL

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

### Dati inviati con POST

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

> 📝 **NOTA**
>
> I template HTML per i form li vedremo nel prossimo capitolo. Per ora basta sapere come Flask riceve i dati.

---

### Restituire codici di stato HTTP

Ogni risposta HTTP ha un **codice di stato**. Flask restituisce `200 OK` per default, ma puoi specificarne uno diverso:

```python
@app.route('/non-trovato')
def non_trovato():
    return 'Pagina non trovata', 404
```

I codici più comuni che incontrerai:

| Codice | Significato                               |
|--------|-------------------------------------------|
| `200`  | OK — tutto bene                           |
| `201`  | Created — risorsa creata                  |
| `400`  | Bad Request — richiesta malformata        |
| `404`  | Not Found — risorsa non trovata           |
| `500`  | Internal Server Error — errore del server |

---

### Esercizi 

**Esercizio 11**

Crea una route `/saluta/<nome>` che restituisca `"Ciao, [nome]! Benvenuto nel corso."`.

--

**Esercizio 12**

Crea una route `/somma/<int:a>/<int:b>` che restituisca il risultato della somma dei due numeri.

> 💡 *Suggerimento:* 
>
> Ricorda che puoi usare f-string per costruire la risposta.

--

**Esercizio 13**

Crea una route `/cerca` che legga un parametro `q` dalla query string e restituisca `"Risultati per: [q]"`. Se il parametro non è presente, restituisca `"Nessun termine di ricerca"` con codice di stato `400`.

<br>
<br>
<br>
