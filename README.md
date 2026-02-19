## 🔹 [Rack](#rack)

## 🔹 [ActiveRecord](#activerecord)

## 🔹 [Transactions](#transactions)

## 🔹 [eager loading](#eager-loading)

## 🔹 [Autoloading](#autoloading)

## 🔹 [Sidekiq](#sidekiq)

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
