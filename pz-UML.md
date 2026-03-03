# PZ-UML: Поведінкові UML-діаграми системи «Gradebook»

**Gradebook** — система обліку навчальних досягнень для військового інституту.
Управління журналами, оцінками, відвідуваністю. Автентифікація через Keycloak (JWT/OAuth2).

---

## 1. Use Case Diagram

```plantuml
@startuml
left to right direction
skinparam actorStyle awesome

actor "Курсант" as CADET
actor "Викладач" as TEACHER
actor "Нач. кафедри" as DEPT_HEAD
actor "Навч. відділ" as ADMIN
actor "Суперадмін" as SUPERADMIN

DEPT_HEAD --|> TEACHER

rectangle "Gradebook — Система обліку навчальних досягнень" {

  usecase "Переглянути журнал" as UC1
  usecase "Переглянути оцінки" as UC2
  usecase "Переглянути відвідуваність" as UC3
  usecase "Створити / редагувати журнал" as UC4
  usecase "Виставити оцінку" as UC5
  usecase "Записати відвідуваність" as UC6
  usecase "Створити заняття" as UC7
  usecase "Управляти дисциплінами\nкафедри" as UC8
  usecase "Управляти користувачами\nта групами" as UC9
  usecase "Імпортувати\nкористувачів (JSON)" as UC10

}

CADET --> UC1
CADET --> UC2
CADET --> UC3

TEACHER --> UC1
TEACHER --> UC4
TEACHER --> UC5
TEACHER --> UC6
TEACHER --> UC7

DEPT_HEAD --> UC8

ADMIN --> UC1
ADMIN --> UC2
ADMIN --> UC3

SUPERADMIN --> UC4
SUPERADMIN --> UC5
SUPERADMIN --> UC9
SUPERADMIN --> UC10

note right of UC5
  Перевіряється:
  1. Роль TEACHER/DEPT_HEAD
  2. Призначення до дисципліни
  3. markValue ≤ maxMarkValue
end note

@enduml
```

---

## 2. Sequence Diagram — Виставлення оцінки

```plantuml
@startuml
actor "Викладач" as T
participant "Клієнт" as App
participant "Keycloak" as KC
participant "API" as API
participant "SecurityService" as Sec
database "PostgreSQL" as DB

T -> App: Відкриває застосунок
App -> KC: OAuth2 логін
KC --> App: JWT токен (ролі + email)

T -> App: Вводить оцінку для курсанта
App -> API: POST /marks {subLessonId, cadetId, markValue}
API -> API: Декодує JWT → ROLE_TEACHER

API -> Sec: canCreateMark(subLessonId)
Sec -> DB: Чи призначений викладач до дисципліни?
DB --> Sec: так / ні
Sec --> API: true / false

alt Доступ заборонено
    API --> App: 403 Forbidden
else Доступ дозволено
    API -> DB: Перевірити markValue ≤ maxMarkValue
    alt Оцінка невалідна
        API --> App: 400 Bad Request
    else Валідна
        API -> DB: INSERT INTO marks
        API -> DB: UPDATE discipline_rate, semester_rate
        API --> App: 201 Created
        App --> T: Оцінку збережено ✓
    end
end

@enduml
```

---

## 3. Activity Diagram — Проведення заняття

```plantuml
@startuml
start

:Відкрити застосунок;

if (JWT активний?) then (ні)
  :Логін через Keycloak;
  :Отримати JWT токен;
endif

:Обрати семестр та дисципліну;

if (Журнал існує?) then (ні)
  :Створити журнал;
endif

:Створити заняття\n(тема, тип, дата, аудиторія);

repeat

  :Обрати дію;

  split
    :Облік відвідуваності;
    repeat
      :Відмітити курсанта\n(YES / NO / ABSENT);
    repeat while (Ще курсанти?) is (так)
    :Зберегти відвідуваність\nPOST /attends/batch;

  split again
    :Оцінювання;
    repeat
      :Ввести оцінку для курсанта;
      if (markValue ≤ maxMarkValue?) then (ні)
        :Показати помилку 400;
      else (так)
        if (Доступ до дисципліни?) then (ні)
          :Показати помилку 403;
        else (так)
          :Зберегти оцінку\nPOST /marks;
          :Перерахувати рейтинг\n(discipline_rate, semester_rate);
        endif
      endif
    repeat while (Ще курсанти?) is (так)

  end split

repeat while (Продовжити роботу?) is (так)

stop
@enduml
```

---

## Зв'язок між діаграмами

- **Use Case** — хто і що може робити в системі
- **Sequence** — як виконується сценарій «Виставити оцінку» (деталізує UC5)
- **Activity** — повний workflow заняття від входу до завершення

Інструмент: усі три діаграми — [plantuml.com](https://www.plantuml.com/plantuml/uml/)