# Horoshop → KeepinCRM XML Export

## Опис

Цей проєкт автоматично експортує товари з **Horoshop API** та формує XML-файл `keepin.xml`, який використовується для імпорту товарів у **KeepinCRM**.

XML генерується у форматі, сумісному з YML-структурою:

* створюється дерево категорій
* створюються товари
* підтримуються модифікації товарів

Після генерації файл автоматично публікується у публічному репозиторії.

KeepinCRM завантажує цей файл напряму по URL.

---

# Архітектура системи

```
Horoshop API
     │
     │ (catalog/export)
     ▼
GitHub Repository
keepin-xml-builder (private)
     │
     │ GitHub Actions
     ▼
export_to_keepin_xml.py
     │
     │ генерує
     ▼
keepin.xml
     │
     │ commit + push
     ▼
Joneroom/keepin-xml-public (public)
     │
     │ raw.githubusercontent.com
     ▼
KeepinCRM
```

---

# Репозиторії

## 1️⃣ keepin-xml-builder (private)

Основний репозиторій генерації XML.

Містить:

```
export_to_keepin_xml.py
.github/workflows/build_keepin_xml.yml
```

Основні задачі:

* авторизація у Horoshop API
* експорт товарів
* побудова структури категорій
* обробка модифікацій товарів
* генерація XML файлу `keepin.xml`
* публікація файлу у public repository

---

## 2️⃣ keepin-xml-public (public)

Публічний репозиторій, що містить:

```
keepin.xml
BUILD.txt
```

Саме цей файл використовується KeepinCRM для імпорту товарів.

URL для імпорту:

```
https://raw.githubusercontent.com/Joneroom/keepin-xml-public/main/keepin.xml
```

---

# Автоматичне оновлення

GitHub Actions запускається автоматично **кожні 6 годин**.

Файл workflow:

```
.github/workflows/build_keepin_xml.yml
```

Розклад:

```
cron: "0 */6 * * *"
```

Це означає:

| UTC   | Київ  |
| ----- | ----- |
| 00:00 | 02:00 |
| 06:00 | 08:00 |
| 12:00 | 14:00 |
| 18:00 | 20:00 |

При кожному запуску:

1. GitHub авторизується у Horoshop
2. отримує товари через API
3. генерує новий XML
4. публікує файл у public repository

---

# Отримання товарів з Horoshop

Використовується API endpoint:

```
POST /api/catalog/export/
```

Фільтр:

```
display_in_showcase = 1
```

Завантажуються поля:

* article
* parent_article
* title
* price
* link
* parent
* images
* gallery_common

---

# Формування категорій

Категорії формуються з поля:

```
parent.value
```

Приклад:

```
Зарядні станції/EcoFlow/Серія Delta
```

На основі цього будується дерево:

```
Зарядні станції
└── EcoFlow
    └── Серія Delta
```

У XML це виглядає так:

```
<categories>
  <category id="1">Зарядні станції</category>
  <category id="2" parentId="1">EcoFlow</category>
  <category id="3" parentId="2">Серія Delta</category>
</categories>
```

---

# Формування товарів

Кожен товар експортується у вигляді:

```
<offer>
```

Приклад:

```
<offer id="EFDELTA1300-EU">
    <name>Зарядна станція EcoFlow DELTA</name>
    <price>44999</price>
    <url>https://airunit.com.ua/zaryadna-stantsiya-ecoflow-delta/</url>
    <categoryId>123</categoryId>
    <vendorCode>EFDELTA1300-EU</vendorCode>
</offer>
```

---

# Підтримка модифікацій товарів

Horoshop використовує поле:

```
parent_article
```

Якщо

```
article ≠ parent_article
```

товар є **модифікацією**.

У XML це виглядає так:

Головний товар:

```
<offer id="EFDELTA1300">
```

Модифікація:

```
<offer id="EFDELTA1300-EU" group_id="EFDELTA1300">
```

Це необхідно для KeepinCRM, щоб модифікації правильно прив’язувалися до головного товару.

---

# Зображення товарів

Використовуються поля:

```
images
gallery_common
```

У XML:

```
<picture>URL</picture>
```

KeepinCRM автоматично завантажує всі зображення.

---

# Обмеження кількості товарів

Для тестування використовується змінна:

```
KEEPIN_MAX_ITEMS
```

Наприклад:

```
KEEPIN_MAX_ITEMS=50
```

Для повного експорту:

```
KEEPIN_MAX_ITEMS=0
```

---

# Налаштування імпорту у KeepinCRM

У налаштуваннях імпорту KeepinCRM потрібно вказати **Теги з XML файлу**.

| Назва тегу | Поле               |
| ---------- | ------------------ |
| vendorCode | Артикул (sku)      |
| name       | Назва (title)      |
| price      | Ціна (price)       |
| url        | URL (link_url)     |
| parent_id  | Parent (parent_id) |

Це дозволяє KeepinCRM правильно обробляти структуру товарів та модифікацій.

---

# Діагностика

Файл:

```
BUILD.txt
```

містить інформацію про останню генерацію XML.

Приклад:

```
Updated: 2026-03-04
Products exported: 324
Offers generated: 324
```

---

# Ручний запуск

Workflow можна запустити вручну у GitHub:

```
Actions → build_keepin_xml → Run workflow
```

---

# Підсумок

Ця система забезпечує автоматичну синхронізацію каталогу:

```
Horoshop → KeepinCRM
```

Оновлення XML відбувається автоматично **кожні 6 годин**, тому KeepinCRM завжди отримує актуальний каталог товарів.

