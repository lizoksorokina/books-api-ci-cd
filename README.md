# Books API - CI CD

REST API для книг на FastAPI. CI/CD: здесь собран полный путь от коммита до работающего контейнера в Docker Swarm.

Пайплайн живёт в GitLab CI, образы хранятся в Docker Hub, деплой идёт через `docker stack deploy`. 

## Идея проекта

Показать, как выглядит полный цикл доставки приложения:

- разработчик пушит коммит;
- GitLab CI собирает Docker-образ;
- тесты и линтер проверяют именно этот образ;
- образ публикуется в Docker Hub с тегом по SHA коммита;
- на ветке `main` обновляется стек в Docker Swarm;
- отчёт Allure собирается и публикуется отдельным контейнером.

## CI/CD пайплайн

Пайплайн описан в `.gitlab-ci.yml` и состоит из трёх стадий: `build` → `test` → `deploy`.

### build

Собирает образ из `Dockerfile` и пушит его в Docker Hub:

- используется `docker:28` и `docker:28-dind` (docker-in-docker), поэтому нужен раннер с `privileged = true`;
- тег — `books-api-$CI_COMMIT_SHORT_SHA`, то есть короткий SHA коммита; `latest` не используется, чтобы каждый образ был уникальным и можно было откатиться на предыдущий;
- перед сборкой выполняется `docker login` с `DOCKER_REGISTRY_USER` и `DOCKER_REGISTRY_PASSWORD`.

### test

Три джобы, которые запускаются в уже собранном образе (`image: $DOCKER_IMAGE_TAG`), а не в отдельном окружении:

- `unit-tests` — `pytest tests/unit --cov=src`, публикует JUnit-отчёт и процент покрытия в merge request;
- `lint` — `flake8 src/`;
- `allure-report` — ручная джоба: прогоняет тесты с Allure-шагами и генерирует HTML-отчёт в артефакты, хранится 1 день.

Так как тесты идут в том же образе, проверяется ровно тот артефакт, который потом поедет на сервер.

### deploy

- `allure-deploy` — берёт готовый отчёт, собирает nginx-образ, пушит его в Docker Hub и деплоит отдельным стеком `allure-report-<branch>`; контекст сборки ограничен каталогом `allure-report`, чтобы не тащить в образ весь репозиторий;
- `deploy` — запускается только для `main` и обновляет стек `books-api` на Swarm.

Деплой выполняется командой:

```bash
docker -H $DEPLOY_SERVER_ADDR stack deploy \
  --with-registry-auth \
  -c docker-swarm/stacks/api.yaml \
  books-api
```

`--with-registry-auth` передаёт креды Docker Hub узлам Swarm, поэтому на worker-нодах не нужен отдельный `docker login`.

## Переменные CI/CD

Все значения задаются в GitLab: Settings → CI/CD → Variables. В репозитории их нет.

- `DOCKER_REGISTRY` — адрес registry. Для Docker Hub это `docker.io` без `https://`, потому что значение подставляется прямо в имя образа.
- `DOCKER_REPOSITORY` — репозиторий внутри registry, например `lizoksorokina/books-api`.
- `DOCKER_REGISTRY_USER` — логин Docker Hub.
- `DOCKER_REGISTRY_PASSWORD` — пароль или personal access token. Лучше токен с правами Read & Write: его можно отозвать, не меняя пароль.
- `DOCKER_AUTH_CONFIG` — JSON с авторизацией для GitLab Runner. Нужен, чтобы раннер мог скачать приватный образ до запуска джобы.
- `DEPLOY_SERVER_ADDR` — адрес manager-ноды Swarm, например `ssh://deploy@10.0.0.5` или `tcp://10.0.0.5:2376`.


### DOCKER_AUTH_CONFIG

Это обычный `~/.docker/config.json`, только в виде переменной:

```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "<base64 от username:token>"
    }
  }
}
```

Поле `auth` — это Base64-строка `username:token`. Получить её:

```bash
echo -n "<username>:<token>" | base64
```

Токен создаётся в Docker Hub: Account Settings → Security → New Access Token, права Read & Write.


## Что используем

- **Python 3.13** и **FastAPI** — сам сервис, данные в памяти;
- **Pydantic** — схемы запросов и ответов;
- **pytest**, **pytest-cov**, **allure-pytest** — тесты и отчёты;
- **flake8** — линтер;
- **Docker 28** и docker-in-docker — сборка образов в CI;
- **Docker Hub** — хранение образов;
- **Docker Swarm** — деплой;
- **GitLab CI** — пайплайн.

## API

- `GET /` — проверка, что сервис жив;
- `POST /books/` — создать книгу;
- `GET /books/` — список книг;
- `GET /books/{id}` — книга по id;
- `PUT /books/{id}` — обновить книгу;
- `DELETE /books/{id}` — удалить книгу.

Создание книги:

```bash
curl -X POST http://localhost:8000/books/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Clean Code", "author": "Robert Martin"}'
```

## Запуск локально

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn src.main:app --host 0.0.0.0 --port 8000 --reload
```

Сервис будет на `http://localhost:8000`, Swagger — на `/docs`.

В Docker:

```bash
docker build -t books-api .
docker run -p 8000:8000 books-api
```

## Тесты

```bash
pytest
pytest --cov=src
pytest --alluredir=allure-results
```

## Структура

```text
.
├── .gitlab-ci.yml          # пайплайн
├── Dockerfile              # образ сервиса
├── docker-swarm/stacks/    # api.yaml и allure-report.yaml
├── ci/allure/              # nginx-образ для отчёта
├── src/                    # код сервиса
├── tests/                  # unit и allure тесты
└── requirements.txt
```
