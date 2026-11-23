## 🔹 [Duck Typing](#duck-typing)

## 🔹 [MRI (Matz's Ruby Interpreter)](#mri)

## 🔹 [Rack](#rack)

## 🔹 [ActiveRecord](#activerecord)

## 🔹 [Transactions](#transactions)

## 🔹 [Eager loading](#eager-loading)

## 🔹 [Autoloading](#autoloading)

## 🔹 [Sidekiq](#sidekiq)

## 🔹 [Cache](#cache)

## 🔹 [Thread](#thread)

## 🔹 [Mutex](#mutex)

## 🔹 [Bundler](#bundler)

## 🔹 [DB indexes](#db-indexes)

## 🔹 [JOIN (INNER / LEFT / RIGHT / FULL)](#join)

## 🔹 [Pessimistic Locking](#pessimistic-locking)

## 🔹 [Deadlock](#deadlock)

## 🔹 [Memory leak](#memory-leak)

## 🔹 [CSRF, XSS, SQL injection, Strong Parameters](#security)

---

# Duck Typing

> "If it walks like a duck and quacks like a duck, then it's a duck."

<b>Duck typing</b> — підхід динамічної типізації, за якого важливий не клас об'єкта, а його поведінка, тобто методи, на які він відповідає.

Якщо код викликає метод, якого об'єкт не реалізує, виникає `NoMethodError`. Перевірити заздалегідь можна через `respond_to?`.

```ruby
[File.open("log.txt", "w"), StringIO.new, $stdout].each { |io| io.puts "hello" }
```

# MRI

> <b>MRI (Matz's Ruby Interpreter) або CRuby</b> — стандартна реалізація мови Ruby, написана на C, яка компілює Ruby-код у байткод і виконує його через віртуальну машину YARV. Використовує GVL (Global VM Lock, часто його називають GIL — Global Interpreter Lock) для керування потоками.

> <b>YARV (Yet Another Ruby VM)</b> — основна віртуальна машина Ruby (з версії 1.9), яка виконує байткод, згенерований MRI, забезпечуючи виконання програм Ruby.

> <b>Lexer (лексичний аналізатор)</b> — компонент, який бере Ruby-код у вигляді тексту і розбиває його на <b>токени</b>.

> <b>Parser</b> бере токени і будує <b>AST (Abstract Syntax Tree)</b> — дерево, яке відображає структуру

```
Assignment
├─ Left: identifier 'user'
└─ Right: MethodCall
    ├─ Receiver: Constant 'User'
    ├─ Method: 'find'
    └─ Argument: Integer 1
```

1. Парсинг

- Ruby-файл (.rb) → Lexer → токени → Parser → AST (Abstract Syntax Tree)

```ruby
user = User.find(1)
```

- → токени: `user, =, User, ., find, (, 1, )`
- → AST: Assignment → MethodCall → Arguments

2. Компіляція в байткод (YARV instructions)

- AST трансформується в набір інструкцій VM:

```
opt_getconstant_path    <User>
putobject_INT2FIX_1_
opt_send_without_block  <mid:find, argc:1>
setlocal_WC_0           user
```

Подивитися байткод: `ruby --dump=insns -e 'user = User.find(1)'`

3. Виконання в Ruby VM

- Стекова машина обробляє інструкції
- Управління передається від інструкції до інструкції
- Локальні змінні живуть на stack frame, об’єкти у heap

4. GC (Garbage Collector)

- Mark-and-sweep + Generational + Incremental GC (з Ruby 2.7 також доступний compaction)
- Періодично видаляє невикористані об’єкти
- Важливо для Rails, бо великі ActiveRecord-об’єкти можуть швидко заповнювати heap

5. C API / Native extensions

- Gems з C-розширеннями (pg, Nokogiri) виконують код нативно, без інтерпретації байткоду → швидше. Під час довгих I/O-операцій C-розширення можуть відпускати GVL

# Rack

> <b>Web Server (Nginx/Apache)</b> — приймає HTTP-запит, може віддати статичні файли або проксувати запит до application server

> <b>Application Server (Puma/Unicorn)</b> — запускає Ruby-процеси та передає запит у Rack

> <b>Rack</b> — це мінімальний, модульний інтерфейс між application-серверами (Puma) та Ruby-фреймворками (Rails)

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
Rails Application (Routes → Controller → Model → etc.)
↓
Rack Middleware (stack)
↓
Client (Response)
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
ActionDispatch::Cookies
ActionDispatch::Session::CookieStore
Rails::Rack::Logger
...
```

Переглянути стек: `bin/rails middleware`. CSRF-перевірка — це не middleware, а before_action у контролері (`protect_from_forgery`).

5. Router знаходить відповідність:

```ruby
get "/users/:id", to: "users#show"
```

6. Виконується:

```ruby
class UsersController < ApplicationController
  def show
    @user = User.find(params[:id])
    render json: @user
  end
end
```

7. Повертається Rack response:

```ruby
[200, { "Content-Type" => "application/json" }, ["{...}"]]
```

# ActiveRecord

> <b>ORM (Object Relational Mapping)</b> — мапінг таблиць БД у Ruby-об’єкти.

> <b>Lazy evaluation</b> — Relation виконує SQL тільки тоді, коли потрібні дані.

> <b>Arel</b> — внутрішній SQL builder.

> <b>Relation</b> — chainable query object.

1. User наслідується від ActiveRecord::Base.

2. where повертає ActiveRecord::Relation:

```ruby
relation = User.where(email: "test@example.com")
```

SQL ще НЕ виконано.

3. Створюється AST (Abstract Syntax Tree) через Arel.

4. SQL виконується лише при:

```ruby
relation.to_a
relation.each
relation.load
```

`relation.first` не завантажує весь relation, а робить окремий запит з `ORDER BY "users"."id" ASC LIMIT 1`.

5. Генерується SQL:

```sql
SELECT "users".*
FROM "users"
WHERE "users"."email" = $1   -- значення передається як bind-параметр
```

6. Adapter виконує запит.

7. Результат мапиться в Ruby-об’єкти.

# Transactions

> <b>Transaction</b> — це послідовність операцій з базою даних, які розглядаються як єдине логічне ціле. Ці операції можуть включати вставку (INSERT), оновлення (UPDATE) або видалення (DELETE) записів у базі даних.

> <b>ACID (Atomicity — атомарність, Consistency — узгодженість, Isolation — ізольованість, Durability — довговічність)</b> — це властивості, що гарантують надійність транзакцій: зміни застосовуються повністю або не застосовуються зовсім, а після COMMIT зберігаються навіть у разі збою.

```ruby
ActiveRecord::Base.transaction do
  user.save!
  account.save!
end
```

1. Rails виконує:

```sql
BEGIN;
```

2. Виконуються INSERT/UPDATE.

3. Відміняє всі зміни, зроблені в межах транзакції, якщо з блоку вилітає exception:

```sql
ROLLBACK;
```

- `save` (без `!`) при помилці валідації повертає `false`, а не кидає exception, тому відкату не буде. У транзакціях використовуйте `save!`/`create!`.
- `raise ActiveRecord::Rollback` відкочує транзакцію, але не прокидається за межі блоку.

4. Якщо все успішно:

```sql
COMMIT;
```

# Eager loading

> <b>Eager loading</b> — попереднє завантаження асоціацій.

> <b>N+1 problem</b> — 1 запит на основну таблицю + N запитів на асоціації.

```ruby
def index
  posts = Post.all

  render json: posts.map { |post|
    {
      id: post.id,
      title: post.title,
      author_name: post.user.name,
      comments: post.comments.map { |comment|
        {
          id: comment.id,
          content: comment.content,
          author_name: comment.user.name
        }
      }
    }
  }
end
```

> <b>Post.preload(:user, comments: :user)</b>

- Pros: avoids row duplication, memory-efficient for large datasets
- Cons: cannot filter/sort on associations easily (no JOIN)

Rails бере id з уже завантажених записів і підставляє їх у `IN (...)`:

```sql
SELECT "posts".* FROM "posts";

SELECT "users".* FROM "users" WHERE "users"."id" IN (1, 2, 3);

SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN (10, 11, 12);

SELECT "users".* FROM "users" WHERE "users"."id" IN (4, 5, 6);
```

4 запити

> <b>Post.eager_load(:user, comments: :user)</b>

- Pros: allows filtering/sorting on associations, single query
- Cons: can cause row explosion (posts repeated per comment), more memory used to reconstruct objects

Спрощено (насправді Rails генерує аліаси на кшталт `t0_r0`, `t1_r0` і `users_comments`):

```sql
SELECT
  posts.id AS post_id,
  posts.title,
  users.id AS user_id,
  users.name AS user_name,
  comments.id AS comment_id,
  comments.content AS comment_content,
  comment_users.id AS comment_user_id,
  comment_users.name AS comment_user_name
FROM posts
LEFT OUTER JOIN users
  ON users.id = posts.user_id
LEFT OUTER JOIN comments
  ON comments.post_id = posts.id
LEFT OUTER JOIN users comment_users
  ON comment_users.id = comments.user_id;
```

> <b>Post.includes(:user, comments: :user)</b>

Rails сам обирає стратегію. За замовчуванням це `preload` (4 запити).

> <b>Post.includes(:user, comments: :user).where(users: { name: "Bob" })</b>

Умова на асоційовану таблицю → `includes` перемикається на `eager_load` (один запит з LEFT OUTER JOIN). Для рядкових умов потрібно явно додати `.references(:users)`.

`Post.includes(...).where(posts: ...)` (умова на основну таблицю) лишається `preload`.

---

Running raw SQL

```ruby
sql = <<~SQL
  SELECT ...
  FROM ...
  JOIN ... ON ... = ...;
SQL

rows = ActiveRecord::Base.connection.exec_query(sql)
```

Final, optimized version

```ruby
def index
  posts = Post.preload(:user, comments: :user).select(:id, :title, :user_id).order(id: :desc).limit(50)

  render json: posts.as_json(
    only: [:id, :title],
    include: {
      user: { only: [:name] },
      comments: {
        only: [:id, :content],
        include: {
          user: { only: [:name] }
        }
      }
    }
  )
end
```

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

1. Під час завантаження Rails Zeitwerk обходить autoload paths (`app/models`, `app/controllers`...) і за конвенцією іменування перетворює шляхи на імена констант: `admin/user.rb` → `Admin::User`.
2. Для кожної константи Zeitwerk реєструє `Module#autoload`, наприклад `Admin.autoload(:User, "app/models/admin/user.rb")`.
3. Код звертається до `Admin::User`. Ruby (<b>constant lookup</b>) бачить зареєстрований autoload і сам робить `require` файлу.
4. Файл визначає константу, і виконання продовжується.

`const_missing` використовував класичний autoloader (Rails до 6). Zeitwerk на нього не покладається. У production (`config.eager_load = true`) Zeitwerk завантажує всі файли одразу при старті.

# Sidekiq

> <b>Sidekiq</b> — це фреймворк для асинхронної обробки задач у Ruby. Він дозволяє запускати фонові Job-и, тобто обробляти важкі або тривалі операції поза основним запитом веб-сервера, не блокуючи його.

> <b>Redis</b> — це високопродуктивна in-memory база даних типу key-value

> <b>Worker / Job class</b> — Ruby-клас, який містить логіку виконання задачі. Має метод perform.

> <b>Job</b> — екземпляр задачі, який буде виконаний.

> <b>Queue</b> — черга в Redis, де зберігаються Job-и до виконання.

> <b>Sidekiq/Worker process</b> — процес Sidekiq, який десеріалізує Job з черги і виконує.

1. Створюємо Worker (клас)

```ruby
class EmailWorker
  include Sidekiq::Job # з Sidekiq 6.3; Sidekiq::Worker лишився як аліас

  def perform(user_id)
    UserMailer.welcome(user_id).deliver_now
  end
end
```

2. Створюємо Job

```ruby
EmailWorker.perform_async(user.id)
```

3. Sidekiq серіалізує Job у JSON

```
{ "class": "EmailWorker", "args": [1], ... }
```

4. Sidekiq передає Job у Redis-чергу (queue). Job додається у список (list) Redis

```
queue:default → [{ "class": "EmailWorker", "args": [1], ... }, ...]
```

5. Sidekiq/Worker process забирає Job з черги (BRPOP), десеріалізує та виконує.

```ruby
worker = EmailWorker.new
worker.perform(1)
```

Може працювати багатопоточно, обробляючи декілька Job-ів одночасно.

6. Якщо perform кидає exception

> Sidekiq переносить Job у <b>Retry set</b> (sorted set у Redis) і повторює виконання з <b>exponential backoff</b> (час між спробами росте експоненційно). За замовчуванням 25 спроб, приблизно за 21 день. Після цього Job потрапляє в <b>Dead set</b>.

Тому job має бути <b>ідемпотентним</b>: повторне виконання не повинно ламати дані. Аргументи — тільки прості JSON-типи (id, а не ActiveRecord-об'єкт).

# Cache

> <b>Cache</b> — це тимчасове збереження результатів (MemoryStore, RedisCacheStore, MemCacheStore)

> <b>Cache key</b> — унікальний ідентифікатор кешованих даних.

> <b>Marshal</b> — це стандартний модуль Ruby для серіалізації та десеріалізації Ruby-об’єктів. Він перетворює будь-який Ruby-об’єкт у байтовий потік

<b>Fragment caching</b> (кеш частини view)

```erb
<% cache @user do %>
  <%= render @user %>
<% end %>
```

Cache key будується з моделі та digest-у шаблону:

```
views/users/_user:<template_digest>/users/1
```

З Rails 5.2 (`cache_versioning`) `updated_at` не входить у ключ, а зберігається окремо як cache version (`20260219120000000000`). Коли запис оновлюється, версія не збігається, і кеш вважається застарілим.

<b>Low-level caching</b> (кеш довільних даних)

```ruby
Rails.cache.fetch("expensive_query", expires_in: 1.hour) do
  User.expensive_method
end
```

`fetch` спочатку читає ключ. Якщо дані є (cache hit), блок не виконується. Якщо немає (cache miss), виконується блок, а результат записується:

1. Серіалізує Ruby-об’єкт

- Якщо User.expensive_method повертає масив хешів, Rails повинен перетворити його в байти або рядок, щоб зберегти у бекенд.
- Використовується <b>Marshal</b> (за замовчуванням) або інколи JSON для простих типів.

2. Викликає метод write бекенду

- Для Redis → `SET key value EX seconds`
- Для Memcached → `set key value expiration`
- Для MemoryStore → зберігає в хеш у пам’яті процесу Rails

# Thread

> <b>Thread</b> — це окремий потік виконання в межах одного процесу Ruby.

> <b>Thread pool</b> — набір потоків, готових виконувати задачі (наприклад, Puma тримає від min до max потоків, які обробляють HTTP request-и).

> <b>Connection pool</b> — це пул готових з’єднань до бази даних, з якого Thread може тимчасово брати "connection" для SQL запитів та повертати. У Rails він реалізований у ActiveRecord.

```ruby
ActiveRecord::Base.connection_pool.with_connection do |connection|
  connection.execute("SELECT 1")
end # з'єднання автоматично повертається в пул
```

Розмір пулу (`pool` у database.yml) має бути не меншим за кількість потоків Puma, інакше потоки чекатимуть на з'єднання (`ActiveRecord::ConnectionTimeoutError`).

Створюємо 10 потоків, кожен з яких робить свій HTTP-запит

```ruby
require "net/http"

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

> Для <b>CPU-bound (обчислення всередині CPU - стиснення файлів, шифрування, складні алгоритми)</b> задач багатопоточність у <b>MRI (Matz's Ruby Interpreter)</b> не дає прискорення, бо <b>GVL (Global VM Lock, він же GIL)</b> гарантує, що у MRI Ruby тільки один Thread одночасно виконує Ruby-код.

> Для <b>I/O-bound (очікування зовнішніх ресурсів - HTTP-запити, DB-запити, Redis)</b> багатопоточність дозволяє ефективну паралельність, Thread може чекати на I/O, а інші thread-и продовжують працювати (під час I/O GVL відпускається).

# Mutex

> <b>Mutex</b> — це механізм синхронізації, який дозволяє уникнути одночасного доступу до критичної секції коду з кількох потоків (Threads).

```ruby
file_path = "log.txt"
mutex = Mutex.new

threads = 5.times.map do |i|
  Thread.new do
    mutex.synchronize do
      File.open(file_path, "a") do |f|
        f.puts "Thread #{i} пише у файл"
        sleep(0.5) # симуляція тривалої операції
      end
    end
  end
end

threads.each(&:join)
```

1. Mutex.new – створює об’єкт mutex, який буде контролювати доступ.
2. `mutex.synchronize do ... end` – блокує критичну секцію, щоб тільки один потік міг відкривати та писати у файл одночасно.
3. Без Mutex декілька потоків могли б одночасно відкрити файл для додавання ("a"), і дані могли б змішатися.

Якщо файл великий або запис часто повторюється, краще використовувати чергу запису або Redis/Sidekiq job, щоб не блокувати потоки надовго.

# Bundler

> <b>Bundler</b> — менеджер залежностей.

> <b>Gem</b> — Ruby-бібліотека.

> <b>Gemfile</b> — файл, де описані залежності додатку.

> <b>Gemfile.lock</b> — файл, який фіксує конкретні версії гемів.

> <b>Dependency resolution</b> — процес підбору сумісних версій гемів.

1. Описуємо потрібні геми в Gemfile
2. Виконуємо команду

```
bundle install
```

3. Якщо Gemfile.lock уже є і відповідає Gemfile — Bundler бере версії з нього, без resolution.

4. Інакше Bundler будує дерево залежностей:

- rails → activerecord, actionpack ...
- sidekiq → redis-client, connection_pool ...

5. Bundler підбирає сумісні версії всіх залежностей (dependency resolution).

6. Bundler встановлює геми:

- Перевіряє, чи є вони локально (GEM_PATH)
- Якщо нема — завантажує потрібні версії з RubyGems.org

7. Bundler зберігає конкретні версії в Gemfile.lock

# DB indexes

> <b>Index</b> — структура даних у базі, яка прискорює пошук рядків.

> <b>B-tree (self-balancing tree)</b> — найпоширеніший тип індексу у PostgreSQL/MySQL (в InnoDB — B+tree), який дозволяє виконувати пошук за логарифмічний час.

```
      [M]        <-  внутрішній вузол
     /   \
[A..L]   [N..Z]  <-  листові вузли (ключі + посилання на рядки)
```

> <b>Query planner</b> — механізм бази даних, який вирішує, який індекс (або повне сканування) використовувати для запиту.

1. Створюємо індекс через міграцію
2. PostgreSQL будує B-tree для колонки
3. Коли виконується запит

- Query planner оцінює вартість і, якщо індекс вигідний, використовує його → пошук за O(log n)
- Без індексу → повне сканування таблиці (Seq Scan) O(n)
- Planner може обрати Seq Scan навіть за наявності індексу (мала таблиця, низька селективність). Перевірити: `EXPLAIN ANALYZE`

> Індекси прискорюють SELECT запити.

> Але кожен INSERT/UPDATE/DELETE, який змінює проіндексовані колонки, трохи сповільнюється, бо індекси потрібно оновлювати.

```
rails generate migration AddIndexToUsersEmailWithDeletedAtFilter
```

```ruby
class AddIndexToUsersEmailWithDeletedAtFilter < ActiveRecord::Migration[7.0]
  disable_ddl_transaction! # потрібно для algorithm: :concurrently

  def change
    # Partial index: only for rows that are not soft-deleted (PostgreSQL; MySQL не підтримує partial indexes)
    add_index :users, :email,
              unique: true,
              where: "deleted_at IS NULL",
              name: "index_users_on_email_not_deleted",
              algorithm: :concurrently # не блокує запис у таблицю під час побудови індексу
  end
end
```

# JOIN

> <b>JOIN</b> — механізм SQL для об’єднання рядків з кількох таблиць за умовою.

```
users:          orders:

1  Alice        1 user_id=1
2  Bob          2 user_id=1
3  Carol        3 user_id=NULL
```

1. INNER JOIN

> Повертає лише ті рядки, де є збігу в обох таблицях.

```sql
SELECT users.*, orders.*
FROM users
INNER JOIN orders ON orders.user_id = users.id;
```

```
Alice + Order 1
Alice + Order 2
```

2. LEFT JOIN

> Повертає всі рядки з лівої таблиці + NULL для правої, якщо немає збігу.

```sql
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

> Повертає всі рядки з правої таблиці + NULL для лівої, якщо немає збігу.

```sql
SELECT users.*, orders.*
FROM users
RIGHT JOIN orders ON orders.user_id = users.id;
```

```
Alice + Order 1
Alice + Order 2
NULL  + Order 3
```

4. FULL JOIN

> Повертає всі рядки з правої таблиці та лівої, NULL там де немає збігу.

```sql
SELECT *
FROM users
FULL OUTER JOIN orders ON orders.user_id = users.id;
```

```
Alice + Order 1
Alice + Order 2
Bob   + NULL
Carol + NULL
NULL  + Order 3
```

# Pessimistic Locking

> Механізм блокування рядків під час SELECT, щоб інші транзакції не могли їх змінити.

```ruby
User.transaction do
  # SELECT ... FOR UPDATE
  user = User.lock.find(1) # або user.with_lock { ... }
  user.balance -= 100
  user.save!
end
```

```sql
BEGIN;
SELECT * FROM users WHERE id = 1 FOR UPDATE;
UPDATE users SET balance = ... WHERE id = 1;
COMMIT;
```

1. Починається транзакція

```sql
BEGIN
```

Lock існує тільки в межах транзакції.

2. PostgreSQL виконує `SELECT ... FOR UPDATE`

- знаходить відповідний рядок
- ставить на нього row-level lock типу FOR UPDATE
- цей lock зберігається до COMMIT або ROLLBACK

Postgres не блокує таблицю для запису — блокується конкретний рядок.

3. Інша транзакція, яка спробує:

```
UPDATE
DELETE
SELECT ... FOR UPDATE / FOR NO KEY UPDATE / FOR SHARE / FOR KEY SHARE
```

→ буде чекати, поки перша транзакція завершиться.

Звичайний `SELECT` (без FOR ...) не блокується і читає останню закомічену версію рядка (MVCC).

# Deadlock

> <b>Deadlock</b> — ситуація, коли дві (або більше) транзакцій чекають одна на одну, і жодна не може завершити роботу.

> Найчастіше deadlock виникає через row-level locks, які автоматично ставляться на UPDATE/DELETE.

```
accounts:

id | balance | user_id
1  | 1000    | 1
2  | 2000    | 2
```

Транзакція A (Thread 1)

```sql
BEGIN;

-- Блокування рядка з id = 1
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- Спроба змінити рядок з id = 2
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
```

Транзакція B (Thread 2)

```sql
BEGIN;

-- Блокування рядка з id = 2
UPDATE accounts SET balance = balance - 200 WHERE id = 2;

-- Спроба змінити рядок з id = 1
UPDATE accounts SET balance = balance + 200 WHERE id = 1;
```

1. Thread 1 блокує id = 1.
2. Thread 2 блокує id = 2.
3. Thread 1 намагається заблокувати id = 2, але він уже заблокований Thread 2 → Thread 1 чекає.
4. Thread 2 намагається заблокувати id = 1, але він уже заблокований Thread 1 → Thread 2 чекає.
5. PostgreSQL виявляє цикл очікування (через `deadlock_timeout`, за замовчуванням 1 с) і скасовує одну з транзакцій з помилкою `deadlock detected`. У Rails це `ActiveRecord::Deadlocked`, після якого транзакцію можна повторити.

Як уникнути

- Фіксований порядок доступу до рядків — завжди оновлювати рядки в одному і тому ж порядку.
- Короткі транзакції — мінімізувати час блокування.
- SELECT ... FOR UPDATE у фіксованому порядку — блокувати всі потрібні рядки одразу на початку транзакції: `Account.where(id: [1, 2]).order(:id).lock`.

# Memory leak

> <b>Memory leak</b> — це ситуація, коли програма не звільняє пам’ять, яка більше не потрібна, і вона накопичується в процесі, призводячи до:

- Зростання споживання пам’яті
- Уповільнення програми
- Можливого краху через `NoMemoryError` або вбивства процесу OOM killer-ом

💡 У Ruby garbage collector (GC) звільняє невикористовувану пам’ять, але leak може виникати, якщо є посилання на об’єкти, які більше не потрібні.

1. Не використовувати глобальні та класові змінні для великих об’єктів.
2. Очищати кеш або сесії, якщо об’єкти більше не потрібні.
3. Для великих вибірок використовувати `find_each` / `find_in_batches` (ActiveRecord) або `each_slice`, не накопичуючи дані в пам'яті.
4. Профілювати пам'ять (memory_profiler, derailed_benchmarks, rack-mini-profiler) і моніторити її в production (New Relic, Datadog). Зменшити фрагментацію допомагає jemalloc.

# Security

> <b>CSRF (Cross-Site Request Forgery)</b> — це вид веб-атаки, яка змушує авторизованого користувача виконати небажані дії на сайті, де він залогінений, через запит зі стороннього сайту (наприклад, змінити пароль, переказати кошти або змінити email) без його відома. Зловмисник використовує довіру сайту до браузера жертви.

🛡 В ApplicationController:

```ruby
protect_from_forgery with: :exception
```

1. Rails генерує CSRF token
2. Кладе його в:

- session
- meta tag
- hidden field у формах

3. Перевіряє токен при кожному запиті, крім GET/HEAD: бере його з `params[:authenticity_token]` або заголовка `X-CSRF-Token` і порівнює з токеном у session.

Спрощено:

```ruby
secure_compare(unmask(params[:authenticity_token]), session[:_csrf_token])
```

Токен у формі щоразу маскується (XOR з випадковим значенням), щоб захиститися від BREACH-атак.

4. Якщо не збігається → ActionController::InvalidAuthenticityToken

> <b>Mass Assignment</b> — вразливість, коли всі параметри запиту напряму передаються в модель, і зловмисник може змінити поля, які не мали б бути доступні (наприклад, `admin`).

```ruby
User.create(params[:user]) # з Rails 4 кидає ActiveModel::ForbiddenAttributesError
```

```
user[admin]=true
```

З `accepts_nested_attributes_for` так само можна створювати/змінювати записи в пов'язаних таблицях, про що розробник міг навіть не замислюватися:

```
user[roles_attributes][0][name]=admin
```

🛡 Рішення — Strong Parameters

```ruby
def user_params
  params.require(:user).permit(:email, :password)
  # Rails 8: params.expect(user: [:email, :password])
end
```

> <b>SQL Injection</b> — це тип уразливості вебдодатків, при якому зловмисник впроваджує шкідливий SQL-код у запити до бази даних через вхідні дані (форми, URL).

Пряма інтерполяція параметрів у рядок запиту дозволяє зловмиснику маніпулювати базою даних.

```ruby
User.where("email = '#{params[:email]}'")
```

🛡 Завжди використовуйте плейсхолдери або hash-синтаксис, які автоматично екранують вхідні дані:

```ruby
User.where(email: params[:email])
User.where("email = ?", params[:email])
User.where("name LIKE ?", "%#{User.sanitize_sql_like(params[:q])}%")
```

> <b>XSS (Cross-Site Scripting)</b> — це тип вразливості веб-додатків, при якій зловмисник впроваджує шкідливий JavaScript-код на сторінку, яку переглядають інші користувачі. Цей код виконується в браузері жертви, дозволяючи викрадати сесійні файли cookie, логіни, паролі, або перенаправляти користувачів на фішингові сайти.

- html_safe

```ruby
# Контролер: отримуємо ім'я з параметрів URL
@user_name = params[:name]
```

```erb
<%# View: виводимо ім'я, позначаючи його як "безпечне" %>
<h2>Вітаємо, <%= @user_name.html_safe %></h2>
```

Якщо жертва перейде за посиланням `?name=<script>alert('Hacked!')</script>`, браузер виконає цей скрипт

- raw

```erb
<%# raw працює аналогічно html_safe, вимикаючи екранування %>
<%= raw @user.bio %>
```

Якщо в полі bio збережено шкідливий код, він виконається у кожного, хто відкриє сторінку.

- link_to

```erb
<%# Якщо користувач може сам вказати URL свого профілю %>
<%= link_to "Мій сайт", @user.website_url %>
```

Атакувальник може ввести `javascript:alert('XSS')`. Коли інший користувач натисне на посилання, виконається JS-код. Захист — валідація URL у моделі (дозволяти лише `http`/`https`).

🛡 Rails автоматично escape-ить HTML

```
&lt;script&gt;alert('Hacked')&lt;/script&gt;
```

`html_safe` і `raw` вимикають цей захист. Якщо потрібно дозволити частину HTML — використовуйте `sanitize`:

```erb
<%= sanitize @user.bio, tags: %w[b i a p], attributes: %w[href] %>
```

🛡 CSP (Content Security Policy)

Навіть якщо Rails auto-escape-ить HTML, XSS все одно можливий через:

- html_safe
- сторонні JS бібліотеки
- inline `<script>`
- CDN, який зламали
- injected third-party script

CSP — це додатковий рівень. Це HTTP header, який говорить браузеру: “Звідки дозволено завантажувати скрипти, стилі, картинки і що можна виконувати.”

```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self
    policy.script_src  :self
    policy.object_src  :none
  end
  config.content_security_policy_nonce_generator = ->(request) { SecureRandom.base64(16) }
end
```
