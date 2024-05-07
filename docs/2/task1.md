# ЛР 2. Потоки. Процессы. Асинхронность.
# Задание 1

<br>

Задача - расчитать сумму чисел от 1 до 1_000_000.

<br>

## Базовая реализация
*Task1/basic.py*
```
def calculate_sum(i_from, i_to):
    return sum(range(i_from, i_to))
```

<br>

Чтобы можно было распараллелить выполнение, разобьем задачу на подзадачи: будем рассчитывать суммы подмножесв промежутка. 
Причем проверим несколько вариантов разбиения: от 2 до 10 подзадач.

## Многопроцессорность
*Task1/multiprocess.py*
```
from multiprocessing import Pool


def calculate_sum(args):
    i_from, i_to = args
    return sum(range(i_from, i_to))


def check_n_splits(n):
    start = time()
    with Pool(n) as p:
        #chunks = np.array_split(range(1, 10**6), n)
        step = 10**6 // n
        chunks = [(i, i + step) for i in range(1, 10**6, step)]
        if chunks[-1][1] != 10**6:
            chunks[-1] = (chunks[-1][0], 10**6)
        result = sum(p.map(calculate_sum, chunks))
```

<br>

## Многопоточность
*Task1/threads.py*
```
import threading


result = 0
lock = threading.Lock() 


def calculate_sum(i_from, i_to):
    global result
    lock.acquire()
    result += sum(range(i_from, i_to))
    lock.release() 


def check_n_splits(n):
    start = time()
    global result
    result = 0
    step = 10**6 // n
    chunks = [(i, i + step) for i in range(1, 10**6, step)]
    if chunks[-1][1] != 10**6:
        chunks[-1] = (chunks[-1][0], 10**6)

    threads = [threading.Thread(target=calculate_sum, args=chunk) 
                                for chunk in chunks]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
```

<br>

## Асинхронность
*Task1/async.py*
```
import asyncio


result = 0

async def calculate_sum(args):
    global result
    i_from, i_to = args
    s = sum(range(i_from, i_to))
    result += s


async def main(n):
    step = 10**6 // n
    chunks = [(i, i + step) for i in range(1, 10**6, step)]
    if chunks[-1][1] != 10**6:
        chunks[-1] = (chunks[-1][0], 10**6)

    async with asyncio.TaskGroup() as tg:
        for chunk in chunks:
            tg.create_task(calculate_sum(chunk))


def check_n_splits(n):
    start = time()
    global result
    result = 0
    asyncio.run(main(n))
```

<br>

## Выводы

| Splits | Basic   | Multiprocessing | Threading | Async   |
|--------|---------|-----------------|-----------|---------|
| 1      | 0.00969 | -               | -         | -       |
| 2      | -       | 0.07265         | 0.01027   | 0.02140 |
| 3      | -       | 0.05437         | 0.00944   | 0.01603 |
| 4      | -       | 0.05546         | 0.00884   | 0.01335 |
| 5      | -       | 0.05597         | 0.00870   | 0.01001 |
| 6      | -       | 0.06143         | 0.00838   | 0.01047 |
| 7      | -       | 0.06875         | 0.00838   | 0.00948 |
| 8      | -       | 0.06896         | 0.00816   | 0.00878 |
| 9      | -       | 0.07992         | 0.00844   | 0.00864 |
| 10     | -       | 0.08809         | 0.00823   | 0.00830 |

Несмотря на то, что вычисление суммы чисел является CPU-bound задачей, в данном случае она слишком легковесна (всего миллион чисел), чтобы распараллеливать ее с помощью многопроцессорности. Здесь больше времени тратится на создание процессов и переключение между ними, чем на сами вычисления, поэтому выходит, что этот способ самый медленнный. Многопоточность и асинхронность очень похожи по времени выполнения, но не дают практически никакого выигрыша по сравнению с синхронной реализацией (ведь эти способы в первую очередь подходят для I/O-bound задач). 

Также интересно, что кол-во подзадач не критично влияет на время выполнения. В многопроцессорности абсолютно нет разницы, а в асинхронности самая большая разница между 2 и 10 подзадачами (но все равно не сильно лучше базового варианта).