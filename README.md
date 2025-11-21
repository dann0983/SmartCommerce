Este documento contém um projeto **completo** de e-commerce usando **Python + Flask + SQLite + SQLAlchemy** com:

* Autenticação (registro/login)
* CRUD de produtos (área admin)
* Carrinho persistente na sessão
* Checkout com **Stripe (exemplo)** (substitua pelas suas chaves) — e nota sobre como adaptar para MercadoPago
* Templates Jinja2 simples (HTML/CSS)

---

## Estrutura do projeto

```
loja-flask/
├── app.py
├── models.py
├── forms.py
├── config.py
├── requirements.txt
├── run.sh
├── static/
│   ├── css/
│   │   └── style.css
│   └── js/
│       └── main.js
├── templates/
│   ├── base.html
│   ├── index.html
│   ├── product.html
│   ├── cart.html
│   ├── checkout.html
│   ├── login.html
│   ├── register.html
│   └── admin/
│       ├── products.html
│       └── product_form.html
└── README.md
```

---

## requirements.txt

```
Flask==2.2.5
Flask-Login==0.6.2
Flask-WTF==1.1.1
Flask-SQLAlchemy==3.0.3
email-validator==1.3.1
stripe==5.1.0
python-dotenv==1.0.0
```

---

## config.py

```python
import os
from dotenv import load_dotenv
load_dotenv()

BASE_DIR = os.path.abspath(os.path.dirname(__file__))

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY', 'troque_esta_chave')
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL', 'sqlite:///' + os.path.join(BASE_DIR, 'app.db'))
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    STRIPE_SECRET_KEY = os.environ.get('STRIPE_SECRET_KEY')
    STRIPE_PUBLISHABLE_KEY = os.environ.get('STRIPE_PUBLISHABLE_KEY')
```

---

## models.py

```python
from flask_sqlalchemy import SQLAlchemy
from flask_login import UserMixin
from werkzeug.security import generate_password_hash, check_password_hash
from datetime import datetime

db = SQLAlchemy()

class User(db.Model, UserMixin):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(128), nullable=False)
    is_admin = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

class Product(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(120), nullable=False)
    description = db.Column(db.Text)
    price = db.Column(db.Integer, nullable=False)  # price in cents
    image = db.Column(db.String(250))
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

class Order(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=True)
    total = db.Column(db.Integer, nullable=False)
    stripe_payment_intent = db.Column(db.String(250))
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

class OrderItem(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    order_id = db.Column(db.Integer, db.ForeignKey('order.id'))
    product_id = db.Column(db.Integer, db.ForeignKey('product.id'))
    quantity = db.Column(db.Integer, nullable=False)
    price = db.Column(db.Integer, nullable=False)
```

---

## forms.py

```python
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, SubmitField, IntegerField, TextAreaField
from wtforms.validators import DataRequired, Email, Length, EqualTo

class RegisterForm(FlaskForm):
    email = StringField('Email', validators=[DataRequired(), Email()])
    password = PasswordField('Senha', validators=[DataRequired(), Length(min=6)])
    confirm = PasswordField('Confirmar senha', validators=[DataRequired(), EqualTo('password')])
    submit = SubmitField('Registrar')

class LoginForm(FlaskForm):
    email = StringField('Email', validators=[DataRequired(), Email()])
    password = PasswordField('Senha', validators=[DataRequired()])
    submit = SubmitField('Entrar')

class ProductForm(FlaskForm):
    name = StringField('Nome', validators=[DataRequired()])
    description = TextAreaField('Descrição')
    price = IntegerField('Preço (centavos)', validators=[DataRequired()])
    image = StringField('URL da imagem')
    submit = SubmitField('Salvar')
```

---

## app.py (arquivo principal)

```python
from flask import Flask, render_template, redirect, url_for, flash, request, session, jsonify
from config import Config
from models import db, User, Product, Order, OrderItem
from forms import RegisterForm, LoginForm, ProductForm
from flask_login import LoginManager, login_user, login_required, logout_user, current_user
import stripe

app = Flask(__name__)
app.config.from_object(Config)

db.init_app(app)
login_manager = LoginManager(app)
login_manager.login_view = 'login'

stripe.api_key = app.config.get('STRIPE_SECRET_KEY')

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

@app.before_first_request
def create_tables():
    db.create_all()

# Home
@app.route('/')
def index():
    products = Product.query.all()
    return render_template('index.html', products=products)

# Produto
@app.route('/product/<int:product_id>')
def product_detail(product_id):
    p = Product.query.get_or_404(product_id)
    return render_template('product.html', product=p)

# Carrinho - usa sessão
@app.route('/cart')
def view_cart():
    cart = session.get('cart', {})
    items = []
    total = 0
    for pid, qty in cart.items():
        prod = Product.query.get(int(pid))
        if prod:
            items.append({'product': prod, 'quantity': qty})
            total += prod.price * qty
    return render_template('cart.html', items=items, total=total)

@app.route('/cart/add/<int:product_id>')
def add_to_cart(product_id):
    cart = session.get('cart', {})
    cart[str(product_id)] = cart.get(str(product_id), 0) + 1
    session['cart'] = cart
    flash('Produto adicionado ao carrinho.')
    return redirect(request.referrer or url_for('index'))

@app.route('/cart/remove/<int:product_id>')
def remove_from_cart(product_id):
    cart = session.get('cart', {})
    cart.pop(str(product_id), None)
    session['cart'] = cart
    return redirect(url_for('view_cart'))

# Checkout com Stripe (cria PaymentIntent)
@app.route('/checkout', methods=['GET', 'POST'])
def checkout():
    cart = session.get('cart', {})
    if not cart:
        flash('Carrinho vazio.')
        return redirect(url_for('index'))

    items = []
    total = 0
    for pid, qty in cart.items():
        prod = Product.query.get(int(pid))
        if prod:
            items.append({'product': prod, 'quantity': qty})
            total += prod.price * qty

    if request.method == 'POST':
        # cria pedido local (status simples) e PaymentIntent
        order = Order(user_id=current_user.id if current_user.is_authenticated else None, total=total)
        db.session.add(order)
        db.session.commit()

        intent = stripe.PaymentIntent.create(
            amount=total,
            currency='brl',
            metadata={'order_id': order.id}
        )
        order.stripe_payment_intent = intent['id']
        db.session.commit()

        return render_template('checkout.html', client_secret=intent.client_secret, publishable_key=app.config.get('STRIPE_PUBLISHABLE_KEY'))

    return render_template('checkout.html', items=items, total=total, publishable_key=app.config.get('STRIPE_PUBLISHABLE_KEY'))

# Webhook (opcional) para confirmar pagamento
@app.route('/webhook', methods=['POST'])
def stripe_webhook():
    payload = request.data
    sig_header = request.headers.get('Stripe-Signature')
    # Se quiser verificar assinatura, use stripe.Webhook.construct_event
    event = None
    try:
        event = stripe.Event.construct_from(request.get_json(force=True), stripe.api_key)
    except Exception:
        pass
    # Simplificado: procurar payment_intent
    data = request.get_json()
    if data and data.get('type') == 'payment_intent.succeeded':
        pi = data['data']['object']
        metadata = pi.get('metadata', {})
        order_id = metadata.get('order_id')
        if order_id:
            order = Order.query.get(order_id)
            if order:
                # Aqui você marcaria como pago e salvaria itens
                pass
    return jsonify({'status': 'ok'})

# Auth
@app.route('/register', methods=['GET', 'POST'])
def register():
    form = RegisterForm()
    if form.validate_on_submit():
        if User.query.filter_by(email=form.email.data).first():
            flash('Email já cadastrado.')
            return redirect(url_for('register'))
        u = User(email=form.email.data)
        u.set_password(form.password.data)
        db.session.add(u)
        db.session.commit()
        flash('Conta criada. Faça login.')
        return redirect(url_for('login'))
    return render_template('register.html', form=form)

@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        u = User.query.filter_by(email=form.email.data).first()
        if u and u.check_password(form.password.data):
            login_user(u)
            flash('Bem-vindo!')
            return redirect(url_for('index'))
        flash('Email ou senha incorretos.')
    return render_template('login.html', form=form)

@app.route('/logout')
def logout():
    logout_user()
    return redirect(url_for('index'))

# Admin - listar e criar produtos (simples, sem autenticação separada além de is_admin)
@app.route('/admin/products')
@login_required
def admin_products():
    if not current_user.is_admin:
        flash('Acesso negado.')
        return redirect(url_for('index'))
    products = Product.query.all()
    return render_template('admin/products.html', products=products)

@app.route('/admin/products/new', methods=['GET', 'POST'])
@login_required
def admin_new_product():
    if not current_user.is_admin:
        flash('Acesso negado.')
        return redirect(url_for('index'))
    form = ProductForm()
    if form.validate_on_submit():
        p = Product(name=form.name.data, description=form.description.data, price=form.price.data, image=form.image.data)
        db.session.add(p)
        db.session.commit()
        flash('Produto criado.')
        return redirect(url_for('admin_products'))
    return render_template('admin/product_form.html', form=form)

if __name__ == '__main__':
    app.run(debug=True)
```

---

## templates (resumo)

### templates/base.html

```html
<!doctype html>
<html lang="pt-BR">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}Loja{% endblock %}</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
  <nav>
    <a href="{{ url_for('index') }}">Home</a>
    <a href="{{ url_for('view_cart') }}">Carrinho</a>
    {% if current_user.is_authenticated %}
      <span>{{ current_user.email }}</span>
      <a href="{{ url_for('logout') }}">Sair</a>
      {% if current_user.is_admin %}<a href="{{ url_for('admin_products') }}">Admin</a>{% endif %}
    {% else %}
      <a href="{{ url_for('login') }}">Login</a>
      <a href="{{ url_for('register') }}">Registrar</a>
    {% endif %}
  </nav>

  <main>
    {% with messages = get_flashed_messages() %}
      {% if messages %}
        <ul class="flashes">
          {% for m in messages %}
            <li>{{ m }}</li>
          {% endfor %}
        </ul>
      {% endif %}
    {% endwith %}

    {% block content %}{% endblock %}
  </main>
</body>
</html>
```

### templates/index.html

```html
{% extends 'base.html' %}
{% block content %}
<h1>Produtos</h1>
<div class="grid">
  {% for p in products %}
    <div class="card">
      <img src="{{ p.image }}" alt="{{ p.name }}">
      <h3>{{ p.name }}</h3>
      <p>{{ (p.price / 100)|round(2) }} BRL</p>
      <a href="{{ url_for('product_detail', product_id=p.id) }}">Detalhes</a>
      <a href="{{ url_for('add_to_cart', product_id=p.id) }}">Adicionar</a>
    </div>
  {% endfor %}
</div>
{% endblock %}
```

### templates/cart.html

```html
{% extends 'base.html' %}
{% block content %}
<h1>Carrinho</h1>
{% if items %}
  <ul>
  {% for it in items %}
    <li>{{ it.product.name }} x {{ it.quantity }} - {{ (it.product.price * it.quantity / 100)|round(2) }} BRL <a href="{{ url_for('remove_from_cart', product_id=it.product.id) }}">Remover</a></li>
  {% endfor %}
  </ul>
  <p>Total: {{ (total / 100)|round(2) }} BRL</p>
  <form method="post" action="{{ url_for('checkout') }}">
    <button type="submit">Ir para pagamento</button>
  </form>
{% else %}
  <p>Carrinho vazio.</p>
{% endif %}
{% endblock %}
```

### templates/checkout.html (esqueleto com Stripe)

```html
{% extends 'base.html' %}
{% block content %}
<h1>Checkout</h1>
{% if client_secret %}
  <!-- Página com Stripe Elements (simplificado) -->
  <p>Processando pagamento... (exemplo)</p>
  <script src="https://js.stripe.com/v3/"></script>
  <script>
    const stripe = Stripe('{{ publishable_key }}');
    const clientSecret = '{{ client_secret }}';
    // Aqui você montaria o Stripe Elements para capturar cartão
  </script>
{% else %}
  <p>Confirme seus dados e clique para pagar.</p>
  <form method="post">
    <button type="submit">Pagar</button>
  </form>
{% endif %}
{% endblock %}
```

---

## CSS básico: static/css/style.css

```css
body { font-family: Arial, sans-serif; margin: 20px; }
nav a { margin-right: 8px; }
.grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 12px; }
.card { background: #fff; padding: 12px; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); }
.flashes { list-style: none; padding: 0; }
```

---
