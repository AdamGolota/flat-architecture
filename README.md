## Організація коду

Правила, зібрані тут, змінються та допрацьовуються. Це нормально, якщо код, що був написаний раніше, не відповідає цим правилам. Для підтримки одноманітності в проєкті слід використовувати "правило бойскаута": якщо ви працюєте над компонентом, залиште його чистішим, ніж він був до вас (у даному випадку - зробіть його ближче до поточних правил, ніж він був до вас)


### Інтерактори

Вся бізнес-логіка розташована у директорії `core` та поділена на _інтерактори_. Зазвичай інтерактор є одним класом. Інтерактори характеризуються тим, що добре роблять одну річ із логіки застосунку. Наприклад, видаляють об'єкт нерухомості або видобувають усі дані для сторінки ЖК. І називаються відповідно: RealtyDeleter чи BuildingLandingGetter. На кожен інтерактор виділяється окрема директорія в форматі `core/<назва_інтерактора>/`. Сам інтерактор розташовується в окремому файлі у цій директорії. Стандартно такий файл називатиметься `interactor.py`.

Інтерактори можна групувати за спільним префіксом у назві. Скажімо, групу інтеракторів

```
core
- my_item_getter
  - interactor.py
  ...
- my_item_saver
  - interactor.py
  ...
- my_item_subitems_getter
  ...
```

можна об'єднати в:

```
core
- my_item
  - getter
    - interactor.py
    ...
  - saver
    - interactor.py
    ...
  - subitems_getter
    ...
```

### Розділення інтерфейсу


Для спрощення повторного використання інтерактори приймають і повертають лише примітивні значення або чисті датакласи. Такі датакласи зазвичай визначаються безпосередньо перед інтерактором і залежать виключно від потреб інтерактора. Приклад далі.


### Dependency injection

Для гнучкості та тестовності інтерактори використовують _впровадження залежностей_ замість прямого імпорту інструментів. Більша частина коду пишеться в класах, а необхідні залежності передаються в їх конструктори.

Зокрема впровадження залежностей використовується для доступу до бази даних. Якщо клас пише в базу, то він робить це через інтерфейс TransactionManager з `mysql_alchemy.orm.transaction_manager`. Якщо просто читає - то через інтерфейс Session з `sqlalchemy.orm.session`.


Приклад читання:

`core/my_item_getter/interactor.py`

```python
from dataclasses import dataclass

from sqlalchemy.orm.session import Session
from mysql_alchemy.orm import MyItem as MyItemRecord


@dataclass
class MyItem:
    my_item_field: str


class MyItemGetter:
    def __init__(self, session: Session):
        self.session = session

    def load_my_item(self) -> MyItem:
        my_item_record = self.session.query(MyItemRecord).one()
        return MyItem(
            my_item_field=my_item_record.my_item_field
        )
```


Тут my_item_record буде екземпляром запису, що спадкує від declarative_base() із sqlalchemy. Сам запис не приймається і не повертається. Замість нього передаються екземпляри датакласів, що містять лише необхідні дані.


Функція TransactionManager.get_context() повертає менеджер контексту, який можна використовувати з ключовим словом with. При виході з контексту реалізація TransactionManager визначає, що станеться: коміт, флаш або роллбек.


Приклад запису:

`core/my_item_saver/interactor.py`

```python
from dataclasses import dataclass

from mysql_alchemy.orm import MyItem as MyItemRecord
from mysql_alchemy.orm import TransactionManager


@dataclass
class MyItem:
    my_item_field: str


class MyItemSaver:
    def __init__(self, transaction_manager: TransactionManager):
        self.transaction_manager = transaction_manager

    def save_my_item(self, my_item: MyItem):
        with self.transaction_manager.get_context() as session:
            my_item_record = MyItemRecord()
            my_item_record.my_item_field = my_item.my_item_field
            session.add(my_item_record)
```


### Ініціалізація

До інтерактора додається файл із функціями для його ініціалізації. Назва такого файлу буде або `init.py`, або з префіксом `init_`, наприклад `init_my_getter.py`.

Продовжуючи попередній приклад, файлова структура:

```
core
- my_item_saver
  * interactor.py
  * init.py
```

`core/my_item_saver/init.py`

```python
from fastapi import Depends

from tools.action_context.init_site import site_init_action_context
from tools.action_context.schema import ActionContext
from core.my_item_saver.interactor import MyItemSaver


def site_init_my_item_saver(
    action_context: ActionContext = Depends(site_init_action_context)
):
    return MyItemSaver(
        transaction_manager=action_context.content_db_transaction_manager
    )
```


### Точка входу

Точкою входу може бути fastapi ендпоінт, celery таск, rabbitmq конс'юмер, команда cli, і інші. Точка входу описується в окремому файлі поряд з інтерактором. Назва такого файлу матиме префікс, що співпадає з типом точки входу, наприклад `fastapi_` (`fastapi_route.py`) або `celery_` (`celery_task.py`).

Приклад файлової структури:

```
core
- my_item_saver
  * fastapi_route.py
  * init.py
  * interactor.py
```

#### FastAPI ендпоінт

Для тестовності інтерактор не створюється безпосередньо у функції-роуті, а підключається через об'єкт Depends із FastAPI. Це дозволяє в подальших тестах підміняти функцію ініціалізації інтерактора. Також для простоти тестування й підключення кожен роут має свій окремий роутер.

`core/my_item_getter/fastapi_route.py`

```python
from fastapi import APIRouter, Depends
from fastapi.responses import JSONResponse
from core.my_item_getter.init import prod_init_my_item_getter
from core.my_item_getter.interactor import MyItemGetter

my_item_getter_router = APIRouter()


@my_item_getter_router.get('/my-item')
def get_my_item(
    my_item_getter: MyItemGetter = Depends(
        prod_init_my_item_getter
    )
):
    return JSONResponse(my_item_getter.load_my_item())

```

Якщо декілька інтеракторів об'єднуються в один роутер, то роутер створюється в окремому файлі в директорії `/routes/*/` і реєструє підроутери викликом `include_router`:

`routes/site/my_item.py`

```python
from fastapi import APIRouter
from core.my_item_getter.fastapi_endpoint import my_item_getter_router
from core.my_item_saver.fastapi_endpoint import my_item_saver_router

router = APIRouter(tags=['my_item'])

router.include_router(my_item_getter_router)
router.include_router(my_item_saver_router)
```

Для спрощення пошуку спільні префікси роутів не винесені в роутер, а пишуться цілим рядком з усім роутом (`/my-item/subitems`)

В будь-якому випадку для роботи в проді роутер потрібно включити в відповідний застосунок. Для сайту це робиться в файлі `routes/site/urls.py`, для контент панелі - в `routes/cp/urls.py`, для мобільного api - в `routes/mobi/urls.py`.

Ось приклад такого файлу:

```python
from fastapi import FastAPI

from core.my_item_getter.fastapi_route import my_item_getter_router
from core.my_item_saver.fastapi_route import my_item_saver_router


def register_routers(app: FastAPI) -> None:
    app.include_router(my_item_getter_router)
    app.include_router(my_item_saver_router)
```

Функція register_routers пізніше викликатиметься безпосередньо з app.py чи іншої точки входу в застосунок.


### Тести

Будь-які файли, що відносяться до тестів, будуть мати префікс `test_`, наприклад `test_unit.py` або `test_init.py`.

#### Smoke тест

Для спрощення запуску іншими розробниками до інтерактора додається один автоматичний smoke тест. Він характеризується тим, що проганяє використання інтерактора "від і до", включаючи, зокрема, здійснення запиту на fastapi (якщо інтерактор використовується в fastapi) чи вилкик celery задачі.

Smoke тест є аналогом перевірки запиту з postman'а чи консолі. У випадку з fastapi в ньому буде тільки одна перевірка - чи повертається 200-й статус при заданих параметрах запиту.

Приклад тесту з fastapi ендпоінтом:

`core/my_item_saver/test_smoke.py`

```python
from unittest import TestCase

from fastapi import FastAPI

from tools.action_context.init_site import site_init_action_context
from tools.action_context.test_init import test_init_action_context
from tools.test_case_runner.run_test_case import run_test_case
from core.my_item_saver.fastapi_endpoint import my_item_saver_router
from starlette.testclient import TestClient


class MyItemSaverSmokeTest(TestCase):
    def test_saver_does_not_fail(self):
        res = self.test_web_client.post('/my-item')
        self.assertEqual(res.status_code, 200)

    @classmethod
    def setUpClass(cls):
        app = FastAPI()
        app.include_router(my_item_saver_router)
        app.dependency_overrides[
            site_init_action_context
        ] = test_init_action_context
        cls.test_web_client = TestClient(app)


if __name__ == '__main__':
    run_test_case(MyItemSaverSmokeTest)
```

В IDE можна налаштувати функцію "налагодити файл" з указанням змінної середовища `PYTHONPATH=<шлях_до_репозиторію>/site/python`, що зведе відлагодження інтерактора до натискання однієї кнопки на відкритому файлі з тестом.

Таким чином базова файлова структура інтерактора буде виглядати приблизно так:

```
core
- my_item_saver
  * fastapi_endpoint.py
  * init.py
  * interactor.py
  * test_smoke.py
```

#### On Push Tests

Після створення стабільного smoke тесту його варто додати до файлу `on_push_tests.py` в корені. Цей файл буде запускатися через Github Actions при кожному пуші в PR на гітхаб.

Ось приклад такого файлу:

```python
import sys
import unittest

from core.my_item_saver.test_smoke import MyItemSaverSmokeTest
from core.my_item_getter.test_smoke import MyItemGetterSmokeTest

runner = unittest.TextTestRunner(verbosity=2)
result = runner.run(unittest.TestSuite([
    unittest.makeSuite(MyItemSaverSmokeTest),
    unittest.makeSuite(MyItemGetterSmokeTest),
]))

sys.exit(0 if result.wasSuccessful() else 1)
```

### Композиція інтеракторів

Якщо інтерактор використовує інший інтерактор, то він повинен використовувати його через _впровадження залежностей_.

`core/my_item_updater/interactor.py`:

```python
class MyItemUpdater:
    def __init__(self, change_recorder: MyItemChangeRecorder):
        self.change_recorder = change_recorder
    
    def update_my_item():
        ...
        self.change_recorder.record_my_item_change(...)
```

`core/my_item_updater/init.py`:

```python
from core.my_item_change_recorder.init import \
    prod_init_my_item_change_recorder

def prod_init_my_item_updater(action_context):
    return MyItemUpdater(
        change_recorder=prod_init_my_item_change_recorder(action_context)
    )
```

### Розділення інтеракторів

Якщо інтерактор починає використовувати декілька зовнішніх залежностей, таких як sqlalchemy, ті чи інші веб API або складні адаптації інших інтеракторів, то ці залежності варто винести в окремі класи. Так само як інтерактори, ці класи будуть приймати і повертати чисті структури даних. Таким чином ми уникнемо впливу зовнішніх залежностей на внутрішню логіку інтерактора.

Кожен клас залежності кладеться в окремий файл поруч із інтерактором. Файли із залежностями від бази даних містять префікс `db_` в назві. Файли із залежностями від зовнішніх веб API містять префікс `api_` в назві. Від пошукової системи типу elasticsearch - префікс `search_`. Файли з адаптерами до інших інтеракторів - префікс із назвою відповідного інтерактора. Список стандартних префіксів може поповнюватися. Серед іншого:
* `schema` – для файлів, що містять виключно структури даних; якщо структура використовується, наприклад, лише у fastapi, то назва інструменту йде першою: `fastapi_schema_`;
* `fn` – для файлів, що містять чисті функції.

Таким чином інтерактор може виглядати приблизно так:

```

`core/my_item_updater/interactor.py`

```python
from core.my_item_updater.api_my_item_getter import (
    ApiMyItemGetter, LoadedMyItem)
from core.my_item_updater.db_my_item_saver import (
    DbMyItemSaver, SavedMyItem)
from core.my_item_updater.analytics_event_sender import (
    AnalyticsEventSender, MyItemUpdationEvent)


class MyItemUpdater:
    def __init__(
        self,
        api_my_item_getter: ApiMyItemGetter,
        db_my_item_saver: DbMyItemSaver,
        analytics_event_sender: AnalyticsEventSender,
    ):
        self.api_my_item_getter = api_my_item_getter
        self.db_my_item_saver = db_my_item_saver
        self.analytics_event_sender = analytics_event_sender
    
    def update_my_item(self):
        my_item = self.api_my_item_getter.get_item()
        ...
        self.db_my_item_saver(SavedMyItem(...))
        self.analytics_event_sender(MyItemUpdationEvent(...))
```
