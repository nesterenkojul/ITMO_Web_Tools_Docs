# ЛР 3. Упаковка FastAPI приложения в Docker. Работа с источниками данных. Очереди

# Задание 2

<br>

#### Вызов парсера из FastAPI

<br>

## Эндпоинт для вызова парсера

В основном FastAPI-приложении добавим еще один эндпойнт.

*LR1/endpoints/parser_endpoint.py*
```
parser_url = os.getenv('PARSER_URL')


parse_router = APIRouter()
auth_handler = AuthHandler()


# basic parser call
@parse_router.post("/parse", tags=['Parser'], 
                   description='One of: "planes", "cruises", "trains", "airbnb", "hotels", "hostels"')
def parse_data(key: str):
    headers = {'accept': 'application/json'}
    data = {"key": key}
    response = requests.post(parser_url, headers=headers, params=data)
    return {"status": response.status_code, "response_msg": response.json()['message']}

```

Также необходимо зарегистрировать новый роутер в приложении (в файле main.py). И еще добавить в env сервиса ссылку на api приложения-парсера. Остальные файлы остаются без изменений.

*docker-compose.yml*
```
services:
  ...
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
  ...
```

<br>
