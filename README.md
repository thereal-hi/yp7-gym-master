# Гимнастический зал — система учёта расписания

Java-приложение для управления расписанием детского и взрослого гимнастического зала.
Позволяет администраторам хранить занятия, искать их по дню и времени, а также считать
нагрузку тренеров для начисления зарплаты.

![Java](https://img.shields.io/badge/Java-17+-blue)
![JUnit](https://img.shields.io/badge/JUnit-5-green)
![Maven](https://img.shields.io/badge/Maven-3.8+-orange)

---

## Возможности

- 📅 **Хранение занятий** с привязкой к дню недели и времени начала.
- ⚡ **Быстрый поиск занятий по дню недели** — O(1).
- ⚡ **Быстрый поиск занятий по дню и времени** — O(1).
- 📊 **Подсчёт количества тренировок каждого тренера** за неделю с сортировкой по убыванию.
- 👶🧑 **Разделение занятий** на детские и взрослые группы.
- 🗂 **Естественный порядок** занятий по времени внутри дня.

---

## Структура проекта

| Класс | Назначение |
|---|---|
| `Timetable` | Хранит расписание и предоставляет методы поиска и подсчёта |
| `TrainingSession` | Одно занятие: группа, тренер, день недели, время |
| `Group` | Группа занимающихся: название, возрастная категория, длительность |
| `Coach` | Тренер: фамилия, имя, отчество |
| `TimeOfDay` | Время начала занятия (часы, минуты) |
| `Age` | Возрастная категория: `CHILD`, `ADULT` |

### Ответственности классов

| Класс | Что делает | Чего **не** делает |
|---|---|---|
| `Timetable` | Хранит занятия, ищет по дню и времени, считает нагрузку | Не знает, как устроено занятие внутри |
| `TrainingSession` | Связывает группу, тренера, день и время | Не сортирует себя |
| `Group` | Хранит название, возраст, длительность | Не знает про расписание |
| `Coach` | Хранит ФИО | Не знает про занятия |
| `TimeOfDay` | Часы и минуты, сравнивается по времени | Не знает про день недели |
| `Age` | Enum `CHILD`, `ADULT` | — |

---

## Ключевые решения

### 1. Вложенная структура хранения

```java
Map<DayOfWeek, Map<TimeOfDay, List<TrainingSession>>> timetable;
```

Внешняя мапа даёт **O(1)** по дню недели. Внутренняя — **O(1)** по времени.

Почему именно так:

- Хочется быстро искать занятия и по дню, и по дню+времени.
- Плоский список заставлял бы перебирать всё расписание — O(n).
- Вложенная мапа использует хеширование: `DayOfWeek` и `TimeOfDay` → бакет с занятиями.

### 2. `TreeMap` для внутренней мапы

```java
Map<TimeOfDay, List<TrainingSession>> inner = new TreeMap<>();
```

Внутренняя мапа — `TreeMap`, а не `HashMap`, потому что:

- Гарантирует **порядок занятий по времени** — `getTrainingSessionsForDay` сразу отдаёт отсортированный список.
- Не нужно сортировать результат вручную.
- `TimeOfDay implements Comparable` — задаёт естественный порядок.

### 3. `TimeOfDay implements Comparable`

```java
public class TimeOfDay implements Comparable<TimeOfDay> {
    @Override
    public int compareTo(TimeOfDay other) {
        ...
    }
}
```

Задаёт **естественный порядок** — `13:00 < 20:00`. Благодаря этому `TreeMap`
автоматически сортирует ключи, а `Collections.sort()` работает без явного `Comparator`.

### 4. `equals`/`hashCode` у `Coach`, `Group`, `TimeOfDay`, `TrainingSession`

Без корректной реализации `equals`/`hashCode`:

- `HashMap` не найдёт ключ, даже если он «равен» существующему.
- Тесты будут падать на сравнении объектов.
- `List.remove(Object)` не сработает.

Правило: если два объекта считаются «одинаковыми» по смыслу — у них должны быть
одинаковые `hashCode` и `equals` возвращать `true`.

### 5. `LinkedHashMap` в `getCountByCoaches`

```java
Map<Coach, Integer> result = new LinkedHashMap<>();
```

После сортировки тренеров по числу занятий порядок нужно **сохранить**. Обычный
`HashMap` перемешал бы порядок, и результат сортировки потерялся бы. `LinkedHashMap`
итерацию ведёт в порядке вставки — то есть в порядке сортировки.

---

## Методы `Timetable`

| Метод | Что делает |
|---|---|
| `addNewTrainingSession(TrainingSession)` | Добавляет занятие в расписание |
| `getTrainingSessionsForDay(DayOfWeek)` | Возвращает все занятия за день, отсортированные по времени |
| `getTrainingSessionsForDayAndTime(DayOfWeek, TimeOfDay)` | Возвращает занятия в конкретный день и время |
| `getCountByCoaches()` | Возвращает количество занятий каждого тренера, по убыванию |

### Пример использования

```java
Timetable timetable = new Timetable();

Coach coach = new Coach("Иванов", "Иван", "Иванович");
Group group = new Group("Младшая", Age.CHILD, 60);
TimeOfDay time = new TimeOfDay(15, 0);

TrainingSession session = new TrainingSession(group, coach, DayOfWeek.MONDAY, time);
timetable.addNewTrainingSession(session);

// Все занятия в понедельник, отсортированные по времени
List<TrainingSession> monday = timetable.getTrainingSessionsForDay(DayOfWeek.MONDAY);

// Занятия в понедельник в 15:00
List<TrainingSession> atThree = timetable.getTrainingSessionsForDayAndTime(
        DayOfWeek.MONDAY, new TimeOfDay(15, 0));

// Нагрузка тренеров за неделю, по убыванию
Map<Coach, Integer> load = timetable.getCountByCoaches();
```

---

## Технологии

- **Java 17+** — records, switch-выражения, `var`.
- **JUnit 5** — модульные тесты.
- **Maven** — сборка и управление зависимостями.
- **Коллекции:**
  - `HashMap` — внешняя мапа в `Timetable`.
  - `TreeMap` — внутренняя мапа, для порядка по времени.
  - `LinkedHashMap` — сохранение порядка после сортировки.
  - `ArrayList` — список занятий в одном временном слоте.
