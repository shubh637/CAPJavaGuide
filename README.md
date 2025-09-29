# 🚀 Full-Stack Development Guide: Flask & CAP Java

-----

## 🐍 Part 1: Flask Web Development Guide

### Introduction to Flask

**Flask** is a lightweight web application framework designed for quick development with the ability to scale up to complex applications. It provides simplicity and flexibility for web development.

-----

## Table of Contents (Flask)

### 1\. Setting Up Development Environment

### 2\. Creating Your First Flask App

### 3\. Blueprint Architecture

### 4\. Database Setup with SQLAlchemy

### 5\. User Authentication

### 6\. CRUD Operations

### 7\. Flash Messages

### 8\. Pagination

### 9\. RESTful APIs

### 10\. Model Commands (SQLAlchemy Queries)

-----

## Setting Up Development Environment

### Virtual Environment Setup

```bash
# Create virtual environment
python3 -m venv venv

# Activate virtual environment (Mac/Linux)
source venv/bin/activate

# Activate virtual environment (Windows)
venv\Scripts\activate
```

### Install Dependencies

```bash
# Install Flask
pip install Flask

# Install from requirements.txt
pip install -r requirements.txt
```

-----

## Creating Your First Flask App

### Basic Application Structure (`__init__.py`)

```python
# __init__.py
from flask import Flask

def create_app():
    app = Flask(__name__)
    return app
```

### Running the Application

```bash
export FLASK_APP=app_name # Replace app_name with the package/file name
flask run
```

-----

## Blueprint Architecture

### Registering Blueprints (`__init__.py`)

```python
# __init__.py
from flask import Flask
from main import main as main_blueprint
from auth import auth as auth_blueprint

def create_app():
    app = Flask(__name__)
    app.register_blueprint(main_blueprint)
    app.register_blueprint(auth_blueprint)
    return app
```

-----

## Database Setup with SQLAlchemy

### Configuration (`config.py`)

```python
# config.py
class Config:
    SQLALCHEMY_DATABASE_URI = 'sqlite:///site.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SECRET_KEY = 'dev-secret-key'
```

### User and Workout Models (`models.py`)

```python
# models.py
from flask_login import UserMixin
class User(db.Model, UserMixin):
    id = db.Column(db.Integer, primary_key=True)
    workouts = db.relationship('Workout', backref='author', lazy=True)

class Workout(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    pushups = db.Column(db.Integer, nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
```

-----

## User Authentication

### User Registration (`auth.py`)

```python
# auth.py
from werkzeug.security import generate_password_hash

@auth.route('/signup', methods=['GET', 'POST'])
def signup():
    if request.method == 'POST':
        new_user = User(
            password=generate_password_hash(password)
        )
        db.session.add(new_user)
        db.session.commit()
        return redirect(url_for('auth.login'))
    return render_template('signup.html')
```

### Protected Routes and Logout

```python
from flask_login import login_required, logout_user

@main.route('/profile')
@login_required
def profile():
    return render_template('profile.html')

@auth.route('/logout')
@login_required
def logout():
    logout_user()
    return redirect(url_for('main.index'))
```

-----

## CRUD Operations

### Create Workout

```python
from flask_login import current_user
@main.route('/workout/new', methods=['GET', 'POST'])
@login_required
def new_workout():
    if request.method == 'POST':
        workout = Workout(pushups=pushups, author=current_user)
        db.session.add(workout)
        db.session.commit()
        return redirect(url_for('main.workouts'))
    return render_template('create_workout.html')
```

-----

## Flash Messages

### Displaying Flash Messages (Jinja Template)

```html
{% with messages = get_flashed_messages() %}
  {% if messages %}
    {% for message in messages %}
      <p class="flash">{{ message }}</p>
    {% endfor %}
  {% endif %}
{% endwith %}
```

-----

## RESTful APIs

### Basic API Resources (`api.py`)

```python
from flask_restful import Resource

class HelloWorld(Resource):
    def get(self):
        return {'data': 'Hello World'}
```

-----

## Model Commands (SQLAlchemy Queries)

### Essential Query Commands

```python
# Get all
users = User.query.all()

# Filter by keyword
user = User.query.filter_by(username='john').first()

# OR condition
users = User.query.filter(
    or_(User.username == 'john', User.email == 'john@example.com')
).all()

# Count rows
total_users = User.query.count()
```

-----

-----

## ☕ Part 2: CAP Java Backend with SAP BTP Deployment

### This guide demonstrates how to build and deploy a **CAP (Cloud Application Programming Model) Java app** to **SAP BTP (Business Technology Platform)** using Cloud Foundry.

-----

## Table of Contents (CAP Java)

### 1\. Prerequisites

### 2\. Project Setup & Data Model

### 3\. Service Definition

### 4\. Run Locally

### 5\. MTA Setup, Build & Deploy

### 6\. Test the Service

-----

## Prerequisites

### Make sure you have the following installed:

| Tool | Command |
| :--- | :--- |
| **Node.js/npm** | `npm install -g @sap/cds-dk` |
| **MBT** | `npm install -g mbt` |
| **Java/Maven** | `java -version`, `mvn -version` |
| **Cloud Foundry CLI** | `cf --version` |

-----

## Project Setup & Data Model

### 1\. Initialize the project:

```bash
mkdir my-cap-app
cd my-cap-app
cds init-add java
```

### 2\. Create `db/schema.cds`:

```cds
namespace my.bookshop;

entity Books {
  key ID    : Integer;
  title     : String;
  author    : String;
  stock     : Integer;
}
```

-----

## Service Definition

### Create **`srv/cat-service.cds`** to expose data via an OData V4 service:

```cds
using my.bookshop as my from '../db/schema';

service CatalogService {
  entity Books as projection on my.Books;
}
```

-----

## Run Locally

### Use the CAP Development Kit to run the service locally with an in-memory database.

```bash
cds watch
```

### Visit `http://localhost:4004` to test OData endpoints.

-----

## MTA Setup, Build & Deploy

### MTA (Multi-Target Application) is the standard deployment format for BTP.

### MTA Setup (`mta.yaml`)

```yaml
ID: my-cap-app
_version: '1.0.0'
version: 1.0.0

modules:
  - name: my-bookshop-srv
    type: java
    path: srv
    parameters:
      memory: 1024M
    requires:
      - name: my-cap-app-db

resources:
  - name: my-cap-app-db
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared
```

### Build & Deploy

```bash
# 1. Clean and Install (Maven)
mvn clean install

# 2. Build the MTA archive (.mtar)
mbt build -p=cf -t mta_archives

# 3. Deploy to Cloud Foundry
cf deploy mta_archives/my-cap-app_1.0.0.mtar
```

-----

## Test the Service

### 1\. Check the deployed applications:

```bash
cf apps
```

### 2\. Open the application URL shown in the `cf apps` output and check the `/catalog/Books` OData endpoint.

```
```
