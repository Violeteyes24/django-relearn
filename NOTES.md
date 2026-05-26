# Study Notes — Django Polls App

Practice loop for every phase: **Read → Predict → Experiment (break it) → Fix → Explain out loud**

---

## Phase 1 — Models + Migration

**Files:** `polls/models.py`, `polls/migrations/0001_initial.py`, `settings.py` (DATABASES block)

**What to understand:**
- A Python class becomes a DB table. A field becomes a column. The migration is the proof — read `0001_initial.py` and map every line back to `models.py`.
- `was_published_recently()` is a Python method, not a column. It runs in Python memory, not SQL.
- `timezone.now()` instead of `datetime.now()` because `USE_TZ = True` — Django stores UTC, converts on display.
- `ForeignKey(Question, on_delete=CASCADE)` — deleting a Question deletes all its Choices in the DB.
- `python-decouple`'s `config()` reads from `.env`. If the key is missing and there is no default, the app crashes at startup.

**Experiments:**
1. `python manage.py shell` → create a `Question`, call `.was_published_recently()`, then set `pub_date` to 2 days ago and call it again.
2. `Question.objects.filter(pub_date__lte=timezone.now())` — append `.query` and print it to see the SQL.
3. Add a field to `Question` (e.g. `author = models.CharField(max_length=100, default="anon")`), run `makemigrations`, read the generated file, then revert.

**Questions to answer without looking:**
- What SQL table name does Django create for `Question` and why?
- What does `on_delete=CASCADE` do at the database level?
- What would happen if you deleted `0001_initial.py` and ran `migrate` on a fresh database?
- Why does the migration list `dependencies = []`?

---

## Phase 2 — URL Routing

**Files:** `polls/urls.py`, `relearnofficial/urls.py`

**What to understand:**
- The project-level `urls.py` uses `include()` to delegate `/polls/` to the app — this is Django's isolation pattern.
- `app_name = "polls"` defines the namespace. Without it, `polls:detail` breaks.
- `<int:pk>` is special — CBVs look for `pk` automatically. `<int:question_id>` must match the FBV parameter name exactly.
- `reverse("polls:results", args=(question.id,))` generates the URL string at runtime.

**Experiments:**
1. Comment out `app_name = "polls"` — start server, read the error.
2. Rename `<int:pk>` to `<int:question_id>` in the detail URL — observe what CBV breaks and why.
3. Add `path("latest/", views.IndexView.as_view(), name="latest")` — visit it, then revert.

**Questions to answer without looking:**
- What two things does Django need to resolve `polls:detail` in a template?
- Why does `DetailView` use `pk` but `vote()` uses `question_id`?
- What is the full path a request to `/polls/3/vote/` takes before it reaches the view?

---

## Phase 3 — Views

**File:** `polls/views.py`

**What to understand:**
- CBVs (`IndexView`, `DetailView`, `ResultsView`) inherit behavior from generics. Most logic is in the parent class.
- `context_object_name = "latest_question_list"` sets the template variable name. Default would be `object_list`.
- `F("votes") + 1` — the increment happens in SQL (`SET votes = votes + 1`), not in Python. This prevents a race condition where two simultaneous votes both read `5`, both write `6`, and one vote is lost.
- `HttpResponseRedirect(reverse(...))` after a POST = the Post/Redirect/Get pattern. Prevents duplicate submissions on back/refresh.
- `get_object_or_404` = `objects.get()` but returns a 404 page instead of crashing.

**Experiments:**
1. Replace `F("votes") + 1` with `selected_choice.votes + 1`, vote from two browser tabs at the same time, check the count — demonstrates the race condition. Revert.
2. Rewrite `DetailView` as a plain function-based view. You'll need `get_object_or_404` and `render`. Swap it into `urls.py`.
3. Add `get_context_data` to `DetailView` to inject an extra variable (e.g. `{"site_name": "My Polls"}`). Display it in `detail.html`.

**Questions to answer without looking:**
- What does `generic.ListView` do that you would have to write yourself in an FBV?
- Why does `ResultsView` need an explicit `template_name` if `DetailView` already has one?
- Where exactly does the `question` variable in `detail.html` come from?

---

## Phase 4 — Templates

**Files:** `polls/templates/polls/index.html`, `detail.html`, `results.html`

**What to understand:**
- `{% url 'polls:detail' question.id %}` generates the URL at render time. If you hardcode `/polls/3/` and rename the URL pattern, it silently breaks.
- `{% csrf_token %}` renders a hidden `<input>` field. `CsrfViewMiddleware` validates the token on every POST. Without it, votes are rejected.
- In templates, `question.choice_set.all` has no parentheses — Django calls callables automatically.
- `{{ choice.votes|pluralize }}` returns `"s"` when votes ≠ 1, `""` when votes = 1.
- There is no `{% extends %}` here — no base template yet. Creating one is a natural next step.

**Experiments:**
1. Hardcode a URL in `index.html`, change the URL pattern in `urls.py` — the template breaks. `{% url %}` would have adapted. Revert.
2. Remove `{% csrf_token %}` from `detail.html`, submit a vote, read the 403 error. Restore it.
3. Create `polls/templates/polls/base.html` with `{% block content %}{% endblock %}` and make all three templates extend it.
4. Add `{{ forloop.counter }}` next to each choice in the `detail.html` loop. Explore `forloop.first` and `forloop.last`.

**Questions to answer without looking:**
- What does `{% csrf_token %}` actually render in the HTML source? (View source in the browser.)
- What is the N+1 query problem and where could it appear in these templates?
- What is the difference between `question.choice_set.all` in a template and `.all()` in Python?

---

## Phase 5 — Settings, Middleware, Request/Response Cycle

**Files:** `relearnofficial/settings.py` (full), `relearnofficial/urls.py`, `polls/apps.py`

**What to understand:**
- Startup order: `runserver` → reads `DJANGO_SETTINGS_MODULE` → imports all `INSTALLED_APPS` → registers models.
- `MIDDLEWARE` wraps every request top-to-bottom going in, bottom-to-top going out. `CsrfViewMiddleware` is in this list — that's where CSRF checking happens.
- `ROOT_URLCONF` is where every request starts URL resolution — always `relearnofficial/urls.py`.
- `APP_DIRS: True` tells Django to look inside each app's `templates/` folder. The inner `polls/` folder namespaces templates so two apps can both have `index.html` without conflict.
- `USE_TZ = True` + `TIME_ZONE = 'Asia/Manila'` — DB stores UTC, Django converts to Manila time on display.

**Experiments:**
1. Remove `CsrfViewMiddleware` from `MIDDLEWARE`, vote without a CSRF token — it works. Restore it.
2. Remove `'polls.apps.PollsConfig'` from `INSTALLED_APPS`, run `migrate`, visit `/polls/` — read both errors.
3. Set `APP_DIRS: False`, visit `/polls/` — read the template-not-found error. Restore.
4. Activate `django-cors-headers` (needed for Next.js): add `'corsheaders'` to `INSTALLED_APPS`, add `'corsheaders.middleware.CorsMiddleware'` at the top of `MIDDLEWARE`, add `CORS_ALLOW_ALL_ORIGINS = True`.

**Questions to answer without looking:**
- What is the order of steps from `runserver` starting to the first view being called?
- Why do templates live at `polls/templates/polls/` and not `polls/templates/`?
- What does `django-cors-headers` solve, and why will you need it for Next.js?

---

## Phase 6 — Django REST Framework

**DRF is already installed (`djangorestframework` in `requirements.txt`). Now wire it up.**

**Files to create:** `polls/serializers.py`
**Files to modify:** `polls/views.py`, `polls/urls.py`, `settings.py`

**What to understand:**
- Serializer = converts a model instance → Python dict → JSON. Parallel to Forms + Templates, but for data only.
- `ModelSerializer` auto-generates fields from the model definition.
- DRF views return `Response(serialized_data)` instead of `render(request, template)`.
- Visiting a DRF endpoint in a browser shows a browsable API interface — useful for manual testing.

**Steps:**
1. Add `'rest_framework'` to `INSTALLED_APPS`.
2. Create `polls/serializers.py` — write `QuestionSerializer` and `ChoiceSerializer` using `ModelSerializer`.
3. Add `QuestionListAPIView` (`generics.ListAPIView`) and `QuestionDetailAPIView` (`generics.RetrieveAPIView`) to `views.py`.
4. Add URL patterns: `polls/api/questions/` and `polls/api/questions/<int:pk>/`.
5. Visit `localhost:8000/polls/api/questions/` — confirm the browsable API renders.
6. Fetch with `Accept: application/json` header — confirm you get raw JSON.

**Questions to answer without looking:**
- What is a Serializer and why is it needed?
- What is the difference between `generic.DetailView` (Django) and `generics.RetrieveAPIView` (DRF)?
- What happens if you return `Response(question_instance)` without serializing first?

---

## Phase 7 — Connect to Next.js

**Prereqs:** Phase 5 (CORS active), Phase 6 (API endpoints working)

**What to understand:**
- Django = data provider (JSON). Next.js = UI renderer.
- `getServerSideProps` runs on the Node server per request. `getStaticProps` runs at build time. Client-side `fetch` runs in the browser. All three can call Django.
- CORS: the browser blocks cross-origin responses unless Django sends the right `Access-Control-Allow-Origin` header — that's what `django-cors-headers` adds.
- Django's default CSRF protection assumes requests come from Django-rendered forms. Next.js POSTs cross-origin, so you need DRF token auth or `CSRF_TRUSTED_ORIGINS`.

**Steps:**
1. `npx create-next-app@latest polls-frontend` outside the Django folder.
2. Fetch `http://localhost:8000/polls/api/questions/` from `getServerSideProps`. Log the result.
3. Set `CORS_ALLOWED_ORIGINS = ["http://localhost:3000"]`. Confirm it works. Change to a wrong port — confirm the CORS error.
4. Build a polls list page and a detail page in Next.js rendering Django data.
5. Implement the vote POST from Next.js — handle CSRF or switch to DRF token auth.

**Questions to answer without looking:**
- What is the difference between SSR and client-side fetching in Next.js?
- Why does Django's default CSRF protection break for Next.js requests?
- If you added a new field to `QuestionSerializer`, what changes are required in Next.js?

---

## Progression Summary

```
Phase 1  models.py + migration      →  Python maps to PostgreSQL
Phase 2  urls.py                    →  Requests are routed
Phase 3  views.py                   →  Request handling, CBV vs FBV
Phase 4  templates/                 →  Server-side rendering
Phase 5  settings.py + middleware   →  Django init and request/response cycle
Phase 6  DRF (new serializers.py)   →  Serve JSON instead of HTML
Phase 7  Next.js                    →  Full-stack integration
```

**The finish line:** Without looking at any code, draw on a whiteboard the full path of:
1. A browser request to `GET /polls/`
2. A Next.js `fetch` to `GET /polls/api/questions/`

Both paths, start to finish, through middleware → URL resolution → view → template/serializer → response.

When you can draw both, you are a full-stack developer.
