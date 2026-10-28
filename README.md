# Books API

REST API для книг на FastAPI. Данные хранятся в памяти, после рестарта контейнера список очищается.

В проекте настроен CI/CD в GitLab: сборка Docker-образа, тесты, публикация в Docker Hub и деплой в Docker Swarm.

## Стек

Python 3.13, FastAPI, Pydantic, Uvicorn. Тесты — pytest и allure-pytest, линт — flake8. Docker, Docker Swarm, GitLab CI.

## Запуск локально

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn src.main:app --host 0.0.0.0 --port 8000 --reload
```

Сервис будет доступен на `http://localhost:8000`, документация — на `/docs`.

В Docker:

```bash
docker build -t books-api .
docker run -p 8000:8000 books-api
```

## API

- `GET /` — проверить, что сервис работает
- `POST /books/` — создать книгу
- `GET /books/` — список книг
- `GET /books/{id}` — книга по id
- `PUT /books/{id}` — обновить книгу
- `DELETE /books/{id}` — удалить книгу

```bash
curl -X POST http://localhost:8000/books/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Clean Code", "author": "Robert Martin"}'
```

## Тесты

```bash
pytest
pytest --cov=src
pytest --alluredir=allure-results
```

## CI/CD

Пайплайн описан в `.gitlab-ci.yml`, стадии `build` → `test` → `deploy`.

- `build` — собирает образ и пушит в Docker Hub с тегом `books-api-<sha>`.
- `unit-tests` — pytest с покрытием, отчёт в артефактах.
- `lint` — flake8.
- `allure-report` — ручная джоба, генерирует HTML-отчёт Allure.
- `allure-deploy` — деплоит отчёт отдельным стеком в nginx.
- `deploy` — деплоит стек `books-api`, только для ветки `main`.

Собранные образы хранятся в Docker Hub, отдельный registry не используется.

Тесты и линт идут внутри собранного образа, поэтому проверяется ровно тот образ, который потом деплоится. Раннеру нужен Docker executor с `privileged = true`.

## Переменные CI/CD

Задаются в GitLab: Settings → CI/CD → Variables.

- `DOCKER_REGISTRY` — адрес registry, `docker.io`
- `DOCKER_REPOSITORY` — репозиторий, например `lizoksorokina/books-api`
- `DOCKER_REGISTRY_USER` — логин Docker Hub
- `DOCKER_REGISTRY_PASSWORD` — токен с правами Read & Write
- `DOCKER_AUTH_CONFIG` — JSON для авторизации раннера, чтобы тянуть приватный образ
- `DEPLOY_SERVER_ADDR` — адрес manager-ноды Swarm, например `ssh://deploy@10.0.0.5`

`DOCKER_AUTH_CONFIG` — это обычный `~/.docker/config.json`:

```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "<base64 от username:token>"
    }
  }
}
```

Base64 для `auth` получается командой:

```bash
echo -n "<username>:<token>" | base64
```

Токен создаётся в Docker Hub: Account Settings → Security → New Access Token.

## Деплой

```bash
docker -H $DEPLOY_SERVER_ADDR stack deploy \
  --with-registry-auth \
  -c docker-swarm/stacks/api.yaml \
  books-api
```

`--with-registry-auth` передаёт креды Docker Hub узлам Swarm.
