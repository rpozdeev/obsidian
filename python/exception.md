# Исключения

Cписок исключений можно посмотреть в [документации](https://docs.python.org/3/library/exceptions.html).
## Встроенные типы исключений

```python
try:
    number = int(input("Введите число: "))
    print("Введенное число:", number)
except ValueError:
    print("Преобразование прошло неудачно")
print("Завершение программы")
```

В Python есть следующие базовые типы исключений:
- *BaseException:* базовый тип для всех встроенных исключений
- *Exception:* базовый тип, который обычно применяется для создания своих типов исключений
- *ArithmeticError:* базовый тип для исключений, связанных с арифметическими операциями (OverflowError, ZeroDivisionError, FloatingPointError).
- *BufferError:* тип исключения, которое возникает при невозможности выполнить операцию с буффером
- *LookupError:* базовый тип для исключений, которое возникают при обращении в коллекциях по некорректному ключу или индексу (например, IndexError, KeyError)

Наиболее часто встречающиеся:
- *IndexError:* исключение возникает, если индекс при обращении к элементу коллекции находится вне допустимого диапазона
- *KeyError:* возникает, если в словаре отсутствует ключ, по которому происходит обращение к элементу словаря.
- *OverflowError:* возникает, если результат арифметической операции не может быть представлен текущим числовым типом (обычно типом float).
- *RecursionError:* возникает, если превышена допустимая глубина рекурсии.
- *TypeError:* возникает, если операция или функция применяется к значению недопустимого типа.
- *ValueError:* возникает, если операция или функция получают объект корректного типа с некорректным значением.
- *ZeroDivisionError:* возникает при делении на ноль.
- *NotImplementedError:* тип исключения для указания, что какие-то методы класса не реализованы
- *ModuleNotFoundError:* возникает при при невозможности найти модуль при его импорте директивой `import`
- *OSError:* тип исключений, которые генерируются при возникновении ошибок системы (например, невозможно найти файл, память диска заполнена и т.д.)

## Генерация исключений и оператор raise

```python
try:  
    age = int(input("Введите возраст: "))  
    if age > 110 or age < 1:  
        raise Exception("Некорректный возраст")  
    print("Ваш возраст:", age)  
except ValueError:  
    print("Введены некорректные данные")  
except Exception as e:  
    print(e)  
print("Завершение программы")
```

### Создание своих типов исключений

```python
class PersonAgeException(Exception):  
    def __init__(self, age, minage, maxage):  
        self.age = age  
        self.minage = minage  
        self.maxage = maxage  
  
    def __str__(self):  
        return f"Недопустимое значение: {self.age}. " \  
               f"Возраст должен быть в диапазоне от {self.minage} до {self.maxage}"  
  
class Person:  
    def __init__(self, name, age):  
        self.__name = name  # устанавливаем имя  
        minage, maxage = 1, 110  
        if minage < age < maxage:  # устанавливаем возраст, если передано корректное значение  
            self.__age = age  
        else:  # иначе генерируем исключение  
            raise PersonAgeException(age, minage, maxage)  
  
    def display_info(self):  
        print(f"Имя: {self.__name}  Возраст: {self.__age}")  
  
try:  
    tom = Person("Tom", 37)  
    tom.display_info()  # Имя: Tom  Возраст: 37  
  
    bob = Person("Bob", -23)  
    bob.display_info()  
except PersonAgeException as e:  
    print(e)  # Недопустимое значение: -23. Возраст должен быть в диапазоне от 1 до 110
```

В конструкторе PersonAgeException получаем три значения - собственное некорректное значение, которое послужило причиной исключения, а также минимальное и максимальное значения возраста.

При применении класса Person нам следует учитывать, что конструктор класса может сгенерировать исключение при передаче некорректного значения. Поэтому создание объектов Person обертывается в конструкцию try..except:

```python
try:  
    tom = Person("Tom", 37)  
    tom.display_info()  # Имя: Tom  Возраст: 37  
  
    bob = Person("Bob", -23)  # генерируется исключение типа PersonAgeException  
    bob.display_info()  
except PersonAgeException as e:  
    print(e)  # Недопустимое значение: -23. Возраст должен быть в диапазоне от 1 до 110
```

### Определение пользовательских исключений

```python
class Error(Exception):
    """Базовый класс для других исключений"""
    pass

class ValueTooSmallError(Error):
    """Вызывается, когда входное значение мало"""
    pass

class ValueTooLargeError(Error):
    """Вызывается, когда входное значение велико"""
    pass
```