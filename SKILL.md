---
name: redpolitika-rules
description: >
  Guide for AI agents writing YAML rules for Redpolitika — a
  Russian text analysis service. Covers rule schema (detect/fix trees, severity,
  categories), detection methods (regex, wordlist, context, composition, ref),
  fix transformations, RE2 regex constraints, scoring, merge layers, and best
  practices.
  Use when user asks to create, edit, or validate Redpolitika rules; when working
  with .yaml rule files for Redpolitika; or when user mentions "redpolitika",
  "правила редполитики", "rule for text analysis".
---

# SKILL.md — Написание правил Редполитики

Навык для ИИ-агентов: как писать YAML-правила для [Редполитики](https://github.com/drupaldoesnotexists/redpolitika-ce).

Читай этот файл целиком перед тем, как писать или редактировать правила. Все методы, форматы и ограничения описаны ниже с примерами.

---

## 1. Быстрый старт

Минимальное правило:

```yaml
rules:
  - id: "category/my-rule"
    severity: 5
    category: "cleanliness"
    detect:
      regex: "плохое\\s+слово"
```

Правило с автоисправлением:

```yaml
rules:
  - id: "typography/ellipsis"
    severity: 5
    category: "readability"
    detect:
      regex: "\\.\\.\\."
    fix:
      replace:
        with: "…"
      suggestion: "Замените три точки на символ многоточия"
```

---

## 2. Полная схема поля

```yaml
rules:
  - id: "category/descriptive-name"   # unique, латиница/транслит
    severity: 5                       # 1–10, выше = хуже (или 0 — helper-правило)
    category: "cleanliness"           # cleanliness | readability | произвольный
    name: "Человеческое название"     # опционально, отображается пользователю
    url: "/rules/my-rule"             # опционально, ссылка на доку
    suggestion: "Что делать"          # опционально, подсказка при срабатывании
    priority: 2                       # опционально, любое целое число — порядок срабатывания
    enabled: true                     # опционально, false = отключить (override)
    detect:                           # обязательно (кроме override)
      # один метод или композиция
    fix:                              # опционально
      # один метод или композиция
      suggestion: "Подсказка"          # можно на уровне fix
    examples:                         # опционально
      bad:
        - "Неверный пример"
      good:
        - "Верный пример"
    related:                          # опционально
      - name: "Связанное правило"
        url: "/rules/related"
```

### 2.1 severity=0 (helper-правила)

Правила с `severity: 0` — служебные: они не производят флагов, не влияют на score, не отправляются клиенту. Используются только как цель для `ref`.

```yaml
rules:
  - id: "commons/strong-words"
    detect:
      wordlist:
        list: ["очень", "крайне", "чрезвычайно"]
  # ↑ severity=0 по умолчанию — helper, ref-only

  - id: "redundancy/strong-adjectives"
    severity: 5
    category: "readability"
    detect:
      and:
        - ref: "commons/strong-words"
        - regex: "\\w+"
```

---

## 3. Дерево детекции (detect)

Один метод или композиция. Движок обрабатывает рекурсивно. Методы делятся на:

- Листовые — regex, wordlist, contains, eq, prefix, suffix, any, ref
- Позиционные — sentence_start, sentence_end, paragraph_start, paragraph_end, word_boundary, length, case
- Контекстные — before, after, surrounded_by, position, threshold, near, exclude
- Логические — and, or, not

### 3.1 Листовые методы

#### regex

RE2-совместимый. Без backreferences, без lookahead/lookbehind.

```yaml
detect:
  regex: "шаблон"
```

С флагами:

```yaml
detect:
  regex:
    pattern: "шаблон"
    case_sensitive: true
```

#### wordlist

Список слов с учётом границ. По умолчанию регистронезависимый.

```yaml
detect:
  wordlist:
    list: ["word1", "word2"]
    case_sensitive: false
```

#### contains

Поиск подстроки без границ слов.

```yaml
detect:
  contains:
    value: "подстрока"
    case_sensitive: false
```

#### eq

Точное совпадение всей строки.

```yaml
detect:
  eq: "точный текст"
```

#### prefix / suffix

Начало / конец текста.

```yaml
detect:
  prefix: "Начало"
```

```yaml
detect:
  suffix: "Конец"
```

#### any

Всегда совпадает. Для default/fallback в композитах.

```yaml
detect:
  any: {}
```

#### ref

Делегирование detect-дереву другого правила. Ссылка по `id`.

```yaml
detect:
  ref: "category/other-rule"
```

С явным ключом:

```yaml
detect:
  ref:
    pattern: "category/other-rule"
```

Целевое правило должно существовать и иметь detect. Циклические ref запрещены.

### 3.2 Позиционные методы

#### sentence_start / sentence_end

Границы предложений (`.`, `!`, `?`). С дочерним методом — оставляет совпадения на границе.

```yaml
detect:
  sentence_start:
    regex:
      pattern: "[а-яё]"
      case_sensitive: true
```

```yaml
detect:
  sentence_end: {}
```

#### paragraph_start / paragraph_end

Границы абзацев.

```yaml
detect:
  paragraph_start:
    wordlist:
      list: ["Во-первых"]
```

#### word_boundary

Оборачивает child проверкой границ слова.

```yaml
detect:
  word_boundary:
    contains:
      value: "слово"
```

#### length

Фильтр по длине совпадения.

```yaml
detect:
  length:
    min: 3
    max: 50
    child:
      regex: "\\b\\w+\\b"
```

#### case

Проверка регистра.

```yaml
detect:
  case:
    mode: "all_caps"   # all_caps | all_lower | capitalized | has_upper | has_lower
    child:
      wordlist:
        list: ["москва"]
```

Без child проверяет каждое слово (`\S+`).

### 3.3 Контекстные методы

#### before / after

Child совпадает, если рядом есть pattern в пределах max_chars.

```yaml
detect:
  before:
    pattern:
      regex: "уважаемый\\s+"
    max_chars: 50
    child:
      wordlist:
        list: ["господин"]
```

Без pattern — проверяет один символ до/после.

#### surrounded_by

Совпадение между левым и правым маркерами.

```yaml
detect:
  surrounded_by:
    left: "«"
    right: "»"
    child:
      contains:
        value: "важно"
```

#### position

Первый/последний абзац.

```yaml
detect:
  position:
    type: "first_paragraph"   # first_paragraph | last_paragraph
    child:
      wordlist:
        list: ["введение"]
```

Числовая позиция:

```yaml
detect:
  position:
    from: 1
    to: 3
```

#### threshold

Флаг только когда count ≥ N в окне.

```yaml
detect:
  threshold:
    count: 3
    per: words          # words | paragraph | text
    window: 100         # для per=words
    child:
      wordlist:
        list: ["вроде"]
```

#### near

Флаг pattern, рядом с которым есть near.

```yaml
detect:
  near:
    pattern:
      wordlist: { list: ["X"] }
    near:
      wordlist: { list: ["Y"] }
    window: sentence    # sentence | chars:N | integer chars
```

#### exclude

Сахар над and + not: pattern минус исключения.

```yaml
detect:
  exclude:
    match:
      regex: "[а-яА-ЯёЁ]*ё[а-яА-ЯёЁ]*"
    without:
      - wordlist:
          list: ["ёлка", "ёжик"]
      - any: {}
```

Старый формат (обратная совместимость):

```yaml
detect:
  exclude:
    list: ["ёлка", "ёжик"]
    child:
      regex: "[а-яА-ЯёЁ]*ё[а-яА-ЯёЁ]*"
```

### 3.4 Логические операторы

#### and

Пересечение — все дети должны совпасть. Вложенный `not` = исключение (вычитается из пересечения).

```yaml
detect:
  and:
    - regex: "очень"
    - contains:
        value: "важно"
```

#### or

Объединение — любой child совпадает.

```yaml
detect:
  or:
    - regex: "кр[ао]сивый"
    - wordlist:
        list: ["прелестный"]
```

#### not

Осмыслен только как исключение внутри `and`. Отдельный `not` возвращает child без изменений (не инвертирует).

```yaml
detect:
  and:
    - wordlist: { list: ["ещё", "всё"] }
    - not:
        wordlist: { list: ["всё"] }
```

---

## 4. Дерево фиксов (fix)

Опционально. Каждый фикс заменяет только span совпадения.

### 4.1 Листовые методы

```yaml
fix:
  replace:
    with: "замена"
```

```yaml
fix:
  remove: {}
```

```yaml
fix:
  regex_replace:
    pattern: ".*"
    replacement: "$2, $1"  # capture groups из detect
```

```yaml
fix:
  when:
    detect:
      case: { mode: all_lower }
    then:
      replace:
        with: "Исправлено"
```

Чтобы удалить слово совсем, его нужно заменить на пустую строку. Клиентский движок сам отобразит подсказку «удалить» в примерах и почистит пунктуацию и лишние пробелы.

**Частая ошибка:** вместо пустой строки указывать пробел — движок не понимает, что слово удалили и оставляет зависший пробел.

### 4.2 Трансформации регистра

```yaml
fix:
  uppercase: {}
```

```yaml
fix:
  lowercase: {}
```

```yaml
fix:
  capitalize: {}       # первая буква заглавная
```

```yaml
fix:
  sentence_case: {}    # первая заглавная, остальные строчные
```

```yaml
fix:
  title_case: {}       # каждое слово с заглавной (Unicode)
```

### 4.3 Манипуляции с текстом

```yaml
fix:
  prepend:
    with: "префикс "
```

```yaml
fix:
  append:
    with: " суффикс"
```

```yaml
fix:
  wrap:
    prefix: "«"
    suffix: "»"
```

```yaml
fix:
  trim: {}
```

```yaml
fix:
  collapse_whitespace: {}
```

### 4.4 Композиция фиксов

```yaml
fix:
  and:
    - lowercase: {}
    - wrap:
        prefix: "*"
        suffix: "*"
```

Фиксы применяются последовательно, каждый видит результат предыдущего.

---

## 5. RE2 — единственный движок regex

Все regex-паттерны обязаны быть RE2-совместимыми.

**Запрещено:**
- `(?!...)`, `(?<=...)`, `(?<!...)` — lookahead/lookbehind
- `\1`, `\2`, `$1` в pattern (только в replacement regex_replace)
- `(?(...)...|...)` — conditional
- `\G`, `\K`

**Можно:**
- `(?:...)` — non-capturing groups
- `\b`, `\B` — word boundaries
- `\d`, `\w`, `\s` — character classes
- `[а-яё]` — unicode ranges
- `+`, `*`, `?`, `{n,m}` — quantifiers (кроме possessive `++`, `*+`)

Если нужен lookaround — используй `before`, `after`, `surrounded_by`, `near`.

---

## 6. Скоринг

Две оценки: чистота и читаемость (0–10). Штраф за каждый pending (непринятый) флаг:

```
score = 10 - clamp(penalty / max(wordCount, 100) * 100, 0, 10)
```

- penalty = сумма severity всех pending флагов в категории
- Принятые и отклонённые флаги не штрафуют
- Тексты короче 100 слов считаются как 100 (не давим короткие тексты)

---

## 7. Загрузка и мерж правил

Три слоя, deep-merge по `id`:

1. **Base** — `RULES_DIR` (по умолч. `./rules`)
2. **Project** — `RULES_PROJECT_DIR`
3. **Override** — `RULES_OVERRIDE_DIR`

Поздний слой перекрывает ранний. Чтобы отключить правило:

```yaml
rules:
  - id: "category/my-rule"
    enabled: false
```

---

## 8. Client vs server

**Client-side** правила — detect содержит только: `regex`, `wordlist`, `any`, `contains`, `eq`, `prefix`, `suffix`, `surrounded_by`, `and`, `or`, `not` (если все дети синхронны). Выполняются на фронтенде.

**Server-side** — содержат `ref` или позиционные/контекстные методы, вызовы плагинов, сложную или долгую логику.

Сервер всегда выполняет ВСЕ правила (и клиентские, и серверные). Клиентские нужны для предварительной проверки.

---

## 9. Полные примеры

### Типография

```yaml
rules:
  - id: "typography/ellipsis"
    severity: 5
    category: "readability"
    detect:
      regex: "\\.\\.\\."
    fix:
      replace:
        with: "…"
      suggestion: "Замените три точки на символ многоточия"
```

### Список слов с автоисправлением

```yaml
rules:
  - id: "cleanliness/obscene"
    severity: 8
    category: "cleanliness"
    detect:
      wordlist:
        list: ["дурак", "идиот", "кретин"]
        case_sensitive: false
    fix:
      replace:
        with: "***"
```

### Исключение из правила

```yaml
rules:
  - id: "typography/yo-exclude"
    severity: 3
    category: "readability"
    detect:
      exclude:
        match:
          wordlist:
            list: ["ёлка", "ёжик"]
        without:
          - regex: "[а-яА-ЯёЁ]*ё[а-яА-ЯёЁ]*"
```

### Тавтология (or + eq)

```yaml
rules:
  - id: "readability/tautology"
    severity: 3
    category: "readability"
    detect:
      or:
        - eq: "масло масляное"
        - eq: "неожиданный сюрприз"
    fix:
      remove: {}
```

### Композиция — строчная после двоеточия

```yaml
rules:
  - id: "capitalization/after-colon"
    severity: 4
    category: "cleanliness"
    detect:
      after:
        pattern:
          contains:
            value: ": "
        max_chars: 80
        child:
          regex: "^[А-Я]"
    fix:
      suggestion: "После двоеточия — строчная буква"
```

### Вспомогательное правило с ref

Правило не будет создавать флаги само по себе. Срабатывает только при вызове из других правил.

```yaml
rules:
  - id: "commons/strong-words"
    detect:
      wordlist:
        list: ["очень", "крайне", "чрезвычайно"]

  - id: "redundancy/strong-adjectives"
    severity: 5
    category: "readability"
    detect:
      and:
        - ref: "commons/strong-words"
        - regex: "\\w+"
```

### Порог в тексте

```yaml
rules:
  - id: "readability/too-many-modal-verbs"
    severity: 3
    category: "readability"
    detect:
      threshold:
        count: 5
        per: paragraph
        child:
          wordlist:
            list: ["должен", "обязан", "нужно", "следует"]
```

---

## 10. Best practices

1. **id**: используй `category/descriptive-name` — транслит, латиница. Пример: `typography/ellipsis`, `cleanliness/obscene`, `redundancy/very`
2. **severity**: 1–3 косметические, 4–6 важные, 7–10 критичные. Для вспомогательных правил ставь 0 или опусти (0 по умолчанию)
3. **category**: `cleanliness` — грамматика, стиль, канцелярит; `readability` — читаемость, типографика
4. **suggestion**: кратко опиши причину, потом чёткое конкретное решение: «Римские цифры сложно читать. Замените на арабские». Не добавляй громоздкие примеры в suggestion — для них есть examples.
5. **regex**: всегда проверяй на RE2-совместимость. Предпочитай сложному паттерну композицию из методов
6. **exclude**: используй для исключения ложных срабатываний вместо сложного regex
7. **ref**: выноси общие подпаттерны в helper-правила (severity=0) для переиспользования. Вспомогательные правила кэшируются. Используй ref для взаимоисключающих правил.
8. **fiх**: если правило только находит, но не исправляет — не пиши fix (будет suggestion, но без автоисправления)
9. **examples**: всегда давай bad/good примеры — они видны пользователю. Можешь давать примеры по парам: они будут визуально противопоставлены по колонкам. Одна колонка может быть больше другой.
10. **related**: старайся всегда указывать related для правил. Предпочитай точные ссылки: не на всю википедию, а на конкретную статью или абзац; не на весь словарь, а на конкретную словарную статью. Для русского языка ссылайся на порталы Грамота.ру, словарь Розенталя, Бюро Горбунова, блог Ильи Бирмана и другие авторитетные источники, которые укажет пользователь. Хороший тон — указывать оригинальное или близкое название статьи и источник в названии ссылки: не «Какая-то статья о зелёных розах», а «Почему розы зелёные — Грамота».

---

## 11. Ошибки загрузки

- Любая ошибка в YAML блокирует запуск
- Циклические ref → ошибка загрузки
- RE2-несовместимый regex → ошибка загрузки
- Голый `not` вне `and` → работает, но бесполезен (предупреждение)
- Правило без `detect` при enabled=true → ошибка (кроме override)

---

## 12. Проверка правил

Перед отправкой проверь:

1. YAML синтаксически корректен
2. `id` уникален, в формате `category/name`
3. `severity` 0–10
4. `category` указана (кроме severity=0)
5. detect содержит существующий метод
6. Все regex RE2-совместимы
7. ref указывает на существующий id
8. Нет циклических ref
9. examples имеют bad/good где возможно
10. Если есть fix — он совместим с detect
11. Правила имеют related, где возможно
12. Проверь, что похожие правила имеют исключения. Предпочитай конкретное правило общему: например, если есть правило о латинице в русском тексте и правило о римских цифрах, правилу о римских цифрах ставь приоритет, правилу о латинице — исключение через exclude и ref.

---

*Этот навык покрывает CE-версию Редполитики — это ядро. EE добавляет плагины с новыми методами, но схема правил остаётся той же. Справка по EE появится позже*
