# CLAUDE.md

Guidance for Claude Code (and humans) working in this repo.

## What this is

A minimal Django starter project: the output of `django-admin startproject` (generated with
Django 4.2.6) plus the Django JET admin theme and Celery with a Redis broker. It has no apps of its
own yet. `/` redirects to `/admin/`, and the only Celery task is `printHello`, which Celery beat
sends every 10 seconds as an example of a periodic task. There is no README.

## Commands

Django and Celery commands run from `django_template/` (the folder that holds `manage.py`), so the
`django_template` package is importable.

```bash
python -m venv .venv && source .venv/bin/activate    # .venv is git-ignored
pip install -r requirements.txt
cd django_template
mkdir -p db                                # SQLite file is db/db.sqlite3; Django won't create the folder
python manage.py migrate                   # includes JET's own migrations
python manage.py createsuperuser
python manage.py runserver                 # http://127.0.0.1:8000 → /admin/ (no env vars needed)
python manage.py check                     # quickest sanity check
python manage.py check --deploy            # with the production variables set (see Configuration)

redis-server                               # broker expected at 127.0.0.1:6379, db 0
celery -A django_template worker -l info
celery -A django_template beat -l info     # sends printHello every 10 s
```

There are no tests. Redis is needed only for the worker, beat, or code that queues a task; the
admin runs without it.

The `Dockerfile` installs `requirements.txt` only and runs `python manage.py runserver
0.0.0.0:8000` in `/var/www/djangoproject` (from `ARG PROJECT`). It does not copy the code, so the
`django_template/` folder has to be mounted there, e.g.
`docker run -p 8000:8000 -v "$PWD/django_template:/var/www/djangoproject" <image>`. Pass the
settings variables with `-e NAME=value` or `--env-file .env`.

## Layout

```
requirements.txt              unpinned: Django, celery, redis, django-jet-4, psycopg2-binary, requests…
Dockerfile                    python:3.7.6 base, see Commands
django_template/
  manage.py
  django_template/
    settings.py               all configuration; security settings and broker URL from env vars
    urls.py                   / → /admin/, jet/, jet/dashboard/, admin/
    celery.py                 Celery app "django_template": CELERY_* settings, autodiscover_tasks()
    __init__.py               imports the Celery app so @shared_task binds to it
    tasks.py                  printHello example task
    wsgi.py, asgi.py          standard entry points
```

## Configuration

Everything lives in `django_template/django_template/settings.py`. Four settings come from
environment variables, read with `os.environ` (standard library only, no django-environ); with none
set, the project behaves as a local development setup and needs no configuration.
`DJANGO_SETTINGS_MODULE` defaults to `django_template.settings` in `manage.py`, `wsgi.py`,
`asgi.py` and `celery.py`.

Nothing loads a `.env` file automatically (`.env` is git-ignored). Export the variables in the
shell (single-quote the secret key: generated keys can contain `$`, `&` and `(`), or pass a
file like this one with `docker run --env-file .env`, which takes values literally: no quotes,
and comments only on their own lines.

```bash
# .env.example: production values
# true/false, 1/0 or yes/no, any case. Unset or empty = on (development).
DJANGO_DEBUG=false
# Required when DEBUG is off. Generate one with:
# python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
DJANGO_SECRET_KEY=
# Comma-separated. Unset: ['*'] while DEBUG is on, [] when it is off.
DJANGO_ALLOWED_HOSTS=example.com,www.example.com
# Unset: redis://127.0.0.1:6379/0
CELERY_BROKER_URL=redis://redis:6379/0
```

- `DEBUG` comes from `DJANGO_DEBUG` and defaults to on, so `runserver` works with no setup. Any
  value other than true/false, 1/0 or yes/no (e.g. `off`) raises `ImproperlyConfigured` rather
  than being guessed.
- `SECRET_KEY` comes from `DJANGO_SECRET_KEY`. When it is unset and DEBUG is on, settings use a
  fixed, public development key (`django-insecure-development-only-…`, which `check --deploy`
  reports as W009). When it is unset and DEBUG is off, settings raise `ImproperlyConfigured`, so
  `manage.py`, the WSGI app and Celery refuse to start. Earlier versions committed a literal key
  that is still in git history: treat it as compromised and never reuse it.
- `ALLOWED_HOSTS` comes from `DJANGO_ALLOWED_HOSTS` (comma-separated; spaces and empty entries are
  dropped). Unset, it is `['*']` while DEBUG is on and `[]` when DEBUG is off, in which case Django
  answers 400 to every request and `runserver` refuses to start.
- `DATABASES['default']` is SQLite at `BASE_DIR / 'db' / 'db.sqlite3'`, where `BASE_DIR` is the
  outer `django_template/`. `psycopg2-binary` is installed but no PostgreSQL database is configured.
- Celery reads settings with the `CELERY_` prefix (`namespace='CELERY'`): `CELERY_BROKER_URL`
  (from the environment variable of the same name, default Redis on localhost; Celery itself also
  gives that variable precedence over the setting), `CELERY_IMPORTS`, `CELERY_BEAT_SCHEDULE`. No
  result backend is set, so task results are not stored. `crontab` is imported in settings.py for
  cron-style schedule entries but not used yet.
- `STATIC_URL` is set, `STATIC_ROOT` is not: static files (JET's CSS/JS) are served only by
  `runserver` with `DEBUG` on, and `collectstatic` needs `STATIC_ROOT` first.

## Conventions / gotchas

- `jet` must stay above `django.contrib.admin` in `INSTALLED_APPS` (it overrides the admin
  templates). The package is `django-jet-4`, a Django 4 fork of django-jet, imported as `jet`; the
  original `django-jet` and `django-jet-new` are commented out in requirements.txt.
- The project package is not in `INSTALLED_APPS`, so `autodiscover_tasks()` does not find
  `django_template/tasks.py`; it is loaded through `CELERY_IMPORTS`. Tasks in a new app's
  `tasks.py` are discovered once the app is in `INSTALLED_APPS`. Add apps with
  `python manage.py startapp <name>` inside `django_template/`.
- `__init__.py` imports the Celery app, so `celery` must be installed even to run `manage.py`.
- Requirements are unpinned. A current Python installs the newest Django and Celery (in October
  2026, on Python 3.13: Django 6.1, Celery 5.6 and django-jet-4 0.0.1 pass `check`, `migrate`,
  the admin login page and a worker/beat run). The Dockerfile's Python 3.7.6 can only resolve
  Django 3.2.x and Celery 5.3.x, so the image and a local setup run different major versions.
  Pin versions if that matters.
- Inside a container `127.0.0.1:6379` is the container itself; set the `CELERY_BROKER_URL`
  environment variable to the Redis host (e.g. `redis://redis:6379/0`) when running Celery in
  Docker.
- Beat writes its schedule database to the current directory as `celerybeat-schedule` (with `-shm`
  and `-wal` files on recent Python). `.gitignore` only matches `celerybeat-schedule.*`, so these
  show up as untracked: don't commit them. `db.sqlite3`, `media/` and `.env` are ignored.
- `urls.py` uses wildcard imports from `django.urls` and `django.conf.urls` (that is where
  `path` and `include` come from).
