Модуль **`argparse`** — це стандартна бібліотека Python для розбору (парсингу) аргументів командного рядка.

  

Це фундаментальний інструмент для DevOps-інженерів та розробників CLI-утиліт (Command Line Interface), оскільки він дозволяє перетворити звичайний Python-скрипт на повноцінну консольну команду з довідкою (`--help`), прапорцями, позиційними параметрами та перевіркою типів.

  

# 📖 Повна документація з `argparse`

## 1. Навіщо потрібен `argparse`?

Коли ти запускаєш скрипт з терміналу, ти часто передаєш йому параметри:

  

Bash

```
python main.py server.log --output report.txt --verbose
```

Без `argparse` тобі довелося б вручну розбирати список `sys.argv`:

  

Python

```
import sys
# sys.argv = ['main.py', 'server.log', '--output', 'report.txt', '--verbose']
```

Це незручно, крихко і вимагає написання десятків строк перевірок: а якщо користувач не передав параметр? а якщо передав не число, а рядок? а як згенерувати довідку?

  

### Що робить `argparse` автоматично:

1. **Генерує довідку (`--help` / `-h`):** Користувач може ввести `python script.py --help` і побачити опис усіх доступних команд.
    
      
    
2. **Типізує дані:** Перетворює вхідні рядки на `int`, `float`, відкриті `file` тощо.
    
      
    
3. **Обробляє прапорці та опції:** Короткі (`-v`) та довгі (`--verbose`) ключі.
    
      
    
4. **Перевіряє помилки:** Якщо передано неіснуючий прапорець чи відсутній обов’язковий аргумент, `argparse` зупинить скрипт і покаже зрозумілу помилку.
    
      
    

## 2. Як це працює під капотом (3 базові кроки)

Будь-яка робота з `argparse` складається з трьох послідовних кроків:

  

Python

```
import argparse

# 1. Створення парсера
parser = argparse.ArgumentParser(description="Опис того, що робить скрипт")

# 2. Додавання аргументів (що ми очікуємо від користувача)
parser.add_argument("filename", help="Шлях до файлу")

# 3. Розбір аргументів (зчитування того, що ввів користувач)
args = parser.parse_args()

# Використання значень
print(args.filename)
```

## 3. Анатомія аргументів: Позиційні vs Опціональні (Прапорці)

У `argparse` є два принципово різних типи аргументів.

  

### А. Позиційні аргументи (Positional Arguments)

Обов’язкові аргументи, значення яких визначається **їхнім порядком** у команді.

Вони створюються **без тире на початку**.

  

Python

```
parser.add_argument("source", help="Джерело")
parser.add_argument("destination", help="Призначення")
```

_Запуск:_ `python script.py /path/a /path/b`

  

- `args.source` буде `"/path/a"`
    
      
    
- `args.destination` буде `"/path/b"`
    
      
    

### Б. Опціональні аргументи (Прапорці / Flags)

Необов’язкові (за замовчуванням) аргументи, які ідентифікуються за **назвою ключа**.

Вони створюються **з тире на початку (`-` або `--`)**.

  

Python

```
parser.add_argument("-p", "--port", help="Порт підключення", default=8080, type=int)
```

_Запуск:_ `python script.py --port 9000` або `python script.py -p 9000` або просто `python script.py` (тоді порт буде `8080`).

  

## 4. Глибокий розбір методів та параметрів `add_argument()`

Метод `.add_argument()` — це серце `argparse`. Передаючи в нього різні аргументи, ти керуєш поведінкою прапорців.

  

### 🔑 Основні параметри `.add_argument()`:

#### 1. `type` — Приведення типів

За замовчуванням усі аргументи зчитуються як рядки (`str`). `type` автоматично конвертує значення:

  

Python

```
parser.add_argument("--count", type=int)      # Конвертує в integer
parser.add_argument("--rate", type=float)     # Конвертує в float
```

#### 2. `default` — Значення за замовчуванням

Якщо опціональний прапорець не передано у консолі, береться це значення:

  

Python

```
parser.add_argument("--host", default="127.0.0.1")
```

#### 3. `required=True` — Зробити опціональний прапорець обов'язковим

Зазвичай прапорці з `--` є необов'язковими, але цим параметром можна змусити користувача обов'язково їх вказати:

  

Python

```
parser.add_argument("-k", "--api-key", required=True, help="API Ключ")
```

#### 4. `action` — Поведінка прапорців (Логічні булі, лічильники)

Визначає, що робити, коли парсер бачить прапорець.

  

- **`action="store_true"` / `"store_false"`** — Перемикач (Flag). Не вимагає введення значення після прапорця:
    
      
    
    Python
    
    ```
    parser.add_argument("-v", "--verbose", action="store_true", help="Увімкнути детальний лог")
    ```
    
    - Якщо передано `--verbose` ➔ `args.verbose` стане `True`.
        
          
        
    - Якщо НЕ передано ➔ `args.verbose` буде `False`.
        
          
        
- **`action="count"`** — Рахує кількість переданих прапорців (часто для рівня деталізації відладки `-v`, `-vv`, `-vvv`):
    
      
    
    Python
    
    ```
    parser.add_argument("-v", "--verbose", action="count", default=0)
    ```
    
    - Запуск: `python script.py -vvv` ➔ `args.verbose` дорівнюватиме `3`.
        
          
        
- **`action="append"`** — Збирає повторювані прапорці у список:
    
      
    
    Python
    
    ```
    parser.add_argument("--ip", action="append")
    ```
    
    - Запуск: `python script.py --ip 1.1.1.1 --ip 8.8.8.8`
        
          
        
    - Результат: `args.ip` дорівнює `['1.1.1.1', '8.8.8.8']`.
        
          
        

#### 5. `choices` — Обмеження вибору варіантів

Дозволяє вказати список тільки дозволених значень:

  

Python

```
parser.add_argument("--env", choices=["dev", "stage", "prod"], default="dev")
```

Якщо користувач введе `--env test`, `argparse` автоматично зупинить скрипт і видасть помилку: `invalid choice: 'test' (choose from 'dev', 'stage', 'prod')`.

  

#### 6. `nargs` — Кількість аргументів (Списки / Масиви)

Визначає, скільки значень іде після прапорця або позиційного аргументу.

  

- **`nargs=N`** (число) — очікує рівно N значень:
    
      
    
    Python
    
    ```
    parser.add_argument("--ports", nargs=2, type=int) # Очікує 2 числа
    ```
    
- **`nargs="*"`** — 0 або більше значень (повертає список):
    
      
    
    Python
    
    ```
    parser.add_argument("files", nargs="*") # Сприйме будь-яку кількість файлів
    ```
    
- **`nargs="+"`** — 1 або більше значень (мінімум одне значення обов'язкове):
    
      
    
    Python
    
    ```
    parser.add_argument("files", nargs="+") 
    ```
    

## 5. Взаємовиключні аргументи (Mutually Exclusive Groups)

Бувають випадки, коли два прапорці **не можуть використовуватися одночасно** (наприклад, `--silent` та `--verbose`).

  

Python

```
group = parser.add_mutually_exclusive_group()
group.add_argument("-v", "--verbose", action="store_true")
group.add_argument("-q", "--quiet", action="store_true")
```

Якщо користувач запровадить `python script.py -v -q`, `argparse` видасть помилку: `argument -q/--quiet: not allowed with argument -v/--verbose`.

  

## 6. Вкладені команди (Sub-commands / Як у `git` чи `docker`)

Коли ти пишеш складну CLI-утиліту, тобі можуть знадобитися підкоманди, як у `git clone` чи `docker run`.

  

Python

```
import argparse

parser = argparse.ArgumentParser(description="DevOps CLI Tool")
subparsers = parser.add_subparsers(dest="command", help="Доступні команди")

# Підкоманда 'start'
start_parser = subparsers.add_parser("start", help="Запустити сервіс")
start_parser.add_argument("--port", type=int, default=80)

# Підкоманда 'stop'
stop_parser = subparsers.add_parser("stop", help="Зупинити сервіс")
stop_parser.add_argument("--force", action="store_true")

args = parser.parse_args()

if args.command == "start":
    print(f"Запуск на порту {args.port}")
elif args.command == "stop":
    print(f"Зупинка (Force={args.force})")
```

## 💎 Повний майстер-шаблон для DevOps

Ось ідеальний приклад консольного скрипту, який об'єднує все вищезазначене:

  

Python

```
import argparse
import sys

def main():
    # 1. Створюємо парсер
    parser = argparse.ArgumentParser(
        prog="LogAnalyzer",
        description="Програма для фільтрації та аналізу серверних логів.",
        epilog="Приклад використання: python analyzer.py server.log -status 500 -o errors.log -v"
    )

    # 2. Позиційний аргумент (Обов'язковий)
    parser.add_argument("logfile", type=str, help="Шлях до лог-файлу для аналізу")

    # 3. Опціональні прапорці з обмеженням вибору та типом
    parser.add_argument(
        "-s", "--status", 
        type=str, 
        default="500", 
        help="Статус-код для фільтрації (наприклад, 200, 404, 500)"
    )

    # 4. Файл для виводу
    parser.add_argument(
        "-o", "--output", 
        type=str, 
        help="Файл для збереження результатів (якщо не вказано, виводить в консоль)"
    )

    # 5. Прапорець-перемикач (Boolean)
    parser.add_argument(
        "-v", "--verbose", 
        action="store_true", 
        help="Виводити детальні логи під час виконання"
    )

    # 6. Парсимо аргументи з консолі
    args = parser.parse_args()

    # --- ЛОГІКА СКРИПТУ ---
    if args.verbose:
        print(f"🔍 Початок аналізу файлу: {args.logfile}")
        print(f"🎯 Шукаємо статус-код: {args.status}")

    try:
        with open(args.logfile, "r", encoding="utf-8") as f:
            matching_lines = [line for line in f if args.status in line]

        if args.verbose:
            print(f"✅ Знайдено {len(matching_lines)} рядків.")

        # Вивід результату
        if args.output:
            with open(args.output, "w", encoding="utf-8") as out:
                out.writelines(matching_lines)
            print(f"💾 Результати збережено у {args.output}")
        else:
            print("--- Знайдені рядки ---")
            print("".join(matching_lines))

    except FileNotFoundError:
        print(f"❌ Помилка: Файл '{args.logfile}' не знайдено!", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    main()
```

### Як працює автоматично згенерована довідка з цим шаблоном:

Якщо запустити `python analyzer.py --help`, розробник побачить вивід:

  

Plaintext

```
usage: LogAnalyzer [-h] [-s STATUS] [-o OUTPUT] [-v] logfile

Програма для фільтрації та аналізу серверних логів.

positional arguments:
  logfile               Шлях до лог-файлу для аналізу

options:
  -h, --help            show this help message and exit
  -s STATUS, --status STATUS
                        Статус-код для фільтрації (наприклад, 200, 404, 500)
  -o OUTPUT, --output OUTPUT
                        Файл для збереження результатів
  -v, --verbose         Виводити детальні логи під час виконання

Приклад використання: python analyzer.py server.log -status 500 -o errors.log -v
```

Тепер ти маєш вичерпне керівництво по `argparse`! З цим інструментом твої Python-скрипти стануть справжніми професійними CLI-утилітами.