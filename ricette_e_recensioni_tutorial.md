# Tutorial Completo: App "Ricette & Recensioni"

Tutorial per creare una web app con Flask, SQLAlchemy, Flask-Login, Flask-WTF e python-dotenv.

---

## FASE 1 — Ambiente Virtuale

```bash
mkdir ricette_e_recensioni
cd ricette_e_recensioni

python3 -m venv .venv

# Attivalo (macOS/Linux)
source .venv/bin/activate

# Attivalo (Windows)
.venv\Scripts\activate
```

Vedrai `(.venv)` nel terminale — significa che sei dentro l'ambiente virtuale.

---

## FASE 2 — Pacchetti da Installare

```bash
pip install flask flask-sqlalchemy flask-login flask-wtf python-dotenv email-validator werkzeug
```

| Pacchetto | Ruolo nell'app |
|---|---|
| `flask` | Framework web principale |
| `flask-sqlalchemy` | Gestione database (ricette, utenti, recensioni) |
| `flask-login` | Login/logout utenti e gestione sessioni |
| `flask-wtf` | Form con validazione e protezione CSRF |
| `python-dotenv` | Legge le variabili dal file .env |
| `email-validator` | Valida le email nei form di registrazione |
| `werkzeug` | Hash sicuro delle password |

```bash
pip freeze > requirements.txt
```

---

## FASE 3 — Struttura del Progetto

```
ricette_e_recensioni/
├── .venv/
├── .env                   ← variabili segrete
├── .gitignore
├── requirements.txt
├── run.py
└── app/
    ├── __init__.py
    ├── models.py
    ├── routes.py
    ├── forms.py
    ├── static/
    │   └── style.css
    └── templates/
        ├── base.html
        ├── index.html
        ├── register.html
        ├── login.html
        ├── nuova_ricetta.html
        ├── ricetta.html
        └── profilo.html
```

---

## FASE 4 — File di Configurazione

### `.env`
```
SECRET_KEY=chiave_segreta_molto_lunga_e_casuale_123456
DATABASE_URL=sqlite:///ricette.db
DEBUG=True
```

### `.gitignore`
```
.venv/
.env
*.db
__pycache__/
*.pyc
```

---

## FASE 5 — Il Codice

### `app/__init__.py`
```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager
from dotenv import load_dotenv
import os

load_dotenv()

db = SQLAlchemy()
login_manager = LoginManager()

def create_app():
    app = Flask(__name__)

    app.config['SECRET_KEY'] = os.getenv('SECRET_KEY')
    app.config['SQLALCHEMY_DATABASE_URI'] = os.getenv('DATABASE_URL')
    app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
    app.config['DEBUG'] = os.getenv('DEBUG', 'False') == 'True'

    db.init_app(app)
    login_manager.init_app(app)
    login_manager.login_view = 'main.login'
    login_manager.login_message = 'Devi fare il login per accedere a questa pagina.'

    from .routes import main
    app.register_blueprint(main)

    with app.app_context():
        db.create_all()

    return app
```

---

### `app/models.py`
```python
from . import db, login_manager
from flask_login import UserMixin
from datetime import datetime
from werkzeug.security import generate_password_hash, check_password_hash

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))


class User(UserMixin, db.Model):
    __tablename__ = 'users'

    id              = db.Column(db.Integer, primary_key=True)
    username        = db.Column(db.String(80), unique=True, nullable=False)
    email           = db.Column(db.String(120), unique=True, nullable=False)
    password        = db.Column(db.String(200), nullable=False)
    data_iscrizione = db.Column(db.DateTime, default=datetime.utcnow)

    ricette    = db.relationship('Ricetta', backref='autore', lazy=True)
    recensioni = db.relationship('Recensione', backref='autore', lazy=True)

    def set_password(self, password_plaintext):
        self.password = generate_password_hash(password_plaintext)

    def check_password(self, password_plaintext):
        return check_password_hash(self.password, password_plaintext)


class Ricetta(db.Model):
    __tablename__ = 'ricette'

    id             = db.Column(db.Integer, primary_key=True)
    titolo         = db.Column(db.String(200), nullable=False)
    descrizione    = db.Column(db.Text, nullable=False)
    ingredienti    = db.Column(db.Text, nullable=False)
    procedimento   = db.Column(db.Text, nullable=False)
    categoria      = db.Column(db.String(50), nullable=False)
    difficolta     = db.Column(db.String(20), nullable=False)
    tempo_min      = db.Column(db.Integer, nullable=False)
    data_creazione = db.Column(db.DateTime, default=datetime.utcnow)
    user_id        = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)

    recensioni = db.relationship('Recensione', backref='ricetta', lazy=True, cascade='all, delete-orphan')

    def media_voti(self):
        if not self.recensioni:
            return None
        return round(sum(r.voto for r in self.recensioni) / len(self.recensioni), 1)


class Recensione(db.Model):
    __tablename__ = 'recensioni'

    id             = db.Column(db.Integer, primary_key=True)
    testo          = db.Column(db.Text, nullable=False)
    voto           = db.Column(db.Integer, nullable=False)
    data_creazione = db.Column(db.DateTime, default=datetime.utcnow)
    user_id        = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    ricetta_id     = db.Column(db.Integer, db.ForeignKey('ricette.id'), nullable=False)
```

---

### `app/forms.py`
```python
from flask_wtf import FlaskForm
from wtforms import (StringField, PasswordField, SubmitField,
                     TextAreaField, SelectField, IntegerField)
from wtforms.validators import DataRequired, Email, Length, EqualTo, NumberRange


class RegisterForm(FlaskForm):
    username = StringField('Username', validators=[DataRequired(), Length(min=3, max=80)])
    email    = StringField('Email', validators=[DataRequired(), Email()])
    password = PasswordField('Password', validators=[DataRequired(), Length(min=6)])
    conferma = PasswordField('Conferma Password',
                             validators=[DataRequired(), EqualTo('password', message='Le password non coincidono')])
    submit   = SubmitField('Registrati')


class LoginForm(FlaskForm):
    email    = StringField('Email', validators=[DataRequired(), Email()])
    password = PasswordField('Password', validators=[DataRequired()])
    submit   = SubmitField('Accedi')


class RicettaForm(FlaskForm):
    titolo       = StringField('Titolo', validators=[DataRequired(), Length(min=3, max=200)])
    descrizione  = TextAreaField('Descrizione breve', validators=[DataRequired()])
    ingredienti  = TextAreaField('Ingredienti (uno per riga)', validators=[DataRequired()])
    procedimento = TextAreaField('Procedimento', validators=[DataRequired()])
    categoria    = SelectField('Categoria',
                               choices=[('Antipasti','Antipasti'),('Primi','Primi'),
                                        ('Secondi','Secondi'),('Contorni','Contorni'),
                                        ('Dolci','Dolci'),('Bevande','Bevande')])
    difficolta   = SelectField('Difficoltà',
                               choices=[('Facile','Facile'),('Media','Media'),('Difficile','Difficile')])
    tempo_min    = IntegerField('Tempo (minuti)', validators=[DataRequired(), NumberRange(min=1)])
    submit       = SubmitField('Pubblica Ricetta')


class RecensioneForm(FlaskForm):
    testo  = TextAreaField('La tua recensione', validators=[DataRequired(), Length(min=10)])
    voto   = SelectField('Voto',
                         choices=[(1,'⭐ 1'),(2,'⭐ 2'),(3,'⭐ 3'),(4,'⭐ 4'),(5,'⭐ 5')],
                         coerce=int)
    submit = SubmitField('Invia Recensione')
```

---

### `app/routes.py`
```python
from flask import Blueprint, render_template, redirect, url_for, flash, request
from flask_login import login_user, logout_user, login_required, current_user
from . import db
from .models import User, Ricetta, Recensione
from .forms import RegisterForm, LoginForm, RicettaForm, RecensioneForm

main = Blueprint('main', __name__)


@main.route('/')
def index():
    categoria = request.args.get('categoria')
    if categoria:
        ricette = Ricetta.query.filter_by(categoria=categoria).order_by(Ricetta.data_creazione.desc()).all()
    else:
        ricette = Ricetta.query.order_by(Ricetta.data_creazione.desc()).all()
    return render_template('index.html', ricette=ricette, categoria=categoria)


@main.route('/register', methods=['GET', 'POST'])
def register():
    if current_user.is_authenticated:
        return redirect(url_for('main.index'))
    form = RegisterForm()
    if form.validate_on_submit():
        if User.query.filter_by(username=form.username.data).first():
            flash('Username già in uso.', 'danger')
            return render_template('register.html', form=form)
        if User.query.filter_by(email=form.email.data).first():
            flash('Email già registrata.', 'danger')
            return render_template('register.html', form=form)
        utente = User(username=form.username.data, email=form.email.data)
        utente.set_password(form.password.data)
        db.session.add(utente)
        db.session.commit()
        flash('Registrazione avvenuta! Ora puoi fare il login.', 'success')
        return redirect(url_for('main.login'))
    return render_template('register.html', form=form)


@main.route('/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('main.index'))
    form = LoginForm()
    if form.validate_on_submit():
        utente = User.query.filter_by(email=form.email.data).first()
        if utente and utente.check_password(form.password.data):
            login_user(utente)
            flash(f'Bentornato, {utente.username}!', 'success')
            next_page = request.args.get('next')
            return redirect(next_page or url_for('main.index'))
        flash('Email o password errati.', 'danger')
    return render_template('login.html', form=form)


@main.route('/logout')
@login_required
def logout():
    logout_user()
    flash('Hai effettuato il logout.', 'info')
    return redirect(url_for('main.index'))


@main.route('/ricetta/nuova', methods=['GET', 'POST'])
@login_required
def nuova_ricetta():
    form = RicettaForm()
    if form.validate_on_submit():
        ricetta = Ricetta(
            titolo=form.titolo.data, descrizione=form.descrizione.data,
            ingredienti=form.ingredienti.data, procedimento=form.procedimento.data,
            categoria=form.categoria.data, difficolta=form.difficolta.data,
            tempo_min=form.tempo_min.data, user_id=current_user.id
        )
        db.session.add(ricetta)
        db.session.commit()
        flash('Ricetta pubblicata!', 'success')
        return redirect(url_for('main.ricetta', ricetta_id=ricetta.id))
    return render_template('nuova_ricetta.html', form=form)


@main.route('/ricetta/<int:ricetta_id>', methods=['GET', 'POST'])
def ricetta(ricetta_id):
    ricetta = Ricetta.query.get_or_404(ricetta_id)
    form    = RecensioneForm()
    if form.validate_on_submit():
        if not current_user.is_authenticated:
            flash('Devi fare il login per lasciare una recensione.', 'warning')
            return redirect(url_for('main.login'))
        già_recensita = Recensione.query.filter_by(user_id=current_user.id, ricetta_id=ricetta_id).first()
        if già_recensita:
            flash('Hai già recensito questa ricetta.', 'warning')
        else:
            recensione = Recensione(testo=form.testo.data, voto=form.voto.data,
                                    user_id=current_user.id, ricetta_id=ricetta_id)
            db.session.add(recensione)
            db.session.commit()
            flash('Recensione aggiunta!', 'success')
        return redirect(url_for('main.ricetta', ricetta_id=ricetta_id))
    return render_template('ricetta.html', ricetta=ricetta, form=form)


@main.route('/ricetta/<int:ricetta_id>/elimina', methods=['POST'])
@login_required
def elimina_ricetta(ricetta_id):
    ricetta = Ricetta.query.get_or_404(ricetta_id)
    if ricetta.user_id != current_user.id:
        flash('Non puoi eliminare questa ricetta.', 'danger')
        return redirect(url_for('main.ricetta', ricetta_id=ricetta_id))
    db.session.delete(ricetta)
    db.session.commit()
    flash('Ricetta eliminata.', 'info')
    return redirect(url_for('main.index'))


@main.route('/profilo/<int:user_id>')
def profilo(user_id):
    utente  = User.query.get_or_404(user_id)
    ricette = Ricetta.query.filter_by(user_id=user_id).order_by(Ricetta.data_creazione.desc()).all()
    return render_template('profilo.html', utente=utente, ricette=ricette)
```

---

### `app/templates/base.html`
```html
<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Ricette & Recensioni{% endblock %}</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
<nav>
    <a class="brand" href="{{ url_for('main.index') }}">Ricette & Recensioni</a>
    <div class="nav-links">
        <a href="{{ url_for('main.index') }}">Home</a>
        {% if current_user.is_authenticated %}
            <a href="{{ url_for('main.nuova_ricetta') }}">+ Nuova Ricetta</a>
            <a href="{{ url_for('main.profilo', user_id=current_user.id) }}">{{ current_user.username }}</a>
            <a href="{{ url_for('main.logout') }}">Logout</a>
        {% else %}
            <a href="{{ url_for('main.login') }}">Login</a>
            <a href="{{ url_for('main.register') }}">Registrati</a>
        {% endif %}
    </div>
</nav>

<main>
    {% with messages = get_flashed_messages(with_categories=true) %}
        {% for categoria, messaggio in messages %}
            <div class="alert alert-{{ categoria }}">{{ messaggio }}</div>
        {% endfor %}
    {% endwith %}
    {% block content %}{% endblock %}
</main>

<footer><p>Ricette & Recensioni — fatto con Flask</p></footer>
</body>
</html>
```

---

### `app/templates/index.html`
```html
{% extends 'base.html' %}
{% block title %}Home — Ricette{% endblock %}
{% block content %}
<h1>Tutte le Ricette</h1>

<div class="filtri">
    <a href="{{ url_for('main.index') }}" class="{% if not categoria %}attivo{% endif %}">Tutte</a>
    {% for cat in ['Antipasti','Primi','Secondi','Contorni','Dolci','Bevande'] %}
        <a href="{{ url_for('main.index', categoria=cat) }}"
           class="{% if categoria == cat %}attivo{% endif %}">{{ cat }}</a>
    {% endfor %}
</div>

<div class="griglia">
    {% for ricetta in ricette %}
    <div class="card">
        <div class="card-body">
            <span class="badge">{{ ricetta.categoria }}</span>
            <h2><a href="{{ url_for('main.ricetta', ricetta_id=ricetta.id) }}">{{ ricetta.titolo }}</a></h2>
            <p>{{ ricetta.descrizione[:100] }}...</p>
            <div class="card-meta">
                <span>{{ ricetta.difficolta }}</span>
                <span>{{ ricetta.tempo_min }} min</span>
                {% if ricetta.media_voti() %}<span>★ {{ ricetta.media_voti() }}</span>{% endif %}
            </div>
            <small>di <a href="{{ url_for('main.profilo', user_id=ricetta.autore.id) }}">{{ ricetta.autore.username }}</a></small>
        </div>
    </div>
    {% else %}
    <p>Nessuna ricetta trovata.</p>
    {% endfor %}
</div>
{% endblock %}
```

---

### `app/templates/register.html`
```html
{% extends 'base.html' %}
{% block title %}Registrati{% endblock %}
{% block content %}
<div class="form-container">
    <h1>Registrati</h1>
    <form method="POST">
        {{ form.hidden_tag() }}
        {% for field in [form.username, form.email, form.password, form.conferma] %}
        <div class="campo">
            {{ field.label }}
            {{ field() }}
            {% for errore in field.errors %}<span class="errore">{{ errore }}</span>{% endfor %}
        </div>
        {% endfor %}
        {{ form.submit(class='btn') }}
    </form>
    <p>Hai già un account? <a href="{{ url_for('main.login') }}">Accedi</a></p>
</div>
{% endblock %}
```

---

### `app/templates/login.html`
```html
{% extends 'base.html' %}
{% block title %}Login{% endblock %}
{% block content %}
<div class="form-container">
    <h1>Accedi</h1>
    <form method="POST">
        {{ form.hidden_tag() }}
        <div class="campo">
            {{ form.email.label }}{{ form.email(placeholder='tua@email.com') }}
            {% for errore in form.email.errors %}<span class="errore">{{ errore }}</span>{% endfor %}
        </div>
        <div class="campo">
            {{ form.password.label }}{{ form.password() }}
            {% for errore in form.password.errors %}<span class="errore">{{ errore }}</span>{% endfor %}
        </div>
        {{ form.submit(class='btn') }}
    </form>
    <p>Non hai un account? <a href="{{ url_for('main.register') }}">Registrati</a></p>
</div>
{% endblock %}
```

---

### `app/templates/nuova_ricetta.html`
```html
{% extends 'base.html' %}
{% block title %}Nuova Ricetta{% endblock %}
{% block content %}
<div class="form-container largo">
    <h1>Pubblica una Ricetta</h1>
    <form method="POST">
        {{ form.hidden_tag() }}
        {% for field in form if field.widget.input_type != 'hidden' and field.name != 'submit' %}
        <div class="campo">
            {{ field.label }}{{ field() }}
            {% for errore in field.errors %}<span class="errore">{{ errore }}</span>{% endfor %}
        </div>
        {% endfor %}
        {{ form.submit(class='btn') }}
    </form>
</div>
{% endblock %}
```

---

### `app/templates/ricetta.html`
```html
{% extends 'base.html' %}
{% block title %}{{ ricetta.titolo }}{% endblock %}
{% block content %}
<article class="ricetta-dettaglio">
    <h1>{{ ricetta.titolo }}</h1>
    <div class="meta">
        <span class="badge">{{ ricetta.categoria }}</span>
        <span>{{ ricetta.difficolta }}</span>
        <span>{{ ricetta.tempo_min }} min</span>
        {% if ricetta.media_voti() %}
            <span>★ {{ ricetta.media_voti() }} ({{ ricetta.recensioni|length }} recensioni)</span>
        {% endif %}
        <span>di <a href="{{ url_for('main.profilo', user_id=ricetta.autore.id) }}">{{ ricetta.autore.username }}</a></span>
    </div>

    <p class="descrizione">{{ ricetta.descrizione }}</p>

    <section>
        <h2>Ingredienti</h2>
        <ul>
            {% for riga in ricetta.ingredienti.splitlines() %}
                {% if riga.strip() %}<li>{{ riga.strip() }}</li>{% endif %}
            {% endfor %}
        </ul>
    </section>

    <section>
        <h2>Procedimento</h2>
        <p>{{ ricetta.procedimento | replace('\n', '<br>') | safe }}</p>
    </section>

    {% if current_user.is_authenticated and current_user.id == ricetta.user_id %}
    <form method="POST" action="{{ url_for('main.elimina_ricetta', ricetta_id=ricetta.id) }}"
          onsubmit="return confirm('Eliminare questa ricetta?')">
        {{ form.hidden_tag() }}
        <button type="submit" class="btn btn-danger">Elimina Ricetta</button>
    </form>
    {% endif %}
</article>

<section class="recensioni">
    <h2>Recensioni</h2>
    {% if current_user.is_authenticated %}
    <div class="form-container">
        <h3>Lascia una Recensione</h3>
        <form method="POST">
            {{ form.hidden_tag() }}
            <div class="campo">{{ form.testo.label }}{{ form.testo(rows=3) }}</div>
            <div class="campo">{{ form.voto.label }}{{ form.voto() }}</div>
            {{ form.submit(class='btn') }}
        </form>
    </div>
    {% endif %}

    {% for recensione in ricetta.recensioni | sort(attribute='data_creazione', reverse=True) %}
    <div class="recensione-card">
        <div class="rec-header">
            <strong>{{ recensione.autore.username }}</strong>
            <span>{{ '★' * recensione.voto }}{{ '☆' * (5 - recensione.voto) }}</span>
            <small>{{ recensione.data_creazione.strftime('%d/%m/%Y') }}</small>
        </div>
        <p>{{ recensione.testo }}</p>
    </div>
    {% else %}
    <p>Ancora nessuna recensione. Sii il primo!</p>
    {% endfor %}
</section>
{% endblock %}
```

---

### `app/templates/profilo.html`
```html
{% extends 'base.html' %}
{% block title %}Profilo di {{ utente.username }}{% endblock %}
{% block content %}
<div class="profilo">
    <h1>{{ utente.username }}</h1>
    <p>Iscritto dal {{ utente.data_iscrizione.strftime('%d/%m/%Y') }}</p>
    <p><strong>{{ ricette|length }}</strong> ricette pubblicate</p>
</div>

<h2>Le sue Ricette</h2>
<div class="griglia">
    {% for ricetta in ricette %}
    <div class="card">
        <div class="card-body">
            <span class="badge">{{ ricetta.categoria }}</span>
            <h3><a href="{{ url_for('main.ricetta', ricetta_id=ricetta.id) }}">{{ ricetta.titolo }}</a></h3>
            <div class="card-meta">
                <span>{{ ricetta.difficolta }}</span>
                <span>{{ ricetta.tempo_min }} min</span>
                {% if ricetta.media_voti() %}<span>★ {{ ricetta.media_voti() }}</span>{% endif %}
            </div>
        </div>
    </div>
    {% else %}
    <p>Nessuna ricetta ancora.</p>
    {% endfor %}
</div>
{% endblock %}
```

---

### `run.py`
```python
from app import create_app

app = create_app()

if __name__ == '__main__':
    app.run()
```

---

## FASE 6 — Avviare l'App

```bash
source .venv/bin/activate
python run.py
```

Apri **`http://127.0.0.1:5000`** nel browser.

---

## Flusso Completo

```
Registrazione → Login → Pubblica Ricetta → Altri Utenti la trovano → Lasciano una Recensione → Profilo mostra le tue ricette
```

Il database `ricette.db` viene creato automaticamente nella cartella `app/` al primo avvio.
