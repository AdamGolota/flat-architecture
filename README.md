## Организация кода

Правила, собранные здесь изменяются и дорабатываются. Это нормально, если ранее написанный код не соответствует этим правилам. Для поддержания единообразия в проекте используется "правило бойскаута": _если вы работаете над компонентом, оставьте его чище, чем он был до вас_ (в данном случае - сделайте его ближе к текущим правилам, чем он был до вас)

### Интеракторы

Вся кодовая база поделена на _интеракторы_. Интеракторы характеризуются тем, что хорошо делают одну вещь из логики приложения. Например, удаляют объект недвижимости или извлекают все данные для страницы ЖК. И называются соответственно: secondary_market_object_deleter или building_landing_loader. На каждый интерактор выделяется отдельная директория в формате `src/<название_интерактора>/`. Зачастую интерактор является классом в отдельном файле. Название такого файла будет либо `interactor.py`, либо с префиксом `interactor_`, например `interactor_my_loader.py`

Директории интеракторов можно группировать по общему префиксу в названии. Скажем, группу интеракторов

```
src
- my_item_loader
  * interactor.py
  ...
- my_item_saver
  ...
- my_item_subitems_loader
  ...
```

можно объединить в:

```
src
- my_item
  - loader
    * interactor.py
    ...
  - saver
    ...
  - subitems_loader
    ...
```

### Разделение интерфейса

Для простоты переиспользования интеракторы принимают и возвращают только примитивные значения или чистые датаклассы. Такие датаклассы обычно определяются непосредственно перед интерактором и зависят исключительно от потребностей интерактора. Пример дальше.


### Dependency injection

Для гибкости и тестируемости интеракторы используют _внедрение зависимостей_ вместо прямого импорта инструментов. Большая часть кода пишется в классах, в конструкторы которых передаются необходимые зависимости.

В частности внедрение зависимостей используется для доступа к базе данных. Если класс пишет в базу, то он делает это через интерфейс TransactionManager из `src.mysql_alchemy.orm.transaction_manager`. Если просто читает - то через интерфейс Session из `sqlalchemy.orm.session`.


Пример чтения:

`src/my_item_loader/interactor.py`

```python
from dataclasses import dataclass

from sqlalchemy.orm.session import Session
from src.mysql_alchemy.orm.my_item import MyItem as MyItemRecord


@dataclass
class MyItem:
    my_item_field: str


class MyItemLoader:
    def __init__(self, session: Session):
        self.session = session

    def load_my_item(self) -> MyItem:
        my_item_record = self.session.query(MyItemRecord).one()
        return MyItem(
            my_item_field=my_item_record.my_item_field
        )
```


Тут my_item_record будет экземпляром записи, наследующей от declarative_base() из sqlalchemy. Сама запись не принимается и не возвращается. Вместо нее передаются экземпляры датаклассов, содержащие исключительно необходимые данные.


Функция TransactionManager.get_context() возвращает менеджер контекста, который можно использовать с ключевым словом with. При выходе из контекста реализация TransactionManager определяет, что произойдет: коммит, флаш или роллбек.


Пример записи:

`src/my_item_saver/interactor.py`

```python
from dataclasses import dataclass

from src.mysql_alchemy.orm.my_item import MyItem as MyItemRecord
from src.mysql_alchemy.orm.transaction_manager import TransactionManager


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


### Инициализация

К интерактору добавляется файл с функциями для его инициализации. Название такого файла будет либо `init.py`, либо с префиксом `init_`, например `init_my_loader.py`.

Продолжая предыдущий пример, файловая структура:

```
src
- my_item_saver
  * interactor.py
  * init.py
```

`src/my_item_saver/init.py`:

```python
from src.my_item_saver.interactor import MyItemSaver
from src.mysql_alchemy.orm.autocommit_transaction_manager import \
    AutocommitTransactionManager


def prod_init_my_item_saver():
    return MyItemSaver(
        transaction_manager=AutocommitTransactionManager()
    )
```


### Точка входа

Точкой входа могут служить fastapi эндпоинт, celery таск, rabbitmq консьюмер, команда cli, и другие. Точка входа описывается в отдельном файле рядом с интерактором. Название такого файла будет иметь префикс, совпадающий с типом точки входа, например `fastapi_` (`fastapi_route.py`) или `celery_` (`celery_task.py`).


Пример файловой структуры:

```
src
- my_item_saver
  * fastapi_route.py
  * init.py
  * interactor.py
```

#### FastAPI эндпоинт

Для тестируемости FastAPI эндпоинты используются в комбинации с библиотекой [Dependency Injector](https://python-dependency-injector.ets-labs.org/), позволяющей подменить вызываемый экземпляр интерактора.

`src/my_item_loader/fastapi_route.py`

```python
from dependency_injector.containers import DeclarativeContainer
from dependency_injector.providers import Object
from dependency_injector.wiring import Provide, inject
from fastapi import APIRouter, Depends
from src.my_item_loader.interactor import MyItemLoader
from src.rest.responses.base_api_json import JsonResponse


class MyItemLoaderContainer(DeclarativeContainer):
    my_item_loader: Object[MyItemLoader] = Object()


my_item_loader_container = MyItemLoaderContainer()
my_item_loader_router = APIRouter()


@my_item_loader_router.get('/my-item')
@inject
def get_my_item(
    my_item_loader: MyItemLoader = Depends(Provide[
        MyItemLoaderContainer.my_item_loader
    ])
):
    return JsonResponse(data=my_item_loader.load_my_item())


my_item_loader_container.wire(modules=[__name__])
```

Если несколько интеракторов объединяются в один роутер, то роутер создается в отдельном файле в директории `/src/rest/routes` и регистрирует роуты прямым вызовом декоратора:

`src/rest/routes/my_item.py`

```python
from fastapi import APIRouter
from src.my_item_loader.fastapi_endpoint import get_my_item
from src.my_item_saver.fastapi_endpoint import post_my_item

router = APIRouter(tags=['my_item'])

router.get('/my-item')(get_my_item)
router.post('/my-item')(post_my_item)
```

Для простоты поиска общие префиксы роутов не выносятся в роутер, а пишутся цельной строкой со всем роутом (`/my-item/subitems`)

В любом случае для работы в проде роутер нужно включить в соответствующее приложение. Для сайта это делается в файле `src/rest/routes/urls_site.py`, для контент панели – в `src/rest/routes/urls_cp.py`, для других приложений – аналогично.


### Тесты

Любые файлы относящиеся к тестам будут иметь префикс `test_`, например `test_unit.py` или `test_init.py`.

#### Smoke тест

Для простоты запуска другими разработчиками к интерактору добавляется один автоматический smoke тест. Он характеризуется тем, что прогоняет использование интерактора "от и до", включая, в частности, совершение запроса на fastapi (если интерактор используется в fastapi).

Smoke тест является аналогом проверки запроса из postman'а или консоли. В случае с fastapi в нем будет только одна проверка - возвращается ли 200-й статус при заданных параметрах запроса. Smoke тест может делать print тела ответа, чтобы разработчик мог визуально проверить, отдаются ли нужные данные.

Для smoke теста в файле `init` создается отдельная функция, инициализирующая интерактор для тестирования.

`src/my_item_saver/init.py`

```python
from src.my_item_saver.interactor import MyItemSaver
from src.mysql_alchemy.orm.autorollback_transaction_manager import \
    init_autorollback_transaction_manager
from src.mysql_alchemy.orm.autocommit_transaction_manager import \
    init_prod_autocommit_transaction_manager
from src.mysql_alchemy.orm.test_orm import create_ge_content_session

def prod_init_my_item_saver():
    return MyItemSaver(
        transaction_manager=init_prod_autocommit_transaction_manager()
    )

def test_init_my_item_saver():
    return MyItemSaver(
        transaction_manager=init_autorollback_transaction_manager(
            create_ge_content_session()
        )
    )
```

Тестовая инициализация интерактора может предусматривать указание базы данных конкретной страны, подключение менеджера транзакций, автоматически откатывающего любую транзакцию (для тестирования на production базе данных), и другое.

Пример теста с fastapi эндпоинтом:

`src/my_item_saver/test_smoke.py`

```python
from unittest import TestCase

from fastapi import FastAPI
from src.test_case_runner.run_test_case import run_test_case
from src.my_item_saver.fastapi_endpoint import (
    my_item_saver_container, my_item_saver_router)
from src.my_item_saver.init import test_init_my_item_saver
from starlette.testclient import TestClient

app = FastAPI()

app.include_router(my_item_loader_router)

test_web_client = TestClient(app)


class MyItemSaverSmokeTest(TestCase):
    def test_saver_does_not_fail(self):
        with self.container.my_item_saver.override(self.test_saver):
            res = test_web_client.post('/my-item')
            self.assertEqual(res.status_code, 200)

    @classmethod
    def setUpClass(cls):
        cls.container = my_item_saver_container
        cls.test_saver = test_init_my_item_saver()


if __name__ == '__main__':
    run_test_case(MyItemSaverSmokeTest)
```

В IDE можно настроить функцию "отладить файл" с указанием переменной окружения `PYTHONPATH=<путь_к_репозиторию>/site/python`, что сведет отладку интерактора к нажатию одной кнопки на открытом файле с тестом.

Таким образом базовая файловая структура интерактора будет выглядеть примерно так:

```
src
- my_item_saver
  * fastapi_endpoint.py
  * init.py
  * interactor.py
  * test_smoke.py
```

#### On Push Tests

После создания стабильного smoke теста его стоит добавить в файл on_push_tests.py в корне. Тогда тест будет запускаться через Github Actions при каждом пуше на гитхаб.


### Композиция интеракторов

Если интерактор должен использовать функциональность из другого интерактора, от также использует для этого _внедрение зависимостей_.

`src/my_item_updater/interactor.py`:

```python
class MyItemUpdater:
    def __init__(self, change_recorder: MyItemChangeRecorder):
        self.change_recorder = change_recorder
    
    def update_my_item():
        ...
        self.change_recorder.record_my_item_change(...)
```

`src/my_item_updater/init.py`:

```python
from src.my_item_change_recorder.init import \
    prod_init_my_item_change_recorder

def prod_init_my_item_updater():
    return MyItemUpdater(
        change_recorder=prod_init_my_item_change_recorder()
    )
```

### Разделение интерактора

Если интерактор начинает использовать несколько внешних зависимостей, таких как sqlalchemy, те или иные веб API или адаптации других интеракторов, то эти зависимости стоит выносить в отдельные классы. Так же как интеракторы, эти классы будут принимать и возвращать чистые датаклассы. Таким образом мы избегаем влияния внешних зависимостей на внутреннюю логику интерактора.

Каждый класс зависимости помещается в отдельный файл рядом с интерактором. Файлы с зависимостями от базы данных содержат префикс `db_` в названии. Файлы с зависимостями от внешних веб API содержат префикс `api_` в названии. От поисковой системы вроде elasticsearch - префикс `search_`. Файлы с адаптерами к другим интеракторам – префикс с названием соответствующего интерактора. Список стандартных префиксов может пополняться. Среди прочего:
 
* `schema_` - для файлов, содержащих исключительно структуры данных; если структура используется, например, только в fastapi, то название инструмента идет первым: `fastapi_schema_`.

Таким образом интерактор может выглядеть примерно так:

`src/my_item_updater/interactor.py`

```python
from src.my_item_updater.api_my_item_loader import (
    ApiMyItemLoader, LoadedMyItem)
from src.my_item_updater.db_my_item_saver import (
    DbMyItemSaver, SavedMyItem)
from src.my_item_updater.analytics_event_sender import (
    AnalyticsEventSender, MyItemUpdationEvent)


class MyItemUpdater:
    def __init__(
        self,
        api_my_item_loader: ApiMyItemLoader,
        db_my_item_saver: DbMyItemSaver,
        analytics_event_sender: AnalyticsEventSender,
    ):
        self.api_my_item_loader = api_my_item_loader
        self.db_my_item_saver = db_my_item_saver
        self.analytics_event_sender = analytics_event_sender
    
    def update_my_item(self):
        my_item = self.api_my_item_loader.get_item()
        ...
        self.db_my_item_saver(SavedMyItem(...))
        self.analytics_event_sender(MyItemUpdationEvent(...))
```
