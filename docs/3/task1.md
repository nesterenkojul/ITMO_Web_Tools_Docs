# ЛР 3. Упаковка FastAPI приложения в Docker. Работа с источниками данных. Очереди
# Задание 1

<br>

#### Упаковка FastAPI приложения, базы данных и парсера данных в Docker

<br>

## Приложение-парсер

Создадим отдельное FastAPI-приложение для парсера.

*parser_app.py*
```
from .parse_basic import *

app = FastAPI()

@app.post("/parse")
def parse(key: str):
    try:
        if key not in URLS.keys():
            raise HTTPException(status_code=404, 
                                detail=f"Cannot find a url with key {key}")
        parse_and_save(key, URLS[key])
        return {"message": "Parsing completed. Data saved to the database."}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

<br>

## Dockerfile для основного FastAPI-приложения
*LR1/Dockerfile*
```
# база - официальный образ python
FROM python:3.8

# создание рабочей директории
WORKDIR /docker_code

# копируем файл с зависимостями в рабочую директорию
COPY ./requirements.txt /docker_code/requirements.txt

# устанавливаем зависимости
RUN pip install --no-cache-dir --upgrade -r /docker_code/requirements.txt

# копируем папку с кодом приложения в рабочую директорию
COPY ./app /docker_code/app

# запускаем приложение в рабочей директории
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

<br>

## Dockerfile для приложения-парсера
*LR3/Dockerfile*
```
# база - официальный образ python
FROM python:3.8

# создание рабочей директории
WORKDIR /docker_code

# копируем файл с зависимостями в рабочую директорию
COPY ./requirements.txt /docker_code/requirements.txt

# устанавливаем зависимости
RUN pip install --no-cache-dir --upgrade -r /docker_code/requirements.txt

# копируем папку с кодом приложения в рабочую директорию
COPY ./app /docker_code/app

# запускаем приложение в рабочей директории
CMD ["uvicorn", "app.parser_app:app", "--host", "0.0.0.0", "--port", "8001", "--reload"]
```

<br>

## Оркестрация сервисов в docker-compose

Базу данных создаем на основе образа postgres и восстнавливаем ее структуру и хранящиеся в ней данные из ЛР1.

*docker-compose.yml*
```
services:
  db:
    image: postgres
    volumes:
      - ./LR3/web_lab1.sql:/docker-entrypoint-initdb.d/web_lab1.sql
      - lab3_data:/var/lib/postgresql/data/
    environment:
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=admin
      - POSTGRES_DB=lab3
      - PGDATA=/var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
  main:
    build: ./LR1
    depends_on:
      - redis
      - db
    environment:
      - DB_ADMIN=postgresql://admin:admin@db/lab3
      - PARSER_URL=http://parser:8001/parse
    ports:
      - "8000:8000"
  parser:
    build: ./LR3
    depends_on:
      - db
    environment:
      - DB_ADMIN=postgresql://admin:admin@db/lab3
    ports:
      - "8001:8001"
volumes:
  lab3_data: 
```

<br>
