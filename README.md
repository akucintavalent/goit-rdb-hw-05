# goit-rdb-hw-05: CTE та Materialized View

Домашнє завдання 5 з курсу «Реляційні бази даних для Data Science». У ноутбуці [`goit_rdb_hw_05.ipynb`](goit_rdb_hw_05.ipynb) на даних Olist будується набір ознак клієнтів для ML: typed-схема в PostgreSQL, CTE-пайплайн, рекурсивний CTE, Materialized View з `REFRESH ... CONCURRENTLY` і порівняння планів `EXPLAIN (ANALYZE, BUFFERS)`.

## Дані

Джерело: [Olist Brazilian E-Commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) (Kaggle, `olistbr/brazilian-ecommerce`).

Використано п'ять файлів, вони також лежать у репозиторії в `data/olist/`:

- `olist_customers_dataset.csv`
- `olist_orders_dataset.csv`
- `olist_order_items_dataset.csv`
- `olist_products_dataset.csv`
- `olist_sellers_dataset.csv`

Отримати файли можна двома способами (обидва є в другій комірці ноутбука):

- **Варіант A (за замовчуванням)** — `kagglehub.dataset_download('olistbr/brazilian-ecommerce')`. У Colab має бути налаштований доступ до Kaggle.
- **Варіант B** — CSV із цього репозиторію. Склонуйте репозиторій у `/content/goit-rdb-hw-05`, закоментуйте рядки варіанта A і розкоментуйте `dataset_dir = '/content/goit-rdb-hw-05/data/olist'`.

## Raw- і typed-шар

- **Raw-шар** — таблиці `olist_*_raw`. CSV завантажуються в них через `pandas.to_sql` без змін, це лише проміжний крок.
- **Typed-шар** — таблиці `olist_customers`, `olist_orders`, `olist_order_items`, `olist_products`, `olist_sellers` з типами колонок, первинними й зовнішніми ключами, `NOT NULL` і `CHECK`. Дані переносяться з raw-шару через `INSERT ... SELECT`. Колонки `product_name_lenght` / `product_description_lenght` з CSV у typed-схемі перейменовано на `..._length`. Усі наступні запити працюють лише з typed-таблицями.

Після перенесення кількість рядків у raw- і typed-таблицях однакова (customers 99 441, orders 99 441, order_items 112 650, products 32 951, sellers 3 095).

## CTE-stages

| Stage | CTE | Що робить |
|---|---|---|
| 0 | `reference_date` | Стабільна дата відліку: день після останнього замовлення в датасеті |
| 1 | `stage1_orders_total` | Один рядок на замовлення та його сума (`price + freight_value`) |
| 2 | `stage2_rmf` | RFM на клієнта: `recency_days`, `frequency`, `monetary` |
| 3 | `stage3_cohort` | `cohort_quarter` за датою першого замовлення |
| 4 | `stage4_customer_30d_averages` | Rolling 30-day AOV для останнього замовлення клієнта |
| 5 | `stage5_cancellation_rate` | Частка скасованих замовлень клієнта |
| 6 | `stage6_customer_revenue_by_categories` | Виручка клієнта за категоріями товарів |
| 7 | `stage7_preferred_categories` | Top-1 категорія за виручкою (нічия — за назвою) |

Повний результат пайплайна збережено в `mv_olist_customer_features` з унікальним індексом на `customer_id`. Для рекурсивного CTE створено синтетичну таблицю `hw5_referrals`, бо в Olist реферальних даних немає.

## Запуск

1. Відкрийте `goit_rdb_hw_05.ipynb` у Google Colab.
2. Оберіть спосіб отримання даних (варіант A або B вище).
3. Виконайте «Runtime → Restart session and run all». Перша комірка встановлює залежності (`pgserver`, `psycopg2-binary`, `sqlalchemy`, `pandas`, `kagglehub`, `sqlparse`), запускає PostgreSQL через `pgserver` і виводить його версію.
