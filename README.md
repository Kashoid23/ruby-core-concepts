# [Rack](#rack)

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
