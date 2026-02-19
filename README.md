## 🔹 [Rack](#rack)

## 🔹 [ActiveRecord](#activerecord)

## 🔹 [Transactions](#transactions)

## 🔹 [eager loading](#eager-loading)

## 🔹 [Autoloading](#autoloading)

## 🔹 [Sidekiq](#sidekiq)

## 🔹 [Cache](#cache)

## 🔹 [Thread](#thread)

## 🔹 [Bundler](#bundler)

## 🔹 [DB indexes](#db-indexes)

## 🔹 [JOIN (INNER / LEFT / RIGHT)](#join)

---

# Rack

> <b>Web Server (Nginx/Apache)</b> — приймає HTTP-запит, може віддати статичні файли або проксувати запит до application server

> <b>Application Server (Puma/Webrick)</b> — запускає Ruby-процеси та передає запит у Rack

> <b>Rack</b> — це мінімальний, модульний інтерфейс між веб-серверами (Puma) та Ruby-фреймворками (Rails)

> <b>Rack Middleware stack</b> — ланцюжок middleware, що виконується до/після основного додатку.

```
Client (Request)
↓
Web Server (Nginx/Apache)
↓
Application Server (Puma/Unicorn)
↓
Rack Middleware (stack)
↓
Rails Application (Routes > Controller → Model → etc.)
↓
Rack Middleware (stack)
```

1. Браузер надсилає:

```
GET /users/1
```

2. Nginx приймає запит і передає його в Puma.
3. Puma викликає Rack application:

```
call(env)
```

> env — це Hash з:

```
REQUEST_METHOD
PATH_INFO
HTTP headers
rack.input
```

4. Middleware stack обробляє запит:

```
cookies
session
CSRF
logging
```

5. Router знаходить відповідність:

```
get "/users/:id", to: "users#show"
```

6. Виконується:

```
class UsersController < ApplicationController
  def show
    @user = User.find(params[:id])
    render json: @user
  end
end
```

7. Повертається Rack response:

```
[200, { "Content-Type" => "application/json" }, ["{...}"]]
```

# ActiveRecord

> <b>ORM (Object Relational Mapping)</b> — мапінг таблиць БД у Ruby-об’єкти.

> <b>Lazy loading</b> — виконання SQL тільки при необхідності.

> <b>Arel</b> — внутрішній SQL builder.

> <b>Relation</b> — chainable query object.

1. User наслідується від ActiveRecord::Base.

2. where повертає ActiveRecord::Relation:

```
relation = User.where(email: "test@example.com")
```

SQL ще НЕ виконано.

3. Створюється AST (Abstract Syntax Tree) через Arel.

4. SQL генерується лише при:

```
relation.to_a
relation.first
relation.each
```

5. Генерується SQL:

```
SELECT "users".*
FROM "users"
WHERE "users"."email" = 'test@example.com'
```

6. Adapter виконує запит.

7. Результат мапиться в Ruby-об’єкти.

# Transactions

> <b>Transaction</b> — це послідовність операцій з базою даних, які розглядаються як єдине логічне ціле. Ці операції можуть включати вставку (INSERT), оновлення (UPDATE) або видалення (DELETE) записів у базі даних.

> <b>ACID (Atomicity — атомарність, Consistency — узгодженість, Isolation — ізольованість, Durability — довговічність)</b> — це властивості, що гарантують цілісність даних з моменту BEGIN до COMMIT або ROLLBACK.

```
ActiveRecord::Base.transaction do
  user.save!
  account.save!
end
```

1. Rails виконує:

```
BEGIN;
```

2. Виконуються INSERT/UPDATE.

3. Відміняє всі зміни, зроблені в межах транзакції, якщо виникає exception:

```
ROLLBACK;
```

4. Якщо все успішно:

```
COMMIT;
```

# eager loading

> <b>Eager loading</b> — попереднє завантаження асоціацій.

> <b>N+1 problem</b> — 1 запит на основну таблицю + N запитів на асоціації.

```
users = User.all
users.each do |user|
  puts user.posts.count
end
```

Якщо 100 користувачів → 101 запит

> User.preload(:posts)

```
SELECT * FROM users;
SELECT * FROM posts WHERE user_id IN (...);
```

2 запити

> User.eager_load(:posts)

```
SELECT users.*, posts.*
FROM users
LEFT OUTER JOIN posts ON posts.user_id = users.id;
```

> User.includes(:posts)

2 запити

> User.includes(:posts).where(posts: ...)

LEFT OUTER JOIN

# Autoloading

> <b>Zeitwerk</b> — це система автозавантаження класів і модулів, яка з’явилася в Rails 6 і стала стандартним loader-ом. Вона відповідає за те, щоб класи та модулі підключалися автоматично, без потреби робити require вручну.

> <b>Constant resolution</b> — механізм пошуку констант.

```
app/models/admin/user.rb

module Admin
  class User
  end
end
```

> Admin::User.first

1. Виклик константи Admin::User. На момент виконання константа ще не визначена.
3. Якщо Ruby бачить (<b>constant lookup</b>), що є посилання на константу Admin::User. Zeitwerk перетворює ім’я константи на шлях до файлу, виходячи з конвенції іменування.
4. Zeitwerk робить require для знайденого файлу. Після підвантаження файл визначає потрібну константу
5. Коли Ruby не знаходить константу, викликається метод const_missing. Zeitwerk перехоплює цей виклик.

# Sidekiq

> <b>Sidekiq</b> — це фреймворк для асинхронної обробки задач у Ruby. Він дозволяє запускати фонові Job-и, тобто обробляти важкі або тривалі операції поза основним запитом веб-сервера, не блокуючи його.

> <b>Redis</b> — це високопродуктивна in-memory база даних типу key-value

> <b>Worker</b> — Ruby-клас, який містить логіку виконання задачі. Має метод perform.

> <b>Job</b> — екземпляр задачі, який буде виконаний.

> <b>Queue</b> — черга в Redis, де зберігаються Job-и до виконання.

> <b>Sidekiq/Worker process</b> — процес Sidekiq, який десеріалізує Job з черги і виконує.

1. Створюємо Worker (клас)

```
class EmailWorker
  include Sidekiq::Worker

  def perform(user_id)
    UserMailer.welcome(user_id).deliver_now
  end
end
```

2. Створюємо Job

```
EmailWorker.perform_async(user.id)
```

3. Sidekiq серіалізує Job у JSON

```
{ "class": "HardJobWorker", "args": [1], ... }
```

4. Sidekiq передає Job у Redis-чергу (queue). Job додається у список (list) Redis

```
queue:default → [{ "class": "EmailWorker", "args": [1], ... }, ...]
```

4. Sidekiq/Worker process десеріалізує та виконує Job.

```
worker = HardJobWorker.new
worker.perform(1)
```

Може працювати багатопоточно, обробляючи декілька Job-ів одночасно.

6. Якщо perform повертає помилку

> Sidekiq робить <b>retry</b> (не видаляє його з черги, а намагається повторити виконання через певний час) з <b>exponential backoff</b> (час між повторними виконаннями росте експоненційно).

# Cache

> <b>Cache</b> — це тимчасове збереження результатів (MemoryStore, RedisCacheStore, MemCacheStore)

> <b>Cache key</b> — унікальний ідентифікатор кешованих даних.

> <b>Marshal</b> — це стандартний модуль Ruby для серіалізації та десеріалізації Ruby-об’єктів. Він перетворює будь-який Ruby-об’єкт у байтовий потік

<b>Fragment caching</b> (кеш частини view)

```
<% cache @user do %>
  <%= render @user %>
<% end %>
```

Cache key (включає ID користувача-updated_at):

```
users/1-20260219120000
```

<b>Low-level caching</b> (кеш довільних даних)

```
Rails.cache.fetch("expensive_query", expires_in: 1.hour) do
  User.expensive_method
end
```

1. Серіалізує Ruby-об’єкт

- Якщо User.expensive_method повертає масив хешів, Rails повинен перетворити його в байти або рядок, щоб зберегти у бекенд.
- Використовується </b>Marshal</b> (за замовчуванням) або інколи JSON для простих типів.

2. Викликає метод write бекенду

- Для Redis → `SET key value EX seconds`
- Для Memcached → `set key value expiration`
- Для MemoryStore → зберігає в хеш у пам’яті процесу Rails

# Thread

> <b>Thread</b> — це окремий потік виконання в межах одного процесу Ruby.

> <b>Thread pool</b> — набір потоків, готових виконувати задачі (наприклад, Puma при старті запускає N потоків, які чекають на HTTP request-и).

> <b>Connection pool</b> — це пул готових з’єднань до бази даних, з якого Thread може тимчасово брати "connection" для SQL запитів та повертати. У Rails він реалізований у ActiveRecord.

```
connection = ActiveRecord::Base.connection_pool.checkout
ActiveRecord::Base.connection_pool.checkin(connection)
```

Створюємо 10 потоків, кожен з яких робить свій HTTP-запит

```
threads = 10.times.map do
  Thread.new do
    response = Net::HTTP.get(URI("https://example.com"))
    puts response.size
  end
end

# чекаємо, поки всі Thread-и закінчать, перед тим як продовжити виконання програми
threads.each(&:join)
```

Rails + Puma

```
HTTP Request
↓
Puma Thread Pool
↓
Thread
↓
Controller
↓
Model / ActiveRecord → ActiveRecord Connection Pool → Database (SQL) → DB connection повертається в ActiveRecord Connection Pool
↓
View / Template
↓
HTTP Response
↓
Thread повертається в Puma Thread Pool
```

> Для <b>CPU-bound (обчислення всередині CPU - стиснення файлів, шифрування, складні алгоритми)</b> задач багатопоточність у <b>MRI (Matz's Ruby Interpreter)</b> не дає прискорення, бо <b>GIL (Global Interpreter Lock)</b> гарантує, що у MRI Ruby тільки один Thread одночасно виконує Ruby-код.

> Для <b>I/O-bound (очікування зовнішніх ресурсів - HTTP-запити, DB-запити, Redis)</b> багатопоточність дозволяє ефективну паралельність, Thread може чекати на I/O, а інші thread-и продовжують працювати.

# Bundler

> <b>Bundler</b> — менеджер залежностей.

> <b>Gem</b> — Ruby-бібліотека.

> <b>Gemfile</b> — файл, де описані залежності додатку.

> <b>Gemfile.lock</b> — файл, який фіксує конкретні версії гемів.

> <b>Dependency resolution</b> — процес підбору сумісних версій гемів.

1. Описує потрібні геми Gemfile
2. Виконуємо команду

```
bundle install
```

3. Bundler завантажує геми:

- Перевіряє, чи є вони локально (GEM_PATH)
- Якщо нема — завантажує потрібні версії з RubyGems.org

4. Bundler будує дерево залежностей:

- rails → activerecord, actionpack ...
- sidekiq → redis, connection_pool ...

5. Bundler підбирає сумісні версії всіх залежностей (dependency resolution).

6. Bundler зберігає конкретні версії в Gemfile.lock

# DB indexes

> <b>Index</b> — структура даних у базі, яка прискорює пошук рядків.

> <b>B-tree (self-balancing tree)</b> — найпоширеніший тип індексу у PostgreSQL/MySQL, який дозволяє виконувати пошук за логарифмічний час.

```
      [M]        <-  внутрішній вузол
     /   \
[A..L]   [N..Z]  <-  листові вузли (ключі + посилання на рядки)
```

> <b>Query planner</b> — механізм бази даних, який вирішує, який індекс (або повне сканування) використовувати для запиту.

1. Створюємо індекс через міграцію
2. PostgreSQL будує B-tree для колонки
3. Коли виконується запит

- Query planner перевіряє наявність індексу → пошук за логарифмічний час O(log n)
- Без індексу → повне сканування таблиці O(n)

> Індекси прискорюють SELECT запити.

> Але кожен INSERT/UPDATE/DELETE, який змінює проіндексовані колонки, трохи сповільнюється, бо індекси потрібно оновлювати.

# JOIN

> <b>JOIN</b> — механізм SQL для об’єднання рядків з кількох таблиць за умовою.

```
users:          orders:

1  Alice        1 user_id=1
2  Bob          2 user_id=1
3  Carol
```

1. INNER JOIN

> Повертає лише ті рядки, де є співпадіння в обох таблицях.

```
SELECT users.*, orders.*
FROM users
INNER JOIN orders ON orders.user_id = users.id;
```

```
Alice + Order 1
Alice + Order 2
```

2. LEFT JOIN

> Повертає всі рядки з лівої таблиці + NULL для правої, якщо немає співпадіння.

```
SELECT users.*, orders.*
FROM users
LEFT JOIN orders ON orders.user_id = users.id;
```

```
Alice + Order 1
Alice + Order 2
Bob   + NULL
Carol + NULL
```

3. RIGHT JOIN

> Повертає всі рядки з правої таблиці + NULL для лівої, якщо немає співпадіння.

```
SELECT users.*, orders.*
FROM users
RIGHT JOIN orders ON orders.user_id = users.id;
```

```
Alice + Order 1
Alice + Order 2
```

4. FULL JOIN

> Повертає всі рядки з правої таблиці та лівої, NULL там де немає співпадіння.

```
SELECT *
FROM users
FULL OUTER JOIN orders ON orders.user_id = users.id;
```

```
Alice + Order 1
Alice + Order 2
Bob   + NULL
Carol + NULL
```
