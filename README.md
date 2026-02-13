# W4 Activity 2 — Database Integration Challenge

---

### What you'll build

A Flask web application that stores software projects in a SQLite database. The app will have web pages for viewing, adding, editing, and deleting projects — a complete CRUD application.

### Final project structure

By the end of this activity, your project folder should look like this:

```
flask_project/
├── app.py                  ← Main application file
├── templates/
│   ├── base.html           ← Shared layout template
│   ├── projects.html       ← List all projects (READ)
│   ├── add_project.html    ← Form to add a project (CREATE)
│   └── edit_project.html   ← Form to edit a project (UPDATE)
└── instance/
    └── data.db              ← SQLite database (auto-generated)
```

### Timing guide (~90 minutes)

| Step | Time | What you'll do |
|------|------|----------------|
| 1. Setup environment | ~5 min | Install packages, create folder, create `app.py` |
| 2. Initialise Flask app | ~10 min | Imports, app config, database connection |
| 3. Design the model | ~10 min | Define the `Project` class |
| 4. Initialise the database | ~5 min | Run the app once to create the database file |
| 5. Base template | ~10 min | Create `base.html` with shared layout |
| 6. READ route | ~15 min | Route + template to list all projects |
| 7. CREATE route | ~15 min | Route + form template to add projects |
| 8. UPDATE route | ~10 min | Route + form template to edit projects |
| 9. DELETE route | ~5 min | Route to delete a project |
| 10. Refactor to `url_for` | ~10 min | Replace hardcoded URLs with dynamic `url_for()` calls |
| 11. Testing | ~5 min | Run the app and test all operations |

---

## Step 1: Set Up Your Development Environment

Open a terminal and run the following commands. If you completed Workshop 3's virtual environment activity, you should already be comfortable with `pip install`.

### Terminal commands (macOS / Linux)

```bash
# 1. Create a project folder and navigate into it
mkdir flask_project
cd flask_project

# 2. (Optional but recommended) Create and activate a virtual environment
python3 -m venv venv
source venv/bin/activate

# 3. Install Flask and Flask-SQLAlchemy
pip install Flask Flask-SQLAlchemy

# 4. Verify the installation
pip show Flask Flask-SQLAlchemy

# 5. Create the templates folder
mkdir templates

# 6. Open the project in your IDE
```

### Terminal commands (Windows — PowerShell)

```powershell
# 1. Create a project folder and navigate into it
mkdir flask_project
cd flask_project

# 2. (Optional but recommended) Create and activate a virtual environment
python -m venv venv
.\venv\Scripts\Activate.ps1

# If you get an "execution policy" error, run this first and then try again:
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned

# 3. Install Flask and Flask-SQLAlchemy
pip install Flask Flask-SQLAlchemy

# 4. Verify the installation
pip show Flask Flask-SQLAlchemy

# 5. Create the templates folder
mkdir templates

# 6. Open the project in VS Code
code .
```

---

## Step 2: Initialise the Flask Application

Create a file called `app.py` in your project folder and start with the following code.

### Key concepts

- **`Flask(__name__)`** — creates the application object. `__name__` tells Flask where the project root is, so it can find the `templates/` folder.
- **`SQLALCHEMY_DATABASE_URI`** — the connection string. `sqlite:///data.db` means: use SQLite, store the file as `data.db`. The three slashes (`///`) indicate a relative path.
- **`SQLAlchemy(app)`** — connects the database engine to the Flask app. The `db` object is what you'll use for all database operations.

### `app.py` — starting point

```python
from flask import Flask, render_template, request, redirect
from flask_sqlalchemy import SQLAlchemy

# --- Create the Flask application ---
app = Flask(__name__)

# --- Configure the database ---
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///data.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

# --- Connect SQLAlchemy to the app ---
db = SQLAlchemy(app)
```

### What the extra imports are for

We import more from Flask than just `Flask` itself. Here's what each one does — don't worry about memorising them all now; we'll introduce each one as it comes up:

| Import | Used in | Purpose |
|--------|---------|--------|
| `render_template` | READ, CREATE, UPDATE | Renders an HTML template file and returns it as the response |
| `request` | CREATE, UPDATE | Accesses form data submitted by the user |
| `redirect` | CREATE, UPDATE, DELETE | Sends the user to a different page after an action |

---

## Step 3: Design the Database Table (Model)

### Key concepts

- A **model** is a Python class that describes the structure of a database table.
- Each **attribute** of the class becomes a **column** in the table.
- The **ORM (Object-Relational Mapper)** translates between Python objects and database rows — so you never need to write raw SQL.
- `primary_key=True` makes the `id` auto-increment — you don't set it manually.
- `nullable=False` means the field is required.

### `app.py` — add below the existing code

```python
# --- Define the Project model ---
class Project(db.Model):
    id          = db.Column(db.Integer, primary_key=True)
    name        = db.Column(db.String(100), nullable=False)
    description = db.Column(db.String(500), nullable=True)

    def __repr__(self):
        return f'<Project {self.id}: {self.name}>'
```

---

## Step 4: Initialise the Database

### Key concepts

- `db.create_all()` inspects all model classes and creates the corresponding tables in the database file. If the file doesn't exist yet, it creates that too.
- This runs **inside** `with app.app_context()` because Flask needs to know which application is active when performing database operations. In a running web server this happens automatically; here we set it up explicitly.
- We place this at the bottom of `app.py` inside the `if __name__ == '__main__':` block so it runs when the file is executed directly.

### `app.py` — add at the bottom

```python
# --- Initialise the database and run the app ---
if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(debug=True)
```

### Checkpoint — run the app for the first time

Run the app from your terminal:

```
python app.py
```

You should see output like:

```
 * Running on http://127.0.0.1:5000
```

At this point, visiting `http://127.0.0.1:5000` in a browser will show a 404 error — that's expected because we haven't defined any routes yet. But the database file (`instance/data.db`) should now exist in your project folder. You can stop the server with `Ctrl+C`.

> **Tip:** With `debug=True`, the server will automatically restart when you save changes to `app.py`. You'll only need to refresh the browser.

---

## Step 5: Create the Base Template

### Key concepts

- Flask uses **Jinja2 templates** — HTML files with special `{{ }}` and `{% %}` tags that let you insert dynamic content.
- A **base template** defines the shared structure (head, navigation, footer) so you don't repeat it in every page. Other templates **extend** this base and fill in the content block.
- `{% block content %}{% endblock %}` defines a placeholder that child templates override.

### `templates/base.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Project Manager</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 40px auto;
            padding: 0 20px;
            background-color: #f5f5f5;
        }
        h1 { color: #333; }
        nav { margin-bottom: 20px; }
        nav a {
            margin-right: 15px;
            text-decoration: none;
            color: #0066cc;
        }
        nav a:hover { text-decoration: underline; }
        table {
            width: 100%;
            border-collapse: collapse;
            background: white;
        }
        th, td {
            padding: 10px 12px;
            border: 1px solid #ddd;
            text-align: left;
        }
        th { background-color: #f0f0f0; }
        input[type="text"], textarea {
            width: 100%;
            padding: 8px;
            margin: 5px 0 15px 0;
            border: 1px solid #ccc;
            border-radius: 4px;
            box-sizing: border-box;
        }
        button, .btn {
            padding: 8px 16px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            text-decoration: none;
            font-size: 14px;
        }
        .btn-primary { background-color: #0066cc; color: white; }
        .btn-danger  { background-color: #cc3333; color: white; }
        .btn-edit    { background-color: #f0ad4e; color: white; }
        .btn:hover   { opacity: 0.85; }
    </style>
</head>
<body>
    <h1>Project Manager</h1>
    <nav>
        <a href="/">All Projects</a>
        <a href="/add">Add New Project</a>
    </nav>
    <hr>
    {% block content %}{% endblock %}
</body>
</html>
```

---

## Step 6: READ — Route and Template to List All Projects

### Key concepts

- `@app.route('/')` is a **decorator** — it tells Flask: "when someone visits this URL, run this function".
- `Project.query.all()` retrieves every row from the `project` table and returns them as a list of Python objects.
- `render_template('projects.html', projects=projects)` does two things: it loads the HTML template file, and it passes your data into it. Inside the template, the variable `projects` becomes available.
- In the template, `{% for project in projects %}` loops over the list just like a Python `for` loop.

### `app.py` — add this route above the `if __name__` block

```python
# --- READ: List all projects ---
@app.route('/')
def list_projects():
    projects = Project.query.all()
    return render_template('projects.html', projects=projects)
```

### `templates/projects.html`

```html
{% extends "base.html" %}

{% block content %}
<h2>All Projects</h2>

{% if projects %}
<table>
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Description</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
        {% for project in projects %}
        <tr>
            <td>{{ project.id }}</td>
            <td>{{ project.name }}</td>
            <td>{{ project.description or "—" }}</td>
            <td>
                <a href="/edit/{{ project.id }}" class="btn btn-edit">Edit</a>
                <a href="/delete/{{ project.id }}" class="btn btn-danger"
                   onclick="return confirm('Are you sure you want to delete this project?');">Delete</a>
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% else %}
<p>No projects yet. <a href="/add">Add one!</a></p>
{% endif %}
{% endblock %}
```

### Checkpoint

Run the app now and visit `http://127.0.0.1:5000/`. You should see the "All Projects" page with the message "No projects yet" (since the database is empty). The Edit/Delete links won't work yet — those routes come next.

---

## Step 7: CREATE — Route and Template to Add a Project

### Key concepts

- This route handles **two types of request**: a `GET` request (you navigate to the page and see the empty form) and a `POST` request (you submit the form with data).
- `methods=['GET', 'POST']` tells Flask to accept both types on this route.
- `request.method == 'POST'` lets us check which type of request came in.
- `request.form['name']` retrieves the value you typed into the form field called `name`.
- After creating the record, we use `redirect('/')` to send you back to the main page — this is the **Post/Redirect/Get** pattern, which prevents duplicate submissions if you refresh the page.

### `app.py` — add this route

```python
# --- CREATE: Add a new project ---
@app.route('/add', methods=['GET', 'POST'])
def add_project():
    if request.method == 'POST':
        name = request.form['name']
        description = request.form.get('description', '')

        new_project = Project(name=name, description=description)
        db.session.add(new_project)
        db.session.commit()

        return redirect('/')

    return render_template('add_project.html')
```

### `templates/add_project.html`

```html
{% extends "base.html" %}

{% block content %}
<h2>Add a New Project</h2>

<form method="POST">
    <label for="name">Project Name (required):</label>
    <input type="text" id="name" name="name" required>

    <label for="description">Description (optional):</label>
    <textarea id="description" name="description" rows="3"></textarea>

    <button type="submit" class="btn btn-primary">Add Project</button>
</form>
{% endblock %}
```

### Checkpoint

You should now be able to:
1. Click "Add New Project" in the navigation
2. Fill in the form and submit
3. Be redirected to the projects list, where your new project appears in the table

---

## Step 8: UPDATE — Route and Template to Edit a Project

### Key concepts

- The route uses a **URL parameter**: `/edit/<int:id>`. The `<int:id>` part is a placeholder — when someone visits `/edit/3`, Flask passes `id=3` to the function.
- `db.session.get(Project, id)` looks up the specific project. If not found, we return a 404 error.
- The template **pre-fills** the form with the project's current values using the `value` attribute in HTML (`value="{{ project.name }}"`).
- The update itself is straightforward: change the object's attributes and commit.

### `app.py` — add this route

```python
# --- UPDATE: Edit an existing project ---
@app.route('/edit/<int:id>', methods=['GET', 'POST'])
def edit_project(id):
    project = db.session.get(Project, id)

    if project is None:
        return 'Project not found', 404

    if request.method == 'POST':
        project.name = request.form['name']
        project.description = request.form.get('description', '')
        db.session.commit()

        return redirect('/')

    return render_template('edit_project.html', project=project)
```

### `templates/edit_project.html`

```html
{% extends "base.html" %}

{% block content %}
<h2>Edit Project: {{ project.name }}</h2>

<form method="POST">
    <label for="name">Project Name (required):</label>
    <input type="text" id="name" name="name" value="{{ project.name }}" required>

    <label for="description">Description (optional):</label>
    <textarea id="description" name="description" rows="3">{{ project.description or "" }}</textarea>

    <button type="submit" class="btn btn-primary">Save Changes</button>
    <a href="/" class="btn">Cancel</a>
</form>
{% endblock %}
```

---

## Step 9: DELETE — Route to Remove a Project

### Key concepts

- This route doesn't need its own template — it performs the deletion and immediately redirects back to the project list.
- We look up the project by id, delete it from the session, and commit.
- In `projects.html`, the Delete link already includes `onclick="return confirm(...)"` — this pops up a browser confirmation dialog so you don't accidentally delete a project.

### `app.py` — add this route

```python
# --- DELETE: Remove a project ---
@app.route('/delete/<int:id>')
def delete_project(id):
    project = db.session.get(Project, id)

    if project is None:
        return 'Project not found', 404

    db.session.delete(project)
    db.session.commit()

    return redirect('/')
```

---

## Step 10: Refactor — Replace Hardcoded URLs with `url_for()`

Now that all four routes are defined, we can improve our code. So far, we've been using hardcoded paths like `/`, `/add`, and `/edit/{{ project.id }}` in our templates and Python routes. This works, but it's fragile — if you ever rename a route path, you'd have to find and update every reference by hand.

Flask provides a function called `url_for()` that generates URLs dynamically from the **function name** of a route. This means your links always stay correct, even if the URL path changes.

### Key concepts

- `url_for('list_projects')` returns `'/'` — because that's the path registered to the `list_projects` function.
- `url_for('edit_project', id=3)` returns `'/edit/3'` — it fills in the URL parameter automatically.
- If you later changed `@app.route('/add')` to `@app.route('/new')`, every `url_for('add_project')` call would automatically return `'/new'` — no other code changes needed.

### Update `app.py`

First, add `url_for` to your import line at the top of `app.py`:

```python
from flask import Flask, render_template, request, redirect, url_for
```

Then replace every `redirect('/')` with `redirect(url_for('list_projects'))`. There are three places — one in each of the CREATE, UPDATE, and DELETE routes:

```python
# In add_project():
        return redirect(url_for('list_projects'))

# In edit_project():
        return redirect(url_for('list_projects'))

# In delete_project():
    return redirect(url_for('list_projects'))
```

### Update `templates/base.html`

Replace the hardcoded nav links:

```html
    <nav>
        <a href="{{ url_for('list_projects') }}">All Projects</a>
        <a href="{{ url_for('add_project') }}">Add New Project</a>
    </nav>
```

### Update `templates/projects.html`

Replace the hardcoded links in the table and the empty-state message:

```html
                <a href="{{ url_for('edit_project', id=project.id) }}" class="btn btn-edit">Edit</a>
                <a href="{{ url_for('delete_project', id=project.id) }}" class="btn btn-danger"
                   onclick="return confirm('Are you sure you want to delete this project?');">Delete</a>
```

And the "No projects yet" link:

```html
<p>No projects yet. <a href="{{ url_for('add_project') }}">Add one!</a></p>
```

### Update `templates/edit_project.html`

Replace the Cancel link:

```html
    <a href="{{ url_for('list_projects') }}" class="btn">Cancel</a>
```

### Checkpoint

Refresh the page in your browser. The app should work exactly as before — the URLs haven't changed, just the way we generate them. The benefit is that your code is now more maintainable and less error-prone.

---

## Step 11: Test Your Application

With all routes and templates in place, restart the app (or let the auto-reloader pick up the changes) and test each CRUD operation through the browser.

### Testing checklist

| Test | What to do | Expected result |
|------|------------|----------------|
| **READ** | Visit `http://127.0.0.1:5000/` | "No projects yet" message (if empty) or a table of projects |
| **CREATE** | Click "Add New Project", fill in the form, submit | Redirects to project list; new project appears in the table |
| **READ** (again) | Check the project list | The newly added project is visible with correct name and description |
| **UPDATE** | Click "Edit" next to a project, change the description, submit | Redirects to project list; description is updated |
| **DELETE** | Click "Delete" next to a project, confirm | Project disappears from the list |

---

## Complete `app.py` — Full Reference

This is the complete file for reference. You should have built this up incrementally through the steps above.

```python
from flask import Flask, render_template, request, redirect, url_for
from flask_sqlalchemy import SQLAlchemy

# --- Create the Flask application ---
app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///data.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)


# --- Define the Project model ---
class Project(db.Model):
    id          = db.Column(db.Integer, primary_key=True)
    name        = db.Column(db.String(100), nullable=False)
    description = db.Column(db.String(500), nullable=True)

    def __repr__(self):
        return f'<Project {self.id}: {self.name}>'


# --- READ: List all projects ---
@app.route('/')
def list_projects():
    projects = Project.query.all()
    return render_template('projects.html', projects=projects)


# --- CREATE: Add a new project ---
@app.route('/add', methods=['GET', 'POST'])
def add_project():
    if request.method == 'POST':
        name = request.form['name']
        description = request.form.get('description', '')

        new_project = Project(name=name, description=description)
        db.session.add(new_project)
        db.session.commit()

        return redirect(url_for('list_projects'))

    return render_template('add_project.html')


# --- UPDATE: Edit an existing project ---
@app.route('/edit/<int:id>', methods=['GET', 'POST'])
def edit_project(id):
    project = db.session.get(Project, id)

    if project is None:
        return 'Project not found', 404

    if request.method == 'POST':
        project.name = request.form['name']
        project.description = request.form.get('description', '')
        db.session.commit()

        return redirect(url_for('list_projects'))

    return render_template('edit_project.html', project=project)


# --- DELETE: Remove a project ---
@app.route('/delete/<int:id>')
def delete_project(id):
    project = db.session.get(Project, id)

    if project is None:
        return 'Project not found', 404

    db.session.delete(project)
    db.session.commit()

    return redirect(url_for('list_projects'))


# --- Initialise the database and run the app ---
if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(debug=True)
```

---

## Route Summary

| Route | Method | Function | Template | CRUD Operation |
|-------|--------|----------|----------|----------------|
| `/` | GET | `list_projects()` | `projects.html` | **Read** — display all projects |
| `/add` | GET | `add_project()` | `add_project.html` | Show the add form |
| `/add` | POST | `add_project()` | *(redirect)* | **Create** — save new project |
| `/edit/<id>` | GET | `edit_project(id)` | `edit_project.html` | Show pre-filled edit form |
| `/edit/<id>` | POST | `edit_project(id)` | *(redirect)* | **Update** — save changes |
| `/delete/<id>` | GET | `delete_project(id)` | *(redirect)* | **Delete** — remove project |
