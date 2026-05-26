# Django Relearn — Polls App

A hands-on learning project built by following the [official Django tutorial (Part 4)](https://docs.djangoproject.com/en/6.0/intro/tutorial04/).

## What this project covers

- Models with a ForeignKey relationship (`Question` → `Choice`)
- Class-based generic views (`ListView`, `DetailView`)
- Function-based view for form handling (`vote`)
- URL routing with app namespaces (`polls:index`, `polls:detail`, etc.)
- Django templates with CSRF protection, `{% url %}` tags, and filters
- Database migrations
- Django admin
- PostgreSQL with credentials via `python-decouple`
- Django REST Framework and `django-cors-headers` installed (not yet wired up)

## Project structure

```
django-relearn/
├── djangotutorialofficial/
│   ├── manage.py
│   ├── requirements.txt
│   ├── relearnofficial/        # Project config (settings, root urls)
│   │   ├── settings.py
│   │   └── urls.py
│   └── polls/                  # The polls app
│       ├── models.py           # Question and Choice models
│       ├── views.py            # IndexView, DetailView, ResultsView, vote()
│       ├── urls.py             # App-level URL patterns
│       ├── admin.py            # Admin registration
│       ├── migrations/
│       └── templates/polls/    # index.html, detail.html, results.html
└── venv/
```

## Setup

```bash
# Create and activate virtual environment
python -m venv venv
venv\Scripts\activate

# Install dependencies
pip install -r djangotutorialofficial/requirements.txt

# Create a .env file in djangotutorialofficial/ with:
# POSTGRES_DB=your_db
# POSTGRES_USER=your_user
# POSTGRES_PASSWORD=your_password
# POSTGRES_HOST=localhost
# POSTGRES_PORT=5432

# Run migrations
python djangotutorialofficial/manage.py migrate

# Start the dev server
python djangotutorialofficial/manage.py runserver
```

## Key URLs

| URL | View | Description |
|-----|------|-------------|
| `/polls/` | `IndexView` | List of latest questions |
| `/polls/<pk>/` | `DetailView` | Question detail + voting form |
| `/polls/<pk>/results/` | `ResultsView` | Vote results |
| `/polls/<pk>/vote/` | `vote()` | Handle POST vote |
| `/admin/` | Django admin | Manage questions |

## Learning goal

Be able to explain every file in this project from memory — models, views, URLs, templates, settings, and migrations — and draw the full request/response cycle on a whiteboard.

See [NOTES.md](NOTES.md) for a phase-by-phase study guide.
