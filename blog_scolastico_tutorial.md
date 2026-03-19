# Tutorial Completo: Blog Scolastico — 5m_v5_sol

App Flask per un blog scolastico con autenticazione, post, preferiti (bookmarks), profilo utente e archivio per data. Usa **SQLite diretto** (senza ORM) con il **pattern Repository**.

---

## FASE 1 — Ambiente Virtuale

```bash
cd blog_scolastico

python3 -m venv .venv

# Attiva (macOS/Linux)
source .venv/bin/activate

# Attiva (Windows)
.venv\Scripts\activate
```

---

## FASE 2 — Pacchetti da Installare

```bash
pip install flask werkzeug python-dotenv
pip freeze > requirements.txt
```

| Pacchetto | Ruolo |
|---|---|
| `flask` | Framework web |
| `werkzeug` | Hash delle password |
| `python-dotenv` | Legge `.env` |

> `sqlite3` è già incluso in Python — nessun ORM necessario.

---

## FASE 3 — File .env e .gitignore

### `.env`
```
SECRET_KEY=chiave_segreta_lunga_123
DATABASE=instance/blog.sqlite
DEBUG=True
```

### `.gitignore`
```
.venv/
.env
instance/
__pycache__/
*.pyc
*.sqlite
```

---

## FASE 4 — Struttura del Progetto

```
blog_scolastico/
├── .venv/
├── .env
├── .gitignore
├── requirements.txt
├── run.py                         ← avvia l'app
├── setup_db.py                    ← crea il database
├── instance/
│   └── blog.sqlite                ← creato da setup_db.py
└── app/
    ├── __init__.py                ← crea l'app Flask, carica .env
    ├── db.py                      ← connessione SQLite
    ├── auth.py                    ← blueprint login/register/logout
    ├── main.py                    ← blueprint pagine principali
    ├── schema.sql                 ← tabelle + dati di prova
    ├── repositories/
    │   ├── __init__.py
    │   ├── user_repository.py
    │   ├── post_repository.py
    │   └── bookmark_repository.py
    └── templates/
        ├── base.html
        ├── index.html
        ├── about.html
        ├── user_profile.html
        ├── bookmarks.html
        ├── archive.html
        ├── auth/
        │   ├── login.html
        │   └── register.html
        └── blog/
            ├── create.html
            └── update.html
```

---

## FASE 5 — Database

### `app/schema.sql`
```sql
DROP TABLE IF EXISTS user;
DROP TABLE IF EXISTS post;
DROP TABLE IF EXISTS bookmark;

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

-- Chiave primaria COMPOSTA: un utente non può salvare lo stesso post due volte
CREATE TABLE bookmark (
  user_id  INTEGER NOT NULL,
  post_id  INTEGER NOT NULL,
  created  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, post_id),
  FOREIGN KEY (user_id) REFERENCES user (id),
  FOREIGN KEY (post_id) REFERENCES post (id)
);

-- Dati di esempio
INSERT INTO user (username, password) VALUES ('admin', 'adminpass');
INSERT INTO user (username, password) VALUES ('student', 'studentpass');

INSERT INTO post (author_id, title, body, created)
  VALUES (1, 'Welcome to the Blog', 'Primo post!', '2024-01-15 10:30:00');
INSERT INTO post (author_id, title, body, created)
  VALUES (2, 'Hello World', 'Post dello studente.', '2024-03-20 14:15:00');
```

### `setup_db.py`
```python
import sqlite3
import os

if not os.path.exists('instance'):
    os.makedirs('instance')

db_path = os.path.join('instance', 'blog.sqlite')
connection = sqlite3.connect(db_path)

with open('app/schema.sql') as f:
    connection.executescript(f.read())

print("Database creato con successo in:", db_path)
connection.close()
```

```bash
# Esegui UNA sola volta
python setup_db.py
```

---

## FASE 6 — Codice Python

### `run.py`
```python
from app import create_app

app = create_app()

if __name__ == '__main__':
    app.run()
```

---

### `app/db.py` — Connessione al Database

```python
import sqlite3
from flask import current_app, g

def get_db():
    """Restituisce la connessione per la richiesta corrente."""
    if 'db' not in g:
        g.db = sqlite3.connect(current_app.config['DATABASE'])
        g.db.row_factory = sqlite3.Row  # le righe si usano come dizionari
    return g.db

def close_db(e=None):
    """Chiude la connessione alla fine della richiesta."""
    db = g.pop('db', None)
    if db is not None:
        db.close()

def init_app(app):
    app.teardown_appcontext(close_db)
```

> `g` è un oggetto Flask che vive solo per la durata di una singola richiesta HTTP.

---

### `app/__init__.py`
```python
import os
from flask import Flask
from dotenv import load_dotenv

load_dotenv()  # legge .env

def create_app():
    app = Flask(__name__, instance_relative_config=True)

    app.config.from_mapping(
        SECRET_KEY = os.getenv('SECRET_KEY', 'dev'),
        DATABASE   = os.path.join(app.instance_path, 'blog.sqlite'),
        DEBUG      = os.getenv('DEBUG', 'False') == 'True',
    )

    from . import db
    db.init_app(app)

    from . import main
    app.register_blueprint(main.bp)

    from . import auth
    app.register_blueprint(auth.bp)

    return app
```

---

### `app/auth.py` — Autenticazione
```python
from flask import (Blueprint, flash, g, redirect,
                   render_template, request, session, url_for)
from werkzeug.security import check_password_hash, generate_password_hash
from app.repositories import user_repository

bp = Blueprint('auth', __name__, url_prefix='/auth')


@bp.before_app_request
def load_logged_in_user():
    """
    Eseguita automaticamente prima di OGNI richiesta.
    Carica l'utente dalla sessione e lo mette in g.user,
    disponibile in tutti i template come g.user.
    """
    user_id = session.get('user_id')
    g.user = user_repository.get_user_by_id(user_id) if user_id else None


@bp.route('/register', methods=('GET', 'POST'))
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        error = None

        if not username:
            error = 'Username obbligatorio.'
        elif not password:
            error = 'Password obbligatoria.'

        if error is None:
            success = user_repository.create_user(username, generate_password_hash(password))
            if success:
                return redirect(url_for('auth.login'))
            else:
                error = f"L'utente {username} è già registrato."

        flash(error)
    return render_template('auth/register.html')


@bp.route('/login', methods=('GET', 'POST'))
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        error = None

        user = user_repository.get_user_by_username(username)

        if user is None:
            error = 'Username non corretto.'
        elif not check_password_hash(user['password'], password):
            error = 'Password non corretta.'

        if error is None:
            session.clear()
            session['user_id'] = user['id']
            return redirect(url_for('main.index'))

        flash(error)
    return render_template('auth/login.html')


@bp.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('main.index'))
```

---

### `app/main.py` — Route Principali
```python
from flask import Blueprint, flash, g, redirect, render_template, request, url_for
from werkzeug.exceptions import abort
from app.repositories import post_repository, user_repository, bookmark_repository

bp = Blueprint("main", __name__)


@bp.route("/")
def index():
    posts = post_repository.get_all_posts()
    bookmarked_ids = []
    if g.user:
        bookmarks = bookmark_repository.get_user_bookmarks(g.user["id"])
        bookmarked_ids = [b["id"] for b in bookmarks]
    return render_template("index.html", posts=posts, bookmarked_ids=bookmarked_ids)


@bp.route("/about")
def about():
    return render_template("about.html")


@bp.route("/create", methods=("GET", "POST"))
def create():
    if g.user is None:
        return redirect(url_for("auth.login"))
    if request.method == "POST":
        title = request.form["title"]
        body  = request.form["body"]
        error = None
        if not title:
            error = "Il titolo è obbligatorio."
        if error is not None:
            flash(error)
        else:
            post_repository.create_post(title, body, g.user["id"])
            return redirect(url_for("main.index"))
    return render_template("blog/create.html")


def get_post(id, check_author=True):
    post = post_repository.get_post_by_id(id)
    if post is None:
        abort(404, f"Il post id {id} non esiste.")
    if check_author and post["author_id"] != g.user["id"]:
        abort(403)
    return post


@bp.route("/<int:id>/update", methods=("GET", "POST"))
def update(id):
    if g.user is None:
        return redirect(url_for("auth.login"))
    post = get_post(id)
    if request.method == "POST":
        title = request.form["title"]
        body  = request.form["body"]
        error = None
        if not title:
            error = "Il titolo è obbligatorio."
        if error is not None:
            flash(error)
        else:
            post_repository.update_post(id, title, body)
            return redirect(url_for("main.index"))
    return render_template("blog/update.html", post=post)


@bp.route("/<int:id>/delete", methods=("POST",))
def delete(id):
    if g.user is None:
        return redirect(url_for("auth.login"))
    get_post(id)
    post_repository.delete_post(id)
    return redirect(url_for("main.index"))


@bp.route("/user/<int:id>")
def user_profile(id):
    user = user_repository.get_user_by_id(id)
    if user is None:
        abort(404)
    posts = post_repository.get_posts_by_author(id)
    return render_template("user_profile.html", user=user, posts=posts)


@bp.route("/archive", methods=("GET", "POST"))
def archive():
    posts, selected_year, selected_month = [], None, None
    if request.method == "POST":
        year_str  = request.form.get("year")
        month_str = request.form.get("month")
        if year_str and month_str:
            selected_year  = int(year_str)
            selected_month = int(month_str)
            posts = post_repository.get_posts_by_month(selected_year, selected_month)
    return render_template("archive.html", posts=posts,
                           selected_year=selected_year, selected_month=selected_month)


@bp.route("/post/<int:id>/bookmark", methods=("POST",))
def toggle_bookmark(id):
    if g.user is None:
        return redirect(url_for("auth.login"))
    if bookmark_repository.is_bookmarked(g.user["id"], id):
        bookmark_repository.remove_bookmark(g.user["id"], id)
    else:
        bookmark_repository.add_bookmark(g.user["id"], id)
    return redirect(url_for("main.index"))


@bp.route("/bookmarks")
def bookmarks():
    if g.user is None:
        return redirect(url_for("auth.login"))
    bookmarks = bookmark_repository.get_user_bookmarks(g.user["id"])
    return render_template("bookmarks.html", bookmarks=bookmarks)
```

---

## FASE 7 — Repository

### `app/repositories/user_repository.py`
```python
from app.db import get_db

def create_user(username, password_hash):
    db = get_db()
    try:
        db.execute("INSERT INTO user (username, password) VALUES (?, ?)",
                   (username, password_hash))
        db.commit()
        return True
    except db.IntegrityError:
        return False

def get_user_by_username(username):
    return get_db().execute(
        "SELECT * FROM user WHERE username = ?", (username,)
    ).fetchone()

def get_user_by_id(user_id):
    return get_db().execute(
        "SELECT * FROM user WHERE id = ?", (user_id,)
    ).fetchone()
```

---

### `app/repositories/post_repository.py`
```python
from app.db import get_db
from datetime import datetime


def get_all_posts():
    posts = get_db().execute("""
        SELECT p.id, p.title, p.body, p.created, p.author_id, u.username
        FROM post p JOIN user u ON p.author_id = u.id
        ORDER BY p.created DESC
    """).fetchall()
    result = []
    for post in posts:
        d = dict(post)
        d["created"] = datetime.fromisoformat(d["created"])
        result.append(d)
    return result


def get_post_by_id(post_id):
    post = get_db().execute("""
        SELECT p.id, p.title, p.body, p.created, p.author_id, u.username
        FROM post p JOIN user u ON p.author_id = u.id
        WHERE p.id = ?
    """, (post_id,)).fetchone()
    if post:
        d = dict(post)
        d["created"] = datetime.fromisoformat(d["created"])
        return d
    return None


def create_post(title, body, author_id):
    db = get_db()
    db.execute("INSERT INTO post (title, body, author_id) VALUES (?, ?, ?)",
               (title, body, author_id))
    db.commit()


def update_post(post_id, title, body):
    db = get_db()
    db.execute("UPDATE post SET title = ?, body = ? WHERE id = ?", (title, body, post_id))
    db.commit()


def delete_post(post_id):
    db = get_db()
    db.execute("DELETE FROM post WHERE id = ?", (post_id,))
    db.commit()


def get_posts_by_author(author_id):
    posts = get_db().execute("""
        SELECT p.id, p.title, p.body, p.created, p.author_id, u.username
        FROM post p JOIN user u ON p.author_id = u.id
        WHERE p.author_id = ?
        ORDER BY p.created DESC
    """, (author_id,)).fetchall()
    result = []
    for post in posts:
        d = dict(post)
        d["created"] = datetime.fromisoformat(d["created"])
        result.append(d)
    return result


def get_posts_by_month(year, month):
    posts = get_db().execute("""
        SELECT p.id, p.title, p.body, p.created, p.author_id, u.username
        FROM post p JOIN user u ON p.author_id = u.id
        WHERE strftime('%Y', p.created) = ?
          AND strftime('%m', p.created) = ?
        ORDER BY p.created DESC
    """, (str(year), str(month).zfill(2))).fetchall()
    result = []
    for post in posts:
        d = dict(post)
        d["created"] = datetime.fromisoformat(d["created"])
        result.append(d)
    return result
```

---

### `app/repositories/bookmark_repository.py`
```python
from app.db import get_db
from datetime import datetime


def add_bookmark(user_id, post_id):
    db = get_db()
    db.execute("INSERT INTO bookmark (user_id, post_id) VALUES (?, ?)", (user_id, post_id))
    db.commit()


def remove_bookmark(user_id, post_id):
    db = get_db()
    db.execute("DELETE FROM bookmark WHERE user_id = ? AND post_id = ?", (user_id, post_id))
    db.commit()


def get_user_bookmarks(user_id):
    bookmarks = get_db().execute("""
        SELECT p.id, p.title, p.body, p.created, p.author_id,
               u.username, b.created as bookmarked_at
        FROM bookmark b
        JOIN post p ON b.post_id = p.id
        JOIN user u ON p.author_id = u.id
        WHERE b.user_id = ?
        ORDER BY b.created DESC
    """, (user_id,)).fetchall()
    result = []
    for b in bookmarks:
        d = dict(b)
        d["created"]       = datetime.fromisoformat(d["created"])
        d["bookmarked_at"] = datetime.fromisoformat(d["bookmarked_at"])
        result.append(d)
    return result


def is_bookmarked(user_id, post_id):
    result = get_db().execute(
        "SELECT 1 FROM bookmark WHERE user_id = ? AND post_id = ?",
        (user_id, post_id)
    ).fetchone()
    return result is not None
```

---

## FASE 8 — Template HTML

### `app/templates/base.html`
```html
<!doctype html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Blog Scolastico{% endblock %}</title>
    <style>
        body { font-family: sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
        nav a { margin-right: 10px; text-decoration: none; color: blue; }
    </style>
</head>
<body>
    <nav>
        <a href="{{ url_for('main.index') }}">Blog Scolastico</a>
        {% if g.user %}
            <span>Ciao, {{ g.user['username'] }}</span>
            <a href="{{ url_for('auth.logout') }}">Logout</a>
        {% else %}
            <a href="{{ url_for('auth.register') }}">Registrati</a>
            <a href="{{ url_for('auth.login') }}">Login</a>
        {% endif %}
    </nav>
    <hr>
    {% for message in get_flashed_messages() %}
        <div style="color: red; border: 1px solid red; padding: 10px;">{{ message }}</div>
    {% endfor %}
    <main>{% block content %}{% endblock %}</main>
    <hr>
    <footer><small>&copy; 2024 - Classe 5^A Informatica</small></footer>
</body>
</html>
```

---

### `app/templates/index.html`
```html
{% extends 'base.html' %}
{% block content %}
<h1>Blog della Classe</h1>

{% if g.user %}
<a href="{{ url_for('main.create') }}">Scrivi un nuovo post</a> |
<a href="{{ url_for('main.bookmarks') }}">I miei Preferiti</a>
<hr>
{% endif %}

{% for post in posts %}
<article>
  <h2>{{ post['title'] }}</h2>
  <small>
    Scritto da
    <a href="{{ url_for('main.user_profile', id=post['author_id']) }}">
      <strong>{{ post['username'] }}</strong>
    </a>
    il {{ post['created'].strftime('%Y-%m-%d') }}
  </small>
  <p>{{ post['body'] }}</p>

  {% if g.user and g.user['id'] == post['author_id'] %}
    <a href="{{ url_for('main.update', id=post['id']) }}">Modifica</a>
  {% endif %}

  {% if g.user %}
  <form method="post" action="{{ url_for('main.toggle_bookmark', id=post['id']) }}" style="display:inline;">
    {% if post['id'] in bookmarked_ids %}
      <button type="submit">★ Salvato</button>
    {% else %}
      <button type="submit">☆ Salva</button>
    {% endif %}
  </form>
  {% endif %}
</article>
<hr>
{% endfor %}

<a href="{{ url_for('main.archive') }}">Archivio per data</a>
{% endblock %}
```

---

### `app/templates/auth/register.html`
```html
{% extends 'base.html' %}
{% block content %}
  <h2>Registrazione</h2>
  <form method="post">
    <label>Username</label><input name="username" required><br>
    <label>Password</label><input type="password" name="password" required><br>
    <input type="submit" value="Registrati">
  </form>
{% endblock %}
```

### `app/templates/auth/login.html`
```html
{% extends 'base.html' %}
{% block content %}
  <h2>Accedi</h2>
  <form method="post">
    <label>Username</label><input name="username" required><br>
    <label>Password</label><input type="password" name="password" required><br>
    <input type="submit" value="Login">
  </form>
{% endblock %}
```

---

### `app/templates/blog/create.html`
```html
{% extends 'base.html' %}
{% block content %}
  <h1>Nuovo Post</h1>
  <form method="post">
    <label>Titolo</label><input name="title" required><br>
    <label>Testo</label><textarea name="body" rows="5" required></textarea><br>
    <input type="submit" value="Pubblica">
  </form>
{% endblock %}
```

### `app/templates/blog/update.html`
```html
{% extends 'base.html' %}
{% block content %}
  <h1>Modifica "{{ post['title'] }}"</h1>
  <form method="post">
    <label>Titolo</label>
    <input name="title" value="{{ request.form.get('title') or post['title'] }}" required><br>
    <label>Testo</label>
    <textarea name="body">{{ request.form.get('body') or post['body'] }}</textarea><br>
    <input type="submit" value="Salva Modifiche">
  </form>
  <hr>
  <form action="{{ url_for('main.delete', id=post['id']) }}" method="post">
    <input type="submit" value="Elimina Post" onclick="return confirm('Sei sicuro?')">
  </form>
{% endblock %}
```

---

### `app/templates/user_profile.html`
```html
{% extends 'base.html' %}
{% block content %}
<h1>Profilo di {{ user['username'] }}</h1>
<p><strong>Post pubblicati:</strong> {{ posts|length }}</p>
<hr>
<h2>I suoi post</h2>
{% if posts %}
<ul>
    {% for post in posts %}
    <li>{{ post['title'] }} <small>({{ post['created'].strftime('%Y-%m-%d') }})</small></li>
    {% endfor %}
</ul>
{% else %}
<p>Nessun post ancora.</p>
{% endif %}
<a href="{{ url_for('main.index') }}">← Torna alla home</a>
{% endblock %}
```

---

### `app/templates/bookmarks.html`
```html
{% extends 'base.html' %}
{% block content %}
<h1>I miei Preferiti</h1>
{% if bookmarks %}
{% for post in bookmarks %}
<article>
    <h2>{{ post['title'] }}</h2>
    <small>
        Di <strong>{{ post['username'] }}</strong>
        il {{ post['created'].strftime('%Y-%m-%d') }}
        — Salvato il {{ post['bookmarked_at'].strftime('%Y-%m-%d') }}
    </small>
    <p>{{ post['body'][:150] }}...</p>
    <form method="post" action="{{ url_for('main.toggle_bookmark', id=post['id']) }}">
        <button type="submit">Rimuovi dai preferiti</button>
    </form>
</article>
<hr>
{% endfor %}
{% else %}
<p>Non hai ancora salvato nessun post.</p>
{% endif %}
<a href="{{ url_for('main.index') }}">← Torna alla home</a>
{% endblock %}
```

---

### `app/templates/archive.html`
```html
{% extends 'base.html' %}
{% block content %}
<h1>Archivio Post</h1>
<form method="post">
    <label>Anno:</label>
    <select name="year">
        <option value="">-- Seleziona --</option>
        {% for y in [2024, 2025, 2026] %}
        <option value="{{ y }}" {% if selected_year == y %}selected{% endif %}>{{ y }}</option>
        {% endfor %}
    </select>
    <label>Mese:</label>
    <select name="month">
        <option value="">-- Seleziona --</option>
        {% set mesi = ['Gennaio','Febbraio','Marzo','Aprile','Maggio','Giugno',
                       'Luglio','Agosto','Settembre','Ottobre','Novembre','Dicembre'] %}
        {% for i in range(1, 13) %}
        <option value="{{ i }}" {% if selected_month == i %}selected{% endif %}>{{ mesi[i-1] }}</option>
        {% endfor %}
    </select>
    <button type="submit">Cerca</button>
</form>
<hr>
{% if selected_year and selected_month %}
<h2>Risultati per {{ '%02d' % selected_month }}/{{ selected_year }}</h2>
{% if posts %}
    {% for post in posts %}
    <article>
        <h3>{{ post['title'] }}</h3>
        <small>Di <strong>{{ post['username'] }}</strong> il {{ post['created'].strftime('%Y-%m-%d') }}</small>
        <p>{{ post['body'][:100] }}...</p>
    </article>
    <hr>
    {% endfor %}
{% else %}
<p>Nessun post trovato per questo periodo.</p>
{% endif %}
{% endif %}
<a href="{{ url_for('main.index') }}">← Torna alla home</a>
{% endblock %}
```

---

## FASE 9 — Avviare l'App

```bash
source .venv/bin/activate
cd blog_scolastico
python setup_db.py   # solo la prima volta
python run.py
```

Apri **`http://127.0.0.1:5000`**

---

## Riepilogo Concetti Chiave

| Concetto | Come funziona |
|---|---|
| **Blueprint** | `auth.py` e `main.py` sono moduli separati registrati nell'app |
| **`g`** | Oggetto Flask che vive solo per una singola richiesta HTTP |
| **`session`** | Cookie cifrato — contiene solo `user_id` dopo il login |
| **`before_app_request`** | Esegue `load_logged_in_user` prima di ogni richiesta |
| **Repository Pattern** | Tutto il SQL sta nei repository — le route non toccano mai il DB |
| **`row_factory`** | `sqlite3.Row` permette `post['title']` invece di `post[0]` |
| **`abort(403/404)`** | Lancia la pagina di errore HTTP corrispondente |
| **Chiave primaria composta** | `bookmark(user_id, post_id)` — evita duplicati |
| **`strftime` SQLite** | Filtra per mese/anno direttamente nella query SQL |
| **`toggle_bookmark`** | Stessa route aggiunge O rimuove — controlla con `is_bookmarked()` |

```
Browser → Blueprint → Repository (SQL) → SQLite → Template HTML → Browser
```
