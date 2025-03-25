#pytest
# Pytest
## Команды

- `pytest -v` - вывод с детализацией
- `pytest -q` - уменьшить вывод детализации
- `pytest -l` - показать значение локальных переменных при ошибке
- `pytest -s` - --capture=no показывать print
- `pytest --collect-only` - собрать данные о тестах, не выполнять их
- `pytest -k "name"` - фильтрация тестов по имени
- `pytest -m "mark"` - фильтрация тестов по меткам
- `pytest -x` - завершить выполнение тестов после первого сбоя
- `pytest --maxfail=N` - завершить выполнение тестов после N сбоев
- `pytest --lf` - повторно запустить тесты, которые прошли в последнем запуске
- `pytest --ff` - сначала повторно запустить тесты, которые прошли в последнем запуске, затем остальные
- `pytest --durations=N` - отобразить время выполнения тестов, N - кол-во самых медленных тестов, которые нужно увидеть в отчете
- `pytest <имя теста>` - запустить конкретный тест 
	- tests/file.py::test_t - определенный тест в файле
	- tests/file.py::test_t[1,2,0.5] - определенный тест в файле c параметром
	- tests/file.py::TestEx - группу тестов
## Параметризация теста

```python
@pytest.mark.parametrize(
	"x, y, res", # передаваемые параметры  
	(1, 2, 0.5), # значения параметров
	(5, -1, -5)
)
def test_t(x, y, res):  
    assert x / y == res
```

## Группировка тестов

```python
class TestEx:  
    @pytest.mark.parametrize("x, y, res", (1, 2, 0.5), (5, -1, -5))
    def test_t(self, x, y, res):  
        assert x / y == res  
  
    @pytest.mark.parametrize("x, y, res", (1, 2, 3), (5, -1, 4))
    def test_s(self, x, y, res):  
        assert x + y == res
```

## Проверка исключений

```python
def test_ex():  
    with pytest.raises(TypeError):  
        assert 1 / 0
```

```python
@pytest.mark.parametrize(  
    "x, y, res, expectation",  
    (1, 2, 0.5, does_not_raise()),  
    (5, 0, -5, pytest.raises(ZeroDivisionError),),  
)
def test_t(self, x, y, res, expectation):  
    with expectation:  
        assert x / y == res
```

## Fixture

```python
@pytest.fixture
def candis():
	candies = [
		Candy(title='a'),
		Candy(title='b')
	]
	return candies

@pytest.fixture
def empty_candies():
	CandiesService.delete_all()

def test_cout_candis(candies, empty_candies):
	# candies = [
	#	 Candy(title='a'),
	#	 Candy(title='b')
	# ]
	# CandiesService.delete_all()
	for candy in candies:
		cansy.save()

	assert CandiesService.count == 2

@pytest.mark.usefixtures("empty_candies")
class TestT:
	def test_b(self, candies):
		# candies = [
		#	 Candy(title='a'),
		#	 Candy(title='b')
		# ]
		# CandiesService.delete_all()
		for candy in candies:
			cansy.save()

		assert CandiesService.count == 2
```

### Область (scope) применения
Определяет время вызова фикстуры

- class
- function (default)
- module
- package
- session

`autouse=True` - автоматически использовать

```python
@pytest.fixture(scope="session", autouse=True)
def setup_db():
	Base.metadata.create_all(engine)
	yild
	Base.metadata.drop_all(engine)
	
```

## .env

```bash 
pip install pytest-dotenv
```

```yaml
[pytest]
env_files = 
	.env
	.test.env
```

```bash
pytest --envfile /tests/.env
```

## conftest.py

Хранит fixture для всего проекта. Поднятие инфраструктуры или данные, которые используют все тесты.

## Файлы с тестовыми данными

Когда путь до файла меняется в зависимости от места запуска pytest, это обычно связано с тем, что текущая рабочая директория (cwd) отличается от того места, где расположен файл с тестовыми данными. Это может привести к ошибке, когда файл не найден.

- **os.path.abspath()** позволяет получить абсолютный путь относительно текущей директории

```python
import os

def get_file_path(filename: str) -> str:
    # Получаем абсолютный путь до файла относительно текущего каталога
    return os.path.join(os.path.dirname(os.path.abspath(__file__)), filename)

# Пример использования
file_path = get_file_path('data/team.json')
print(file_path)  # Путь до файла будет правильным независимо от текущей директории
```

- **pathlib.Path** — более современный и удобный способ работы с путями

```python
from pathlib import Path

def get_file_path(filename: str) -> Path:
    # Получаем путь до файла относительно текущего местоположения теста
    return Path(__file__).parent / filename

# Пример использования
file_path = get_file_path('data/team.json')
print(file_path)  # Путь до файла будет правильным независимо от текущей директории
```