## 🔹 [MRI (Matz's Ruby Interpreter)](#mri)

## 🔹 [Thread](#thread)

## 🔹 [Mutex](#mutex)

## 🔹 [Memory leak](#memory-leak)

## 🔹 [Rack](#rack)

## 🔹 [CSRF, XSS, SQL injection, Strong Parameters](#security)

## 🔹 [ActiveRecord](#activerecord)

## 🔹 [Transactions](#transactions)

## 🔹 [eager loading](#eager-loading)

## 🔹 [Autoloading](#autoloading)

## 🔹 [Cache](#cache)

## 🔹 [Bundler](#bundler)

## 🔹 [Sidekiq](#sidekiq)

## 🔹 [DB indexes](#db-indexes)

## 🔹 [JOIN (INNER / LEFT / RIGHT)](#join)

## 🔹 [Pessimistic Locking](#pessimistic-locking)

## 🔹 [Deadlock](#deadlock)

---

# MRI

> <b>MRI (Matz's Ruby Interpreter) або CRuby</b> — стандартна реалізація мови Ruby, написана на C, яка компілює Ruby-код у байткод і виконує його через віртуальну машину YARV. Використовує GIL (Global Interpreter Lock) для керування потоками.

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

```
user = User.find(1)
```

- → токени: `user, =, User, ., find, (, 1, )`
- → AST: Assignment → MethodCall → Arguments

2. Компиляція в байткод (YARV instructions)

- AST трансформується в набір інструкцій VM:

```
putself
putstring "User"
send :find, 1
setlocal user
```

3. Виконання в Ruby VM

- Стекова машина обробляє інструкції
- Управління передається від інструкції до інструкції
- Локальні змінні живуть на stack frame, об’єкти у heap

4. GC (Garbage Collector)

- Mark-and-sweep + Generational GC
- Періодично видаляє невикористані об’єкти
- Важливо для Rails, бо великі ActiveRecord-об’єкти можуть швидко заповнювати heap

5. C API / Native extensions

- Gems з C (pg, Nokogiri) виконуються нативно → обробка швидше, bypass Ruby VM

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

```
SELECT "posts".* FROM "posts";

SELECT "users".* 
FROM "users" 
WHERE "users"."id" IN (SELECT DISTINCT "posts"."user_id" FROM "posts");

SELECT "comments".* 
FROM "comments" 
WHERE "comments"."post_id" IN (SELECT "posts"."id" FROM "posts");

SELECT "users".* 
FROM "users" 
WHERE "users"."id" IN (SELECT DISTINCT "comments"."user_id" FROM "comments" WHERE "comments"."post_id" IN (SELECT "posts"."id" FROM "posts"));
```

4 запити

> <b>Post.eager_load(:user, comments: :user)</b>

- Pros: allows filtering/sorting on associations, fewer queries
- Cons: can cause row explosion (posts repeated per comment), more memory used to reconstruct objects

```
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

4 запити

> <b>Post.includes(:user, comments: :user).where(posts: ...)</b>

LEFT OUTER JOIN

---

Running raw SQL

```
sql = <<-SQL
  SELECT ...
  FROM ...
  JOIN ... ON ... = ...;
SQL

rows = ActiveRecord::Base.connection.exec_query(sql)
```

Final, optimized version

```
def index
  posts = Post.includes(:user, comments: :user).select(:id, :title, :user_id).limit(50)

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

# Mutex

> <b>Mutex</b> — це механізм синхронізації, який дозволяє уникнути одночасного доступу до критичної секції коду з кількох потоків (Threads).

```
mutex = Mutex.new

threads = 5.times.map do
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

Якщо файл великий або запис часто повторюється, краще використовувати чергу запису або Redis/Sidekiq job, щоб не блокувати потоки на довго.

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

```
rails generate migration AddIndexToUsersEmailWithDeletedAtFilter
```

```
class AddIndexToUsersEmailWithDeletedAtFilter < ActiveRecord::Migration[7.0]
  def change
    # Partial index: only for rows that are not soft-deleted
    add_index :users, :email, unique: true, where: "deleted_at IS NULL", name: "index_users_on_email_not_deleted"
  end
end
```

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

# Pessimistic Locking

> Механізм блокування рядків під час SELECT, щоб інші транзакції не могли їх змінити.

```
User.transaction do
  # SELECT ... FOR UPDATE
  user = User.lock.find(1)
  user.balance -= 100
  user.save!
end
```

```
BEGIN;
SELECT * FROM users WHERE id = 1 FOR UPDATE;
UPDATE users SET balance = ... WHERE id = 1;
COMMIT;
```

1. Починається транзакція

```
BEGIN
```

Lock існує тільки в межах транзакції.

2. PostgreSQL виконує `SELECT ... FOR UPDATE`

- знаходить відповідний рядок
- ставить на нього row-level lock типу FOR UPDATE
- цей lock зберігається до COMMIT або ROLLBACK

Postgres не блокує таблицю — блокується конкретна версія рядка.

3. Інша транзакція, яка спробує:

```
UPDATE
DELETE
SELECT ... FOR UPDATE
SELECT ... FOR NO KEY UPDATE
```

→ буде чекати, поки перша транзакція завершиться.

# Deadlock

> <b>Deadlock</b> — ситуація, коли дві (або більше) транзакцій чекають одна на одну, і жодна не може завершити роботу.

> Deadlock виникає через row-level locks, які автоматично ставляться на UPDATE/DELETE.

```
id | balance | user_id
1  | 1000    | 1
2  | 2000    | 2
```

Транзакція A (Thread 1)

```
BEGIN;

-- Блокування рядка з id = 1
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- Спроба змінити рядок з id = 2
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
```

Транзакція B (Thread 2)

```
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

Як уникнути

- Фіксований порядок доступу до рядків — завжди оновлювати рядки в одному і тому ж порядку.
- Короткі транзакції — мінімізувати час блокування.
- SELECT ... FOR UPDATE — блокувати рядки явно перед оновленням.

# Memory leak

> <b>Memory leak</b> — це ситуація, коли програма не звільняє пам’ять, яка більше не потрібна, і вона накопичується в процесі, призводячи до:

- Зростання споживання пам’яті
- Уповільнення програми
- Можливого краху через OutOfMemoryError

💡 У Ruby garbage collector (GC) звільняє невикористовувану пам’ять, але leak може виникати, якщо є посилання на об’єкти, які більше не потрібні.

1. Не використовувати глобальні та класові змінні для великих об’єктів.
2. Очищати кеш або сесії, якщо об’єкти більше не потрібні.
3. Для великих циклів використовувати each_slice або обробку потоками без накопичення даних.
4. У production — моніторити пам’ять (rack-mini-profiler, NewRelic)

# Security

> <b>CSRF (Cross-Site Request Forgery)</b> — це вид веб-атаки, яка змушує авторизованого користувача виконати небажані дії на іншому сайті (наприклад, змінити пароль, переказати кошти або змінити email) без його відома. Зловмисник використовує довіру сайту до браузера жертви.

🛡 В ApplicationController:

```
protect_from_forgery with: :exception
```

1. Rails генерує CSRF token
2. Кладе його в:

- session
- meta tag
- hidden field у формах

3. Перевіряє токен при кожному non-GET запиті

```
session[:_csrf_token] == params[:authenticity_token]
```

4. Якщо не співпадає → ActionController::InvalidAuthenticityToken

> <b>Strong Parameters (Mass Assignment Protection)</b> - accepts_nested_attributes_for створював «чорний хід» для створення записів у пов'язаних таблицях, про який розробник міг навіть не замислюватися.

```
User.create(params[:user])
```

```
user[roles_attributes][0][admin]=true
```

🛡 Рішення — Strong Parameters

```
def user_params
  params.require(:user).permit(:email, :password)
end
```

> <b>SQL Injection</b> — це тип уразливості вебдодатків, при якому зловмисник впроваджує шкідливий SQL-код у запити до бази даних через вхідні дані (форми, URL).

Пряма інтерполяція параметрів у рядок запиту дозволяє зловмиснику маніпулювати базою даних.

```
User.where("email = '#{params[:email]}'")
```

🛡 Завжди використовуйте плейсхолдери або hash-синтаксис, які автоматично екранують вхідні дані:

```
User.where(email: params[:email])
```

> <b>XSS (Cross-Site Scripting)</b> — це тип вразливості веб-додатків, при якій зловмисник впроваджує шкідливий JavaScript-код на сторінку, яку переглядають інші користувачі. Цей код виконується в браузері жертви, дозволяючи викрадати сесійні файли cookie, логіни, паролі, або перенаправляти користувачів на фішингові сайти.

- html_safe

```
# Контролер: отримуємо ім'я з параметрів URL
@user_name = params[:name] 

# В'юшка: виводимо ім'я, позначаючи його як "безпечне"
<h2>Вітаємо, <%= @user_name.html_safe %></h2>
```

Якщо зловмисник перейде за посиланням ?name=<script>alert('Hacked!')</script>, браузер виконає цей скрипт

- raw

```
# View: raw працює аналогічно html_safe, вимикаючи екранування
<%= raw @user.bio %>
```

Якщо в полі bio збережено шкідливий код, він виконається у кожного, хто відкриє сторінку.

- link_to

```
# Якщо користувач може сам вказати URL свого профілю
<%= link_to "Мій сайт", @user.website_url %>
```

Атакувальник може ввести javascript:alert('XSS'). Коли інший користувач натисне на посилання, виконається JS-код.

🛡 Rails автоматично escape-ить HTML

```
&lt;script&gt;alert('Hacked')&lt;/script&gt;
```

html_safe, raw - вимикає захист

🛡 CSP (Content Security Policy)

Навіть якщо Rails (Ruby on Rails) auto-escape-ить HTML, XSS все одно можливий через:

- html_safe
- сторонні JS бібліотеки
- inline <script>
- CDN, який зламали
- injected third-party script

CSP — це додатковий рівень. Це HTTP header, який говорить браузеру: “Звідки дозволено завантажувати скрипти, стилі, картинки і що можна виконувати.”
