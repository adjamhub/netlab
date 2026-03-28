# Form e flash messages


Un form HTML permette all'utente di inserire dati e inviarli al server. La struttura di base è:

```html
<form action="/destinazione" method="POST">
    <input type="text" name="nome">
    <button type="submit">Invia</button>
</form>
```

Gli attributi fondamentali del tag `<form>`:
- `action` — l'URL a cui inviare i dati
- `method` — il metodo HTTP da usare (`GET` o `POST`)

L'attributo `name` di ogni campo è la chiave con cui Flask leggerà il valore inviato.

---

### 1.2 Un form completo

Creiamo un esempio concreto: un form di contatto. Prima il template `contatto.html`:

```html
{% extends 'base.html' %}

{% block titolo %}Contatti{% endblock %}

{% block contenuto %}
<h1>Scrivici</h1>

<form action="/contatto" method="POST">
    <label for="nome">Nome:</label><br>
    <input type="text" id="nome" name="nome"><br><br>

    <label for="messaggio">Messaggio:</label><br>
    <textarea id="messaggio" name="messaggio" rows="4"></textarea><br><br>

    <button type="submit">Invia</button>
</form>
{% endblock %}
```

Poi la route in `app.py`:

```python
from flask import Flask, render_template, request

app = Flask(__name__)

@app.route('/contatto', methods=['GET', 'POST'])
def contatto():
    if request.method == 'POST':
        nome = request.form.get('nome')
        messaggio = request.form.get('messaggio')
        return f'Grazie {nome}, abbiamo ricevuto il tuo messaggio!'
    return render_template('contatto.html')
```

Quando l'utente apre `/contatto` con GET, vede il form. Quando lo compila e clicca "Invia", 
il browser fa una richiesta POST alla stessa URL e Flask elabora i dati.

---

### Validazione lato server

Non possiamo fidarci ciecamente dei dati inviati dall'utente. È importante **validare** che i campi siano compilati e che i valori abbiano senso.

```python
@app.route('/contatto', methods=['GET', 'POST'])
def contatto():
    if request.method == 'POST':
        nome = request.form.get('nome', '').strip()
        messaggio = request.form.get('messaggio', '').strip()

        if not nome:
            return 'Errore: il campo nome è obbligatorio', 400
        if not messaggio:
            return 'Errore: il campo messaggio è obbligatorio', 400

        return f'Grazie {nome}, abbiamo ricevuto il tuo messaggio!'

    return render_template('contatto.html')
```

Il metodo `.strip()` rimuove gli spazi iniziali e finali — utile per evitare che un campo con solo spazi venga considerato compilato.

Restituire un semplice testo di errore funziona, ma non è elegante. Nel prossimo paragrafo vedremo un modo migliore per gestire il feedback.

---

### Esercizi

**Esercizio 301**

Crea un form con i campi `nome`, `cognome` ed `email`. Quando l'utente invia il form, mostra una pagina di conferma con i dati inseriti.

--

**Esercizio 302**

Aggiungi la validazione all'esercizio precedente: tutti i campi sono obbligatori. Se manca qualcosa, restituisci un messaggio di errore con codice `400`.

--

**Esercizio 303**

Aggiungi un campo `età` di tipo numerico. Verifica che il valore inserito sia un intero compreso tra 10 e 99. In caso contrario, restituisci un errore appropriato.

> 💡 *Suggerimento:* 
> 
> usa `int()` per convertire il valore, e gestisci l'eccezione `ValueError` nel caso in cui l'utente inserisca un testo non numerico.

---


## Flash messages


La gestione degli errori con `return 'Errore...', 400` è rozza: l'utente vede una pagina bianca con il testo dell'errore e deve tornare indietro con il browser. 
Vorremmo invece:

- mostrare un messaggio di feedback **sulla stessa pagina**
- che il messaggio scompaia dopo essere stato letto

I **flash messages** di Flask risolvono esattamente questo.

---

### Come funzionano i flash messages

I flash messages si basano sulla **sessione** di Flask: un messaggio viene salvato temporaneamente, mostrato una volta, e poi eliminato automaticamente.

Per usarli servono due cose:

1. Una **secret key** nell'applicazione (necessaria per le sessioni)
2. Importare `flash` e `redirect`

```python
from flask import Flask, render_template, request, flash, redirect

app = Flask(__name__)
app.secret_key = 'chiave_super_segretissima'
```

> ⚠️ **ATTENZIONE!**
> 
> La `secret_key` serve a Flask per firmare i dati di sessione. In produzione deve essere una stringa lunga e casuale, mai hardcoded nel codice.

---

### Inviare un flash message

Si usa la funzione `flash()` seguita da un `redirect`:

```python
@app.route('/contatto', methods=['GET', 'POST'])
def contatto():
    if request.method == 'POST':
        nome = request.form.get('nome', '').strip()
        messaggio = request.form.get('messaggio', '').strip()

        if not nome or not messaggio:
            flash('Tutti i campi sono obbligatori.', 'errore')
            return redirect( '/contatto' )

        flash(f'Grazie {nome}, messaggio ricevuto!', 'successo')
        return redirect( '/contatto' )

    return render_template('contatto.html')
```

`flash(testo, categoria)` accetta due argomenti:
- il **testo** del messaggio
- la **categoria** (una stringa libera che usiamo per lo stile: `'successo'`, `'errore'`, `'info'`...)

`redirect( '/contatto' )` reindirizza l'utente alla route `/contatto` con una nuova richiesta GET — in questo modo, 
se l'utente ricarica la pagina, il form non viene reinviato.

---

### Mostrare i flash messages nel template

I messaggi vanno mostrati nel template con `get_flashed_messages()`. Il posto migliore per definirli è nel `base.html`, così sono disponibili in tutte le pagine:

```html
{% with messaggi = get_flashed_messages(with_categories=true) %}
    {% if messaggi %}
        {% for categoria, testo in messaggi %}
            <div class="messaggio {{ categoria }}">
                {{ testo }}
            </div>
        {% endfor %}
    {% endif %}
{% endwith %}
```

E un po' di CSS per distinguere i tipi:

```html
<style>
    .messaggio {
        padding: 10px;
        margin: 10px;
        border-radius: 5px;
    }
    .successo { background-color: #d4edda; color: #155724; }
    .errore   { background-color: #f8d7da; color: #721c24; }
    .info     { background-color: #d1ecf1; color: #0c5460; }
</style>
```

---

### Il pattern POST → redirect → GET

Nell'esempio sopra abbiamo seguito un pattern importante: dopo ogni POST, Flask esegue sempre un `redirect`. Questo si chiama **PRG pattern** (Post/Redirect/Get) e risolve un problema classico: se l'utente ricarica la pagina dopo aver inviato un form, il browser non reinvia i dati.

```
1. Utente compila il form  →  richiesta POST
2. Flask elabora i dati    →  redirect (302)
3. Browser segue il redirect  →  richiesta GET
4. Flask mostra la pagina con il flash message
```

---

### Esercizi

**Esercizio 321**

Riprendi il form dell'Esercizio 1.1 e sostituisci la gestione degli errori con i flash messages. Usa la categoria `'errore'` per i campi mancanti e `'successo'` per la conferma.

--

**Esercizio 322**

Crea un form con un solo campo `numero`. L'utente inserisce un numero intero e Flask risponde con un flash message che dice se il numero è pari o dispari. Se il valore inserito non è un numero intero, mostra un messaggio di errore.

--

**Esercizio 323** *(più impegnativo)*

Crea una piccola app con un form che permetta di inserire un nome in una lista. I nomi vengono salvati in un file di testo `nomi.txt`, uno per riga. La pagina mostra sempre la lista aggiornata e un flash message di conferma ogni volta che si aggiunge un nome. Gestisci il caso in cui il nome sia già presente nella lista.

<br>
<br>
<br>
