# Deploy

In questo capitolo cercheremo di:

- capire perché il server di sviluppo Flask non va usato in produzione
- descrivere cos'è uno Web Server Gateway WSGI
- descrivere cos'è un reverse proxy

---

## Il server di sviluppo non basta


Quando avviamo la nostra app con:

```bash
python app.py
```

Flask avvia il suo **server di sviluppo integrato**. È comodo durante lo sviluppo — si riavvia automaticamente ad ogni modifica, mostra errori dettagliati nel browser — ma ha limiti importanti che lo rendono inadatto alla produzione:

- gestisce **una sola richiesta alla volta** — se due utenti aprono la pagina contemporaneamente, il secondo deve aspettare che il primo abbia finito
- la modalità `debug=True` espone informazioni sensibili sul server a chiunque
- non è progettato per essere stabile sotto carico continuativo
- Flask stesso avvisa esplicitamente: *"Do not use the development server in a production environment"*

---

### La catena di produzione

In un ambiente di produzione l'applicazione Flask non viene esposta direttamente a Internet. Davanti ad essa ci sono due componenti aggiuntivi:

```
Internet → Nginx → Gunicorn → Flask
```

Ognuno ha un ruolo preciso:

- **Reverse Proxy** — riceve le connessioni da Internet, gestisce HTTPS, serve i file statici, e inoltra le richieste dinamiche al web server

- **Web Server Gateway** — esegue l'applicazione Flask con più processi paralleli, uno per ogni richiesta simultanea
- 
- **Flask** — elabora la richiesta e produce la risposta, esattamente come in sviluppo

Dal punto di vista di Flask non cambia nulla — riceve richieste HTTP e produce risposte HTTP, senza sapere da dove arrivano.

---

## WSGI

Python ha prodotto nel tempo molti framework web: Flask, Django, FastAPI, Bottle e altri. Allo stesso modo esistono molti server web capaci di eseguire applicazioni Python. Senza uno standard comune, ogni framework avrebbe dovuto essere compatibile con ogni server — un problema di integrazione enorme.

**WSGI** (Web Server Gateway Interface) risolve questo problema: è uno standard Python che definisce un'interfaccia comune tra server web e applicazioni web. Qualsiasi server WSGI può eseguire qualsiasi applicazione WSGI, indipendentemente dal framework usato.

Flask è un'applicazione WSGI. Gunicorn è un server WSGI. Parlano la stessa lingua.

---

### 2.2 Server WSGI disponibili

Un server WSGI esegue l'applicazione Flask gestendo più richieste in parallelo attraverso un sistema di **worker** — processi indipendenti, ognuno dei quali può elaborare una richiesta simultaneamente.

```
Server WSGI
├── worker 1  ←  elabora la richiesta dell'utente A
├── worker 2  ←  elabora la richiesta dell'utente B
├── worker 3  ←  elabora la richiesta dell'utente C
└── worker 4  ←  in attesa
```

Con quattro worker, quattro utenti possono essere serviti contemporaneamente. Il server di sviluppo Flask ne gestisce solo uno alla volta.

I server WSGI più diffusi in ambito Python sono:

| Server       | Note                                              |
|--------------|---------------------------------------------------|
| **Gunicorn** | Semplice, affidabile, molto usato su Linux        |
| **uWSGI**    | Molto configurabile, diffuso su hosting condiviso |
| **Waitress** | Scritto in Python puro, funziona anche su Windows |
| **mod_wsgi** | Modulo per Apache, comune in ambienti enterprise  |

Una regola pratica comune per il numero di worker è `2 × numero_di_CPU + 1`.

---

## Reverse proxy

Un **proxy** è un intermediario: riceve richieste da un client e le inoltra a un altro server per conto suo. Un proxy **diretto** (forward proxy) viene usato dai client per accedere a risorse esterne — ad esempio un proxy aziendale che filtra il traffico in uscita.

Un **reverse proxy** funziona al contrario: si posiziona davanti ai server e riceve richieste da Internet, inoltrandole al server appropriato. Dal punto di vista del client, il reverse proxy *è* il server — non sa che dietro c'è un'altra macchina.

```
Client                Reverse proxy         Server interno
──────                ─────────────         ──────────────
browser  →  richiesta → Nginx       → richiesta → Gunicorn+Flask
browser  ←  risposta  ← Nginx       ← risposta  ← Gunicorn+Flask
```

---

### Reverse proxy disponibili

Usare un reverse proxy davanti al server WSGI aggiunge funzionalità essenziali che quest'ultimo non è progettato per gestire:

**Gestione delle connessioni** — un reverse proxy è estremamente efficiente nel gestire migliaia di connessioni simultanee, anche lente. 
Un client lento non occupa un worker per tutta la durata della connessione.

**File statici** — il reverse proxy serve file CSS, JavaScript e immagini direttamente dal disco, senza coinvolgere Flask. 
È molto più veloce e scarica lavoro dall'applicazione.

**HTTPS** — il reverse proxy gestisce i certificati SSL/TLS e cifra le connessioni. 
Flask riceve sempre richieste HTTP semplici, senza occuparsi della crittografia.

**Sicurezza** — può limitare la dimensione delle richieste, bloccare IP sospetti, gestire rate limiting. 
Fa da scudo prima che le richieste raggiungano l'applicazione.

I reverse proxy più diffusi sono:

| Software    | Note                                                                                |
|-------------|-------------------------------------------------------------------------------------|
| **Nginx**   | Leggero, performante, molto usato con Flask e Django                                |
| **Apache**  | Storico e diffusissimo, configurabile con mod_proxy                                 |
| **Caddy**   | Moderno, gestisce HTTPS automaticamente                                             |
| **Traefik** | Pensato per ambienti container e microservizi                                       |
| **HAProxy** | Altissime prestazioni, eccellente per bilanciamento del carico e alta disponibilità |

---

### Il flusso completo

Vediamo cosa succede quando un utente apre la nostra app in produzione:

1. Il browser invia una richiesta HTTPS al server
2. **Nginx** riceve la richiesta, decifra HTTPS e la analizza
3. Se è una richiesta per un file statico (CSS, immagine), Nginx la serve direttamente dal disco
4. Se è una richiesta dinamica (una pagina Flask, un endpoint API), Nginx la inoltra a **Gunicorn**
5. **Gunicorn** assegna la richiesta a un worker libero
6. Il worker esegue la funzione Flask corrispondente e produce la risposta
7. La risposta risale la catena: Gunicorn → Nginx → browser

Dal punto di vista del codice Flask, i passi 2, 3, 4 e 5 sono completamente trasparenti.

---

## In produzione


**Cosa cambia nel codice??**

Molto poco. Le differenze principali rispetto allo sviluppo sono:

- `debug=False` — niente errori dettagliati esposti al browser
- La `SECRET_KEY` viene letta da una variabile d'ambiente, non scritta nel codice
- I file di log vengono scritti su disco invece che stampati sul terminale

Il codice dell'applicazione — route, template, logica — rimane identico.

---

### Variabili d'ambiente

In produzione i valori sensibili come la `SECRET_KEY` non vanno nel codice sorgente — potrebbero finire su Git e diventare pubblici. 
Si usano le **variabili d'ambiente**: valori impostati nel sistema operativo del server, fuori dal codice.

```python
import os

class ProdConfig:
    DEBUG = False
    SECRET_KEY = os.environ.get('SECRET_KEY')
```

`os.environ.get('SECRET_KEY')` legge il valore dalla variabile d'ambiente `SECRET_KEY` impostata sul server. 
Se la variabile non esiste, restituisce `None`.


<br>
<br>
<br>
