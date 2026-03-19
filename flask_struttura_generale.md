# Guida Completa ai File di un'App Web Flask

---

## `.env` — Variabili d'Ambiente

È il file dove tieni tutte le **informazioni sensibili e configurabili** fuori dal codice. Non viene mai caricato su GitHub.

```
SECRET_KEY=una_stringa_lunga_e_casuale_impossibile_da_indovinare
DATABASE_URL=sqlite:///mia_app.db
DEBUG=True
```

**Perché esiste:**
- La `SECRET_KEY` serve a Flask per cifrare i cookie di sessione. Se qualcuno la conosce, può falsificare sessioni utente.
- `DATABASE_URL` indica dove si trova il database — cambia tra sviluppo e produzione senza toccare il codice.
- `DEBUG=True` mostra gli errori dettagliati in sviluppo. Va sempre messo a `False` in produzione.

**Come si legge nel codice:**
```python
from dotenv import load_dotenv
import os

load_dotenv()  # legge il file .env e carica tutto in os.environ

secret = os.getenv('SECRET_KEY')
```

---

## `.gitignore` — File Ignorati da Git

Dice a Git quali file **non** caricare mai su GitHub.

```
.venv/          ← l'ambiente virtuale (enorme, inutile da condividere)
.env            ← contiene segreti
__pycache__/    ← file compilati di Python
*.pyc           ← bytecode Python
*.db            ← il database (dati locali)
instance/       ← cartella con db e config locali
```

---

## `requirements.txt` — Dipendenze del Progetto

Elenco di tutti i pacchetti necessari. Si crea con:

```bash
pip freeze > requirements.txt
```

Si installa tutto con:

```bash
pip install -r requirements.txt
```

```
flask==3.0.0
flask-sqlalchemy==3.1.1
flask-login==0.6.3
flask-wtf==1.2.1
python-dotenv==1.0.0
werkzeug==3.0.1
```

Serve a chiunque voglia far girare il tuo progetto: clona il repo, esegue il comando e ha tutto installato.

---

## `run.py` — Punto di Avvio

È il file che **avvii per far partire il server**. Contiene il minimo indispensabile.

```python
from app import create_app

app = create_app()  # chiama la factory function in app/__init__.py

if __name__ == '__main__':
    app.run()  # avvia il server di sviluppo su http://127.0.0.1:5000
```

**Perché è separato da `__init__.py`:**
Tenere il punto di avvio fuori dalla cartella `app/` permette di importare `app` come modulo senza avviare subito il server. Utile per i test e per strumenti come Gunicorn in produzione.

---

## `setup_db.py` — Creazione del Database (solo con SQLite diretto)

Usato nei progetti che **non usano SQLAlchemy** ma scrivono SQL puro. Va eseguito **una sola volta** prima di avviare l'app.

```python
import sqlite3
import os

# Crea la cartella instance/ se non esiste
if not os.path.exists('instance'):
    os.makedirs('instance')

db_path = os.path.join('instance', 'app.sqlite')
connection = sqlite3.connect(db_path)

# Legge ed esegue lo schema SQL
with open('app/schema.sql') as f:
    connection.executescript(f.read())

print("Database creato:", db_path)
connection.close()
```

```bash
python setup_db.py   # esegui solo la prima volta
```

> Con SQLAlchemy questo file non serve: le tabelle vengono create automaticamente da `db.create_all()` dentro `__init__.py`.

---

## `app/schema.sql` — Schema del Database (solo con SQLite diretto)

File SQL che definisce la **struttura delle tabelle**. Viene letto da `setup_db.py`.

```sql
DROP TABLE IF EXISTS user;
DROP TABLE IF EXISTS post;

CREATE TABLE user (
  id       INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT UNIQUE NOT NULL,
  password TEXT NOT NULL
);

CREATE TABLE post (
  id        INTEGER PRIMARY KEY AUTOINCREMENT,
  author_id INTEGER NOT NULL,
  created   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  title     TEXT NOT NULL,
  body      TEXT NOT NULL,
  FOREIGN KEY (author_id) REFERENCES user (id)
);

-- Dati di esempio per testare subito l'app
INSERT INTO user (username, password) VALUES ('admin', 'password_hash_qui');
```

**Concetti chiave:**
- `PRIMARY KEY AUTOINCREMENT` — l'id viene assegnato automaticamente
- `UNIQUE NOT NULL` — il campo è obbligatorio e non può ripetersi
- `FOREIGN KEY` — collega due tabelle (ogni post appartiene a un utente)
- `DEFAULT CURRENT_TIMESTAMP` — la data viene salvata automaticamente all'inserimento

---

## `app/__init__.py` — Factory dell'App

È il **cuore dell'applicazione**. Crea e configura Flask, inizializza tutte le estensioni e registra i blueprint.

Si chiama *factory* perché è una funzione che *produce* l'app — questo permette di creare istanze diverse (es. una per i test, una per la produzione).

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager
from dotenv import load_dotenv
import os

load_dotenv()

# Istanze globali delle estensioni (non ancora legate a nessuna app)
db = SQLAlchemy()
login_manager = LoginManager()

def create_app():
    app = Flask(__name__)

    # 1. Configurazione (letta da .env)
    app.config['SECRET_KEY'] = os.getenv('SECRET_KEY')
    app.config['SQLALCHEMY_DATABASE_URI'] = os.getenv('DATABASE_URL')
    app.config['DEBUG'] = os.getenv('DEBUG', 'False') == 'True'

    # 2. Inizializzazione estensioni (ora vengono legate all'app)
    db.init_app(app)
    login_manager.init_app(app)
    login_manager.login_view = 'auth.login'  # redirect se non loggato

    # 3. Registrazione blueprint (i moduli con le route)
    from .routes import main
    app.register_blueprint(main)

    from .auth import auth
    app.register_blueprint(auth)

    # 4. Creazione tabelle (solo con SQLAlchemy)
    with app.app_context():
        db.create_all()

    return app
```

---

## `app/db.py` — Connessione al Database (solo con SQLite diretto)

Gestisce la connessione a SQLite **senza ORM**. Viene usato al posto di SQLAlchemy.

```python
import sqlite3
from flask import current_app, g

def get_db():
    """
    Apre la connessione al DB se non esiste ancora per questa richiesta.
    'g' è un oggetto Flask che vive solo per la durata di una singola richiesta HTTP.
    """
    if 'db' not in g:
        g.db = sqlite3.connect(current_app.config['DATABASE'])
        g.db.row_factory = sqlite3.Row  # permette post['title'] invece di post[0]
    return g.db

def close_db(e=None):
    """Chiude la connessione automaticamente alla fine di ogni richiesta."""
    db = g.pop('db', None)
    if db is not None:
        db.close()

def init_app(app):
    """Registra close_db come funzione da chiamare all'uscita del contesto."""
    app.teardown_appcontext(close_db)
```

**`g` vs `session`:**
- `g` — vive solo per la durata di una richiesta HTTP (sparisce dopo)
- `session` — vive tra una richiesta e l'altra (salvata nel cookie del browser)

---

## `app/models.py` — Modelli del Database (solo con SQLAlchemy)

Definisce le **tabelle del database come classi Python**. SQLAlchemy traduce le classi in SQL automaticamente.

```python
from . import db, login_manager
from flask_login import UserMixin
from datetime import datetime
from werkzeug.security import generate_password_hash, check_password_hash

# Questa funzione dice a Flask-Login come caricare un utente dall'id in sessione
@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))


class User(UserMixin, db.Model):  # UserMixin aggiunge is_authenticated, is_active, ecc.
    __tablename__ = 'users'

    id       = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email    = db.Column(db.String(120), unique=True, nullable=False)
    password = db.Column(db.String(200), nullable=False)
    created  = db.Column(db.DateTime, default=datetime.utcnow)

    # Relazione: un utente ha molti post
    # backref='autore' aggiunge post.autore come scorciatoia inversa
    posts = db.relationship('Post', backref='autore', lazy=True)

    def set_password(self, plaintext):
        self.password = generate_password_hash(plaintext)

    def check_password(self, plaintext):
        return check_password_hash(self.password, plaintext)


class Post(db.Model):
    __tablename__ = 'posts'

    id      = db.Column(db.Integer, primary_key=True)
    titolo  = db.Column(db.String(200), nullable=False)
    corpo   = db.Column(db.Text, nullable=False)
    created = db.Column(db.DateTime, default=datetime.utcnow)

    # Chiave esterna: collega ogni post a un utente
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
```

**Differenza tra SQLite diretto e SQLAlchemy:**

| | SQLite diretto | SQLAlchemy |
|---|---|---|
| Tabelle definite in | `schema.sql` | `models.py` |
| Query scritte in | SQL puro (`SELECT * FROM ...`) | Python (`Post.query.all()`) |
| Creazione DB | `python setup_db.py` | `db.create_all()` automatico |
| Flessibilità | Massima | Minore ma più rapido da scrivere |

---

## `app/repositories/` — Pattern Repository (solo con SQLite diretto)

Cartella che contiene un file per ogni tabella. **Separano il codice SQL dalle route** — le route non toccano mai il database direttamente.

```
repositories/
├── __init__.py          ← vuoto, rende la cartella un modulo Python
├── user_repository.py
├── post_repository.py
└── bookmark_repository.py
```

**Esempio — `post_repository.py`:**
```python
from app.db import get_db
from datetime import datetime

def get_all_posts():
    """Tutti i post con JOIN per avere anche lo username dell'autore."""
    posts = get_db().execute("""
        SELECT p.id, p.title, p.body, p.created, u.username
        FROM post p
        JOIN user u ON p.author_id = u.id
        ORDER BY p.created DESC
    """).fetchall()

    # Converte le righe SQLite in dizionari normali e parsa le date
    result = []
    for post in posts:
        d = dict(post)
        d['created'] = datetime.fromisoformat(d['created'])
        result.append(d)
    return result

def create_post(title, body, author_id):
    db = get_db()
    db.execute("INSERT INTO post (title, body, author_id) VALUES (?, ?, ?)",
               (title, body, author_id))
    db.commit()  # senza commit le modifiche non vengono salvate
```

**Perché i `?` al posto dei valori:**
Usare `?` (parametri preparati) protegge da **SQL Injection** — un attacco dove l'utente inserisce del codice SQL nei campi del form. Mai concatenare stringhe nelle query.

```python
# SBAGLIATO — vulnerabile a SQL Injection
db.execute(f"SELECT * FROM user WHERE username = '{username}'")

# GIUSTO — sicuro
db.execute("SELECT * FROM user WHERE username = ?", (username,))
```

---

## `app/forms.py` — Form con Flask-WTF

Definisce i form come classi Python. Flask-WTF si occupa di **validazione** e **protezione CSRF** automaticamente.

```python
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, TextAreaField, SelectField, SubmitField
from wtforms.validators import DataRequired, Email, Length, EqualTo

class RegisterForm(FlaskForm):
    username = StringField('Username', validators=[
        DataRequired(),          # campo obbligatorio
        Length(min=3, max=80)    # lunghezza tra 3 e 80 caratteri
    ])
    email = StringField('Email', validators=[
        DataRequired(),
        Email()                  # controlla che sia una email valida
    ])
    password = PasswordField('Password', validators=[
        DataRequired(),
        Length(min=6)
    ])
    conferma = PasswordField('Conferma Password', validators=[
        DataRequired(),
        EqualTo('password', message='Le password non coincidono')
    ])
    submit = SubmitField('Registrati')


class PostForm(FlaskForm):
    titolo    = StringField('Titolo', validators=[DataRequired()])
    corpo     = TextAreaField('Testo', validators=[DataRequired()])
    categoria = SelectField('Categoria', choices=[
        ('news', 'Notizie'),
        ('tutorial', 'Tutorial'),
        ('altro', 'Altro')
    ])
    submit = SubmitField('Pubblica')
```

**CSRF (Cross-Site Request Forgery):**
È un attacco dove un sito malevolo fa fare richieste a nome dell'utente. Flask-WTF aggiunge un token segreto nascosto in ogni form (`{{ form.hidden_tag() }}`). Se il token manca o è sbagliato, la richiesta viene rifiutata.

---

## `app/auth.py` — Blueprint Autenticazione

Gestisce tutto ciò che riguarda **identità degli utenti**: registrazione, login, logout.

```python
from flask import Blueprint, render_template, redirect, url_for, flash, request, session, g
from werkzeug.security import generate_password_hash, check_password_hash

auth = Blueprint('auth', __name__, url_prefix='/auth')
# url_prefix='/auth' → tutte le route di questo blueprint iniziano con /auth
# es: /auth/login, /auth/register, /auth/logout


@auth.before_app_request
def load_logged_in_user():
    """
    Viene eseguita PRIMA DI OGNI RICHIESTA dell'intera app.
    Legge l'id utente dalla sessione e carica l'utente dal DB.
    Rende g.user disponibile in tutte le route e in tutti i template.
    """
    user_id = session.get('user_id')
    g.user = User.query.get(user_id) if user_id else None


@auth.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():  # True solo se POST e tutti i validatori passano
        user = User.query.filter_by(email=form.email.data).first()

        if user and user.check_password(form.password.data):
            session.clear()
            session['user_id'] = user.id  # salva l'id nel cookie cifrato
            flash('Login effettuato!', 'success')
            return redirect(url_for('main.index'))

        flash('Credenziali errate.', 'danger')

    return render_template('login.html', form=form)
    # Se GET → mostra il form vuoto
    # Se POST con errori → mostra il form con gli errori


@auth.route('/logout')
def logout():
    session.clear()  # elimina tutti i dati dalla sessione
    return redirect(url_for('main.index'))
```

---

## `app/routes.py` (o `main.py`) — Blueprint Principale

Contiene le **route delle pagine principali** dell'app. Una route è l'associazione tra un URL e una funzione Python.

```python
from flask import Blueprint, render_template, redirect, url_for, flash, request, g
from flask_login import login_required, current_user
from werkzeug.exceptions import abort

main = Blueprint('main', __name__)


@main.route('/')
def index():
    """Pagina principale — visibile a tutti."""
    posts = Post.query.order_by(Post.created.desc()).all()
    return render_template('index.html', posts=posts)


@main.route('/post/nuovo', methods=['GET', 'POST'])
@login_required  # redirect automatico a login se non autenticato
def nuovo_post():
    form = PostForm()
    if form.validate_on_submit():
        post = Post(
            titolo=form.titolo.data,
            corpo=form.corpo.data,
            user_id=current_user.id
        )
        db.session.add(post)
        db.session.commit()
        flash('Post pubblicato!', 'success')
        return redirect(url_for('main.index'))
    return render_template('nuovo_post.html', form=form)


@main.route('/post/<int:post_id>/elimina', methods=['POST'])
@login_required
def elimina_post(post_id):
    """
    <int:post_id> → Flask converte automaticamente la parte dell'URL in intero
    methods=['POST'] → accetta solo richieste POST (mai GET per operazioni distruttive)
    """
    post = Post.query.get_or_404(post_id)  # se non esiste → pagina 404 automatica

    if post.user_id != current_user.id:
        abort(403)  # Forbidden — non sei l'autore

    db.session.delete(post)
    db.session.commit()
    return redirect(url_for('main.index'))
```

**Anatomia di una route:**
```
@main.route('/post/<int:post_id>', methods=['GET', 'POST'])
    │              │                    │
    │              │                    └── metodi HTTP accettati
    │              └── parametro dinamico nell'URL (convertito in int)
    └── decoratore che registra la route nel blueprint
```

---

## `app/templates/` — I Template HTML

I template usano **Jinja2**, il motore di template di Flask. Permettono di inserire logica Python nell'HTML.

---

### `base.html` — Template Base (Ereditarietà)

È lo scheletro comune a tutte le pagine. Definisce la struttura HTML, la navbar e i blocchi che le pagine figlie possono sovrascrivere.

```html
<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}La Mia App{% endblock %}</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>

<nav>
    <a href="{{ url_for('main.index') }}">Home</a>

    {# g.user è disponibile grazie a before_app_request in auth.py #}
    {% if g.user %}
        <span>Ciao, {{ g.user.username }}</span>
        <a href="{{ url_for('auth.logout') }}">Logout</a>
    {% else %}
        <a href="{{ url_for('auth.login') }}">Login</a>
        <a href="{{ url_for('auth.register') }}">Registrati</a>
    {% endif %}
</nav>

{# Messaggi flash: feedback temporanei dopo un'azione #}
{% with messages = get_flashed_messages(with_categories=true) %}
    {% for category, message in messages %}
        <div class="alert alert-{{ category }}">{{ message }}</div>
    {% endfor %}
{% endwith %}

<main>
    {% block content %}{% endblock %}
</main>

<footer>© 2024 La Mia App</footer>

</body>
</html>
```

---

### `index.html` — Pagina Principale

Estende `base.html` e riempie il blocco `content`.

```html
{% extends 'base.html' %}

{% block title %}Home{% endblock %}

{% block content %}
<h1>Ultimi Post</h1>

{% if g.user %}
    <a href="{{ url_for('main.nuovo_post') }}">+ Nuovo Post</a>
{% endif %}

{% for post in posts %}
<article>
    <h2>{{ post.titolo }}</h2>
    <small>di {{ post.autore.username }} — {{ post.created.strftime('%d/%m/%Y') }}</small>
    <p>{{ post.corpo[:200] }}...</p>

    {% if g.user and g.user.id == post.user_id %}
        <a href="{{ url_for('main.modifica_post', post_id=post.id) }}">Modifica</a>

        <form method="POST" action="{{ url_for('main.elimina_post', post_id=post.id) }}">
            {{ form.hidden_tag() }}
            <button type="submit" onclick="return confirm('Sei sicuro?')">Elimina</button>
        </form>
    {% endif %}
</article>
{% else %}
    <p>Nessun post ancora. Sii il primo!</p>
{% endfor %}

{% endblock %}
```

---

### Template per i Form (login, register, ecc.)

```html
{% extends 'base.html' %}
{% block title %}Accedi{% endblock %}

{% block content %}
<div class="form-container">
    <h1>Accedi</h1>

    <form method="POST">
        {{ form.hidden_tag() }}

        <div class="campo">
            {{ form.email.label }}
            {{ form.email(placeholder='tua@email.com') }}
            {% for errore in form.email.errors %}
                <span class="errore">{{ errore }}</span>
            {% endfor %}
        </div>

        <div class="campo">
            {{ form.password.label }}
            {{ form.password() }}
            {% for errore in form.password.errors %}
                <span class="errore">{{ errore }}</span>
            {% endfor %}
        </div>

        {{ form.submit(class='btn') }}
    </form>

    <p>Non hai un account? <a href="{{ url_for('auth.register') }}">Registrati</a></p>
</div>
{% endblock %}
```

---

### Sintassi Jinja2 — Riepilogo

```jinja2
{{ variabile }}                        ← stampa il valore di una variabile
{% if condizione %} ... {% endif %}    ← blocco condizionale
{% for x in lista %} ... {% endfor %}  ← ciclo
{% extends 'base.html' %}              ← eredita da un template
{% block nome %} ... {% endblock %}    ← definisce/sovrascrive un blocco
{# questo è un commento #}             ← non viene mai mostrato nell'HTML

{{ url_for('blueprint.funzione') }}    ← genera l'URL di una route
{{ testo | upper }}                    ← filtro: testo in maiuscolo
{{ lista | length }}                   ← filtro: conta gli elementi
{{ testo[:100] }}                      ← slicing Python nei template
```

---

## `app/static/` — File Statici

Contiene CSS, JavaScript e immagini. Vengono serviti direttamente dal server senza passare per Flask.

```
static/
├── style.css
├── script.js
└── img/
    └── logo.png
```

**Come si linkano nei template:**
```html
{# MAI scrivere il percorso a mano — usa sempre url_for #}
<link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
<script src="{{ url_for('static', filename='script.js') }}"></script>
<img src="{{ url_for('static', filename='img/logo.png') }}">
```

---

## Flusso Completo di una Richiesta

```
Browser
   │
   │  GET /post/42
   ▼
Flask riceve la richiesta
   │
   ├── before_app_request → carica g.user dalla sessione
   │
   ├── routes.py → trova la route @main.route('/post/<int:post_id>')
   │
   ├── chiama la funzione Python corrispondente
   │       │
   │       ├── (SQLAlchemy)  Post.query.get(42)
   │       │   oppure
   │       └── (Repository)  post_repository.get_post_by_id(42)
   │
   ├── render_template('post.html', post=post)
   │       │
   │       └── Jinja2 compila l'HTML con i dati
   │
   ▼
Browser riceve l'HTML e lo mostra
```

---

## Riepilogo: Quale Approccio per il Database?

| | **SQLite diretto + Repository** | **SQLAlchemy + Models** |
|---|---|---|
| File necessari | `schema.sql`, `db.py`, `setup_db.py`, `repositories/` | `models.py` |
| Come si interroga | SQL puro (`SELECT * FROM ...`) | Python (`Post.query.all()`) |
| Creazione tabelle | `python setup_db.py` manuale | `db.create_all()` automatico |
| Quando usarlo | Vuoi controllare ogni query SQL | Vuoi scrivere meno codice |
