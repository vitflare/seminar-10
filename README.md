# Семинар 10: Индексы в PostgreSQL

## Описание
На этом семинаре мы изучим различные типы индексов в PostgreSQL, способы их создания и анализа эффективности. Семинар разделен на две части:

1. Базовые B-tree индексы и их применение
2. Специальные случаи использования индексов (составные, функциональные, партиционированные)

## Подготовка окружения

1. Установите Docker и Docker Compose если ещё не установлены
2. Запустите PostgreSQL из папки `src`:
   ```bash
   docker compose up -d
   ```
3. PostgreSQL будет доступен по адресу localhost:5432
   - База данных: workshop
   - Пользователь: student
   - Пароль: student

## Задания

### Задание 1: Базовые B-tree индексы

Цель: Понять принципы работы B-tree индексов и их влияние на производительность запросов.

Задачи:
1. Анализ планов выполнения запросов без индексов
2. Создание и тестирование простых B-tree индексов
3. Исследование влияния статистики на планы выполнения
4. Сравнение эффективности индексов для разных типов запросов

### Задание 2: Специальные случаи использования индексов

Цель: Изучить продвинутые возможности индексов и особые случаи их применения.

Задачи:
1. Работа с составными индексами
2. Создание и использование функциональных индексов
3. Анализ селективности индексов
4. Оптимизация сложных запросов с помощью индексов

## Требования к выполнению

1. Создайте ветку `seminar-10/lastname-firstname`
3. Заполните файлы `task1.md` и `task2.md` результатами выполнения заданий
4. Создайте Pull Request

### Структура решения
```
lastname-firstname/
├── task1.md       # Отчет по первому заданию
├── task2.md       # Отчет по второму заданию
└── screenshots/   # Скриншоты планов выполнения
```

### Критерии оценки

- **10 баллов**: 
  - Все задания выполнены правильно
  - Предоставлен подробный анализ каждого плана выполнения
  - Даны исчерпывающие объяснения выбранных решений

- **9 баллов**:
  - Все задания выполнены правильно
  - Планы выполнения представлены для всех запросов
  - Объяснения результатов частично недостаточно подробные
  
- **8 баллов**:
  - Задания выполнены правильно
  - Требуются небольшие доработки в анализе или объяснениях
  
- **7 баллов**:
  - Есть существенные недочеты
  - Отсутствует анализ части результатов

## Дополнительные материалы

- [Документация PostgreSQL по индексам](https://www.postgresql.org/docs/current/indexes.html)
- [Use The Index, Luke!](https://use-the-index-luke.com/)

  
