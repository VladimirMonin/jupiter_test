# Jupyter Notebook с OpenAI API

Проект демонстрирует работу с OpenAI API через Jupyter Notebook на Python.

## Требования

- Python 3.12+
- pip3
- Виртуальное окружение Python
- OpenAI API ключ

## Установка

### Windows

```bash
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

### MacOS/Linux

```bash
python3 -m venv venv
source venv/bin/activate
pip3 install -r requirements_mac.txt
```

## Настройка

1. Создайте файл `.env` в корневой директории проекта
2. Добавьте ваш OpenAI API ключ:

OPENAI_API_KEY="ваш-ключ-api"


## Запуск

1. Активируйте виртуальное окружение (если еще не активировано)
2. Запустите Jupyter Lab:

```bash
jupyter notebook
```


3. Откройте `Untitled.ipynb` в браузере

## Структура проекта

- `requirements.txt` - зависимости для Windows
- `requirements_mac.txt` - зависимости для MacOS
- `test.md` - тестовый markdown файл
- `Untitled.ipynb` - основной Jupyter Notebook с примером работы OpenAI API
- `.env` - файл с переменными окружения (не включен в репозиторий)
- `.env.example` - пример файла с переменными окружения

## Функциональность

Notebook демонстрирует:

- Настройку OpenAI API
- Отправку запросов к API
- Получение и обработку ответов
- Работу с Markdown в Jupyter