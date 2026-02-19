## 🔹 [Rack](#rack)

## 🔹 [ActiveRecord](#activerecord)

---

# Rack

> <b>Web server</b> — приймає HTTP-запити (наприклад, Nginx).

> <b>Application server</b> — запускає Ruby-процес (наприклад, Puma).

> <b>Rack</b> — стандартний інтерфейс між Ruby-фреймворком і веб-сервером.

> <b>Middleware</b> — обробник, що виконується до/після основного додатку.

1. Браузер надсилає:

```
GET /users/1
```

2. Nginx приймає запит і передає його в Puma.
3. Puma викликає Rack application:

```
call(env)
```

4. env — це Hash з:

```
REQUEST_METHOD
PATH_INFO
HTTP headers
rack.input
```

5. Middleware stack обробляє запит:

```
cookies
session
CSRF
logging
```

6. Router знаходить відповідність:

```
get "/users/:id", to: "users#show"
```

7. Виконується:

```
class UsersController < ApplicationController
  def show
    @user = User.find(params[:id])
    render json: @user
  end
end
```

8. Повертається Rack response:

```
[200, { "Content-Type" => "application/json" }, ["{...}"]]
```

# ActiveRecord

> <b>ORM</b> — мапінг таблиць БД у Ruby-об’єкти.

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
