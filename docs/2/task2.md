# ЛР 2. Потоки. Процессы. Асинхронность.
# Задание 2

<br>

Задача - спарсить даннные с нескольких сайтов и сохранить их в базу данных.

База данных взята из лабораторной работы №1 (тема - сервис для планирования совместных путешествий), а именно две таблицы:

- Stay

- Transition

Для каждой таблицы найдены по три сайта-источника данных, итого 6 ссылок:
```
URLS = {
    "planes": "https://buenos-aires-international-airport.com/flights/departures/",
    "cruises": "https://www.cruisecritic.co.uk/find-a-cruise/port-dubrovnik",
    "trains": "https://www.thetrainline.com/",
    "airbnb": "https://www.airbnb.co.uk/?tab_id=home_tab&refinement_paths%5B%5D=%2Fhomes&search_mode=flex_destinations_search&flexible_trip_lengths%5B%5D=one_week&location_search=MIN_MAP_BOUNDS&monthly_start_date=2024-06-01&monthly_length=3&monthly_end_date=2024-09-01&price_filter_input_type=2&channel=EXPLORE&search_type=category_change&price_filter_num_nights=5&category_tag=Tag%3A8225",
    "hotels": "https://www.booking.com/hotel/index.en-gb.html?aid=397594&label=gog235jc-1BCAEiBWhvdGVsKIICOOgHSDNYA2i2AYgBAZgBCbgBB8gBDdgBAegBAYgCAagCA7gC1I-YsQbAAgHSAiQ4MmE2NmQxMi02MGYwLTQzNzgtOTZjYy0zMDhhZTdiZDhkZmHYAgXgAgE&sid=bec09db0ecce614ac10826c52f468f76",
    "hostels": "https://www.booking.com/hostels/index.en-gb.html?aid=397594&label=gog235jc-1BCAEiBWhvdGVsKIICOOgHSDNYA2i2AYgBAZgBCbgBB8gBDdgBAegBAYgCAagCA7gC1I-YsQbAAgHSAiQ4MmE2NmQxMi02MGYwLTQzNzgtOTZjYy0zMDhhZTdiZDhkZmHYAgXgAgE&sid=bec09db0ecce614ac10826c52f468f76&from_booking_home_promotion=1&",
    }
```

<br>

## Базовая реализация

### Site-specific скрейпинг
*Task2/parse_basic.py*
```
def parse_planes(soup):
    elems = soup.find_all(class_="menu-item-object-custom")
    names = [r.text.split(' | ') for r in elems if '|' in r.text]
    names = [' '.join(n[1].split()[:-2]) + ', ' + n[0] for n in names]
    return [("Buenos Aires, Argentina", n, "plane") for n in names]


def parse_cruises(soup):
    elems = soup.find_all(class_="css-18jw8gc") 
    return [(e.text, "Dubrovnik, Croatia", 'ship') for e in elems if e.text.isalpha()]


def parse_trains(soup):
    elems = soup.find_all("a", class_="route-suggestions-links")
    texts = [e.text.split() for e in elems[:-5]]
    return [(x[0], x[2], 'train') for x in texts if x[2][0].isupper()]


def parse_airbnb(soup):
    elems = soup.find_all(attrs={"data-testid": "listing-card-title"})
    locations = [e.text for e in elems]
    return [(l, l, 'apartments') for l in locations]
 

def parse_hotels(soup):
    elems = soup.find_all("ul", class_="lp-hotel__explore_world_hotels")
    names = []
    locations = []
    for elem in elems:
        names.extend(elem.find_all("span", class_="bui-list__description-title"))
        locations.extend(elem.find_all("span", class_="bui-list__description-subtitle"))
    names = [n.decode_contents() for n in names]
    locations = [l.decode_contents().split() for l in locations]
    locations = [' '.join(filter(lambda x: re.match("[A-Za-z],?", x), l)) for l in locations]
    pairs = list(zip(locations, names))
    trios = [(p[0], p[1], 'hotel') for p in pairs]
    return trios  


def parse_hostels(soup):
    names = soup.find_all("a", class_="bui-card__title")
    locations = soup.find_all("p", class_="bui-card__subtitle")
    names = [n.text for n in names]
    locations = [l.decode_contents().split() for l in locations]
    locations = [' '.join(filter(lambda x: re.match("[A-Za-z],?", x), l)) for l in locations]
    pairs = list(zip(locations, names))
    trios = [(p[0], p[1], 'hostel') for p in pairs]
    return trios
```

### Базовая функция для синхронного скрейпинга
*Task2/parse_basic.py*
```
def parse_and_save(key, url):
    try:
        page = requests.get(url)
        soup = BeautifulSoup(page.text, 'html.parser')
        if key == "planes":
            data = parse_planes(soup)
            save_transitions(data)
        if key == "cruises": 
            data = parse_cruises(soup)
            save_transitions(data)
        if key == "trains": 
            data = parse_trains(soup)
            save_transitions(data)
        if key == "airbnb":
            data = parse_airbnb(soup)
            save_stays(data)
        if key == "hotels": 
            data = parse_hotels(soup)
            save_stays(data)
        if key == "hostels": 
            data = parse_hostels(soup)
            save_stays(data) 
    except Exception as e:
        print("OH NO, EXCEPTION!")
        raise e
```

### Синхронное подключение к БД
*Task2/db_conn.py*
```
engine = create_engine(db_url, echo=False)
session = Session(engine)


def save_stays(data):
    for d in data:
        statement = select(Stay).where(Stay.location == d[0],
                                       Stay.address == d[1],
                                       Stay.accomodation == d[2])          
        existing = session.exec(statement).all()
        if not existing:
            stay = StayDefault(location = d[0],
                               address = d[1],
                               accomodation = d[2])
            stay = Stay.model_validate(stay)
            session.add(stay)
            session.commit()
            session.refresh(stay)
    session.close()
    print(f"Saved {len(data)} items into the STAY database")
    

def save_transitions(data):
    for d in data:
        statement = select(Transition).where(Transition.location_from == d[0],
                                             Transition.location_to == d[1],
                                             Transition.transport == d[2])             
        existing = session.exec(statement).all()
        if not existing:
            transition = TransitionDefault(location_from = d[0],
                                        location_to = d[1],
                                        transport = d[2])
            transition = Transition.model_validate(transition)
            session.add(transition)
            session.commit()
            session.refresh(transition)
    session.close()
    print(f"Saved {len(data)} items into the TRANSITION database")
```

<br>

Чтобы можно было распараллелить выполнение, разобьем задачу на подзадачи: случайно поделим список ссылок на подмножества. 
Причем проверим несколько вариантов разбиения: 2, 3 и 6 подмножеств (то есть по 3, 2 или 1 ссылке на подзадачу).

## Многопроцессорность
*Task2/parse_multiprocess.py*
```
import random
import pandas as pd
from multiprocessing import Pool
from time import time

from parse_basic import *


def parse_chunk(chunk):
    for key, url in chunk:
        parse_and_save(key, url)


def check_n_splits(n):
    print(n, "SPLITS")
    start = time()
    keys = list(URLS.keys())
    
    with Pool(n) as p:
        random.shuffle(keys)
        chunks = [keys[i:i + len(URLS) // n] for i in range(0, len(URLS), len(URLS) // n)]
        chunks = [[(k, URLS[k]) for k in c] for c in chunks]
        p.map(parse_chunk, chunks)
    return n, time() - start
```

<br>

## Многопоточность
*Task2/parse_threads.py*
```
import random
import threading
from time import time
import pandas as pd

from parse_basic import *


def parse_chunk(chunk):
    for key, url in chunk:
        parse_and_save(key, url)


def check_n_splits(n):
    print(n, "SPLITS")
    start = time()
    keys = list(URLS.keys())
    random.shuffle(keys)
    chunks = [keys[i:i + len(URLS) // n] for i in range(0, len(URLS), len(URLS) // n)]
    chunks = [[(k, URLS[k]) for k in c] for c in chunks]
    threads = [threading.Thread(target=parse_chunk, kwargs={"chunk": chunk}) 
                                for chunk in chunks]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    return n, time() - start
```

<br>

## Асинхронность

### Асинхронный скрейпинг
*Task2/parse_async.py*
```
import asyncio
import aiohttp
import random
from time import time
import pandas as pd

from parse_basic import *
from db_conn import save_stays_async, save_transitions_async


async def parse_and_save_async(key, url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            html = await resp.text()
            soup = BeautifulSoup(html, 'html.parser')
            if key == "planes":
                data = parse_planes(soup)
                await save_transitions_async(data)
            if key == "cruises": 
                data = parse_cruises(soup)
                await save_transitions_async(data)
            if key == "trains": 
                data = parse_trains(soup)
                await save_transitions_async(data)
            if key == "airbnb":
                data = parse_airbnb(soup)
                await save_stays_async(data)
            if key == "hotels": 
                data = parse_hotels(soup)
                await save_stays_async(data)
            if key == "hostels": 
                data = parse_hostels(soup)
                await save_stays_async(data) 


async def parse_chunk(chunk):
    for key, url in chunk:
        await parse_and_save_async(key, url)


async def check_n_splits(n):
    print(n, "SPLITS")
    start = time()
    keys = list(URLS.keys())
    random.shuffle(keys)
    chunks = [keys[i:i + len(URLS) // n] for i in range(0, len(URLS), len(URLS) // n)]
    chunks = [[(k, URLS[k]) for k in c] for c in chunks]
    await asyncio.gather(*(parse_chunk(chunk) for chunk in chunks))
    return n, time() - start


if __name__ == "__main__": 
    results = {}
    for n in [2, 3, 6]:
        loop = asyncio.get_event_loop()
        splits, extime = loop.run_until_complete(check_n_splits(n))
        results[splits] = extime
    print(pd.DataFrame({"Splits": results.keys(), "Execution Time": results.values()}))
    print()
```

### Асинхронное подключение к БД
*Task2/db_conn.py*
```
async_engine = create_async_engine(db_url_async, echo=False)
async_session = sessionmaker(async_engine, class_=AsyncSession, expire_on_commit=False)


async def save_stays_async(data):
    async with async_session() as session:
        for d in data:
            statement = select(Stay).where(Stay.location == d[0],
                                        Stay.address == d[1],
                                        Stay.accomodation == d[2])          
            existing = (await session.execute(statement)).all()
            if not existing:
                stay = StayDefault(location = d[0],
                                address = d[1],
                                accomodation = d[2])
                stay = Stay.model_validate(stay)
                session.add(stay)
                await session.commit()
    print(f"Saved {len(data)} items into the STAY database")
    

async def save_transitions_async(data):
    async with async_session() as session:
        for d in data:
            statement = select(Transition).where(Transition.location_from == d[0],
                                                Transition.location_to == d[1],
                                                Transition.transport == d[2])             
            existing = (await session.execute(statement)).all()
            if not existing:
                transition = TransitionDefault(location_from = d[0],
                                            location_to = d[1],
                                            transport = d[2])
                transition = Transition.model_validate(transition)
                session.add(transition)
                await session.commit()

    print(f"Saved {len(data)} items into the TRANSITION database")
```

<br>

## Выводы

| Splits | Basic      | Multiprocessing | Threading | Async     |
|--------|------------|-----------------|-----------|-----------|
| 1      | 23.7892501 | -               | -         | -         |
| 2      | -          | 9.41433         | 4.931119  | 4.84545   |
| 3      | -          | 5.851309        | 13.978662 | 16.463624 |
| 6      | -          | 15.763105       | 3.699792  | 2.554284  |

В этом задании мы работали с I/O-bound задачей, поэтому ожидалось, что многопоточность и асинхронность должны были хорошо справиться с оптимизацией. В принципе, так и вышло. Мы еще в первом задании увидели, что многопоточность и асинхронность очень похожи по времени выполнения, и здесь видно то же самое. Особенно интересно (и непонятно), что оба подхода "проседают" на 3 подзадачах. Тем не менее, на 2 и 6 подзадачах эти подходы значительно "переигрывают" и многопроцессорный, и синхронный подходы. Лучший результат дала асинхронность с 6 подзадачами.


Стоит отметить, что в сравнении с базовой реализацией многопроцессорность также дает выигрыш во времени, однако при большом количестве процессов (здесь - 6), "менеджирование" процессов начинает забирать слишком много ресурсов, и время выполнения программы сильно увеличивается. 

В общем и целом, любой вариант распараллеливания выполнения программы (как с точки зрения подхода, так и с точки зрения количества подзадач) дает преимущество перед синхронной реализацией.