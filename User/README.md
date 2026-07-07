# User Service

## Описание проекта

**User Service** — микросервис, реализующий управление пользователями.

Сервис предоставляет REST API для:

- регистрации пользователя;
- авторизации пользователя;
- получения профиля;
- изменения данных пользователя;
- удаления пользователя.

При разработке используются:

- Go 1.26
- Gin
- GORM
- PostgreSQL
- JWT
- bcrypt
- Docker

---

# Архитектура проекта

Проект построен по принципу Layered Architecture (слоистая архитектура).

Каждый слой отвечает только за свою задачу.

```
HTTP Request
      │
      ▼
Transport (Handler)
      │
      ▼
Service
      │
      ▼
Repository
      │
      ▼
PostgreSQL
```

Каждый следующий слой ничего не знает о предыдущем.

Например:

Handler ничего не знает о SQL.

Repository ничего не знает о HTTP.

Service ничего не знает о Gin.

Благодаря этому приложение легко поддерживать и тестировать.

---

# Структура проекта

```
cmd/
    main.go

internal/

    auth/
        jwt.go
        password.go

    config/
        database.go

    dto/
        user.go

    models/
        user.go

    repository/
        repository.go
        errors.go

    services/
        service.go
        errors.go

    transport/
        handler.go
```

---

# Назначение каждого пакета

## cmd

Содержит точку входа приложения.

Здесь создаются все зависимости:

```
Repository

↓

Service

↓

Handler

↓

Router

↓

HTTP Server
```

---

## auth

Пакет содержит всю работу с безопасностью.

В него входят:

- генерация JWT;
- проверка пароля;
- хеширование пароля;
- валидация пароля.

---

## config

Отвечает за конфигурацию приложения.

Сейчас здесь находится:

- подключение PostgreSQL;
- загрузка .env;
- миграции.

---

## dto

DTO (Data Transfer Object).

Используются только для передачи данных.

DTO никогда не работают с базой данных.

DTO бывают двух типов.

Request DTO

```
Клиент

↓

JSON

↓

DTO
```

Response DTO

```
Model

↓

DTO

↓

JSON
```

---

## models

Model описывает структуру таблицы PostgreSQL.

Каждая модель соответствует одной таблице.

Например

```
type User struct
```

↓

```
users
```

---

## repository

Repository отвечает исключительно за работу с PostgreSQL.

Он умеет:

- INSERT
- SELECT
- UPDATE
- DELETE

Repository ничего не знает про HTTP.

Repository ничего не знает про JWT.

Repository ничего не знает про bcrypt.

---

## services

Service — сердце приложения.

Именно здесь находится бизнес-логика.

Service:

- проверяет данные;
- вызывает Repository;
- вызывает bcrypt;
- вызывает JWT;
- принимает решения.

Именно Service определяет правила работы приложения.

---

## transport

Transport (Handler) отвечает только за HTTP.

Он:

- получает запрос;
- читает JSON;
- вызывает Service;
- возвращает JSON.

Больше он ничего делать не должен.

---

# Поток выполнения запроса

Любой запрос проходит одинаковый путь.

```
HTTP Request

↓

Gin Router

↓

Handler

↓

Service

↓

Repository

↓

PostgreSQL

↓

Repository

↓

Service

↓

Handler

↓

HTTP Response
```

Например регистрация пользователя.

```
POST /auth/register

↓

Register()

↓

UserRegister()

↓

Create()

↓

INSERT INTO users

↓

201 Created
```

---

# main.go

Именно отсюда начинается работа программы.

```
func main()
```

Когда приложение запускается, происходит несколько последовательных действий.

---

## 1. Подключение PostgreSQL

```
db := config.SetUpDatabaseConnection()
```

Функция:

- загружает .env;
- получает параметры подключения;
- подключается к PostgreSQL;
- проверяет соединение;
- выполняет миграции.

После этого получается объект

```
*gorm.DB
```

который используется всеми Repository.

---

## 2. Создание Repository

```
userRepo := repository.NewUserRepository(db)
```

Создается Repository.

Он знает как выполнять SQL-запросы.

---

## 3. Создание Service

```
userService := services.NewUserService(userRepo)
```

Создается бизнес-логика.

Service получает Repository.

---

## 4. Создание Handler

```
userHandler := transport.NewUserHandler(userService)
```

Handler получает Service.

---

## 5. Создание Gin

```
r := gin.Default()
```

Создается HTTP сервер.

Автоматически подключаются middleware.

- Logger
- Recovery

---

## Logger

Выводит информацию обо всех запросах.

Например

```
GET /users/1
```

После выполнения увидим

```
200 OK
4 ms
```

---

## Recovery

Если внутри Handler возникнет panic,

Recovery:

- перехватит ошибку;
- не даст приложению завершиться;
- вернет HTTP 500.

---

# Регистрация маршрутов

Создаются две группы.

```
/auth
```

и

```
/users
```

Маршруты регистрации и авторизации

```
POST /auth/register

POST /auth/login
```

Маршруты пользователя

```
GET /users/:id

PUT /users/:id

DELETE /users/:id
```

После регистрации маршрутов вызывается

```
r.Run(":8000")
```

После этого сервер начинает принимать HTTP-запросы.

---

# Пользовательский флоу №1

# Регистрация пользователя

Endpoint

```
POST /auth/register
```

---

## Запрос клиента

Клиент отправляет

```json
{
    "name":"Alex",
    "email":"alex@gmail.com",
    "phone":"+79991234567",
    "password":"password123",
    "favorite_sport":"football"
}
```

---

## Шаг 1

Запрос попадает в Router.

```
POST /auth/register
```

↓

```
userHandler.Register()
```

---

## Шаг 2

Handler получает JSON.

```
ctx.ShouldBindJSON(&req)
```

Gin автоматически преобразует JSON

↓

```
dto.UserRegister
```

Также автоматически выполняется проверка полей по тегам `binding`.

Например:

- `required`
- `email`
- `min`
- `max`
- `e164`

Если хотя бы одна проверка не проходит, Handler сразу возвращает:

```
400 Bad Request
```

До Service выполнение даже не дойдет.

---

## Шаг 3

После успешной валидации Handler вызывает:

```
userService.UserRegister(req)
```

С этого момента начинается бизнес-логика.

---

## Шаг 4. Нормализация данных

Service очищает входные данные.

Имя:

```
strings.TrimSpace(req.Name)
```

Было:

```
"     Alex     "
```

Стало:

```
Alex
```

Email:

```
strings.ToLower(strings.TrimSpace(req.Email))
```

Было:

```
Alex@Gmail.Com
```

Стало:

```
alex@gmail.com
```

Это позволяет избежать регистрации двух пользователей с одинаковым email, отличающимся только регистром символов.

---

## Шаг 5. Проверка email

Service вызывает:

```
repo.GetByEmail(email)
```

Repository выполняет SQL-запрос:

```sql
SELECT *
FROM users
WHERE email = ?
LIMIT 1;
```

Если пользователь найден,

Service возвращает ошибку:

```
ErrUserAlreadyExist
```

Handler преобразует ее в HTTP-ответ.

```
409 Conflict
```

или другой статус, в зависимости от реализации обработчика ошибок.

---

## Шаг 6. Проверка телефона

После проверки email выполняется проверка телефона.

Service вызывает

```
repo.GetByPhone(phone)
```

Repository выполняет запрос к PostgreSQL.

```sql
SELECT *
FROM users
WHERE phone = ?
LIMIT 1;
```

Если пользователь уже существует,

Service возвращает

```
ErrUserAlreadyExist
```

Регистрация прекращается.

Таким образом в системе невозможно зарегистрировать двух пользователей с одинаковым номером телефона.

---

## Шаг 7. Проверка пароля

После проверки уникальности пользователя начинается проверка пароля.

Вызывается

```
auth.ValidatePassword(req.Password)
```

Функция выполняет две проверки.

### Проверка длины

Используется

```
utf8.RuneCountInString(password)
```

Минимальная длина пароля

```
8 символов
```

Почему используется именно RuneCountInString, а не len()?

Потому что

```
len()
```

считает байты.

Например

```
пароль
```

имеет

```
12 байт
```

но

```
6 символов
```

RuneCountInString считает именно символы.

---

### Проверка символов

Далее выполняется цикл

```
for _, r := range password
```

Каждый символ проверяется.

Разрешены только

- буквы
- цифры

Например

```
Password123
```

валидный пароль.

А

```
Password!!!
```

не пройдет проверку.

---

Если пароль не соответствует требованиям,

возвращается

```
ErrInvalidPassword
```

---

## Шаг 8. Хеширование пароля

После успешной проверки пароль никогда больше не хранится в открытом виде.

Вызывается

```
auth.HashPassword(password)
```

Внутри используется библиотека

```
bcrypt
```

Вызывается функция

```
bcrypt.GenerateFromPassword()
```

Она:

- генерирует случайную соль;
- объединяет пароль и соль;
- вычисляет bcrypt-хеш.

Например

Пароль

```
password123
```

станет

```
$2a$10$4YQ...
```

Именно эта строка будет записана в PostgreSQL.

Настоящий пароль нигде не сохраняется.

---

## Почему хранится именно хеш?

Если злоумышленник получит доступ к базе данных,

он увидит

```
$2a$10$4YQ...
```

а не

```
password123
```

Восстановить пароль из bcrypt-хеша практически невозможно.

---

## Шаг 9. Создание модели

После успешного хеширования создается модель.

```
models.User
```

Заполняются поля

```
Name

Email

Phone

PasswordHash

FavoriteSport

Role
```

Получается полноценная модель базы данных.

DTO больше не используется.

---

## Шаг 10. Сохранение пользователя

Service вызывает

```
repo.Create(user)
```

Repository выполняет

```
db.Create(user)
```

GORM генерирует SQL

```sql
INSERT INTO users
(name,email,phone,password_hash,role,favorite_sport)
VALUES(...);
```

После успешного выполнения PostgreSQL создает запись.

Также автоматически заполняется

```
ID
```

который был сгенерирован базой данных.

Например

До сохранения

```
ID = 0
```

После сохранения

```
ID = 15
```

---

## Шаг 11. Формирование ответа

Service создает

```
dto.UserResponse
```

В него входят

```
Name

Email

FavoriteSport
```

Пароль в ответ никогда не включается.

Даже PasswordHash клиент никогда не увидит.

---

## Шаг 12. Ответ клиенту

Handler получает DTO.

Вызывает

```
ctx.JSON()
```

и возвращает

```
201 Created
```

Ответ

```json
{
    "name":"Alex",
    "email":"alex@gmail.com",
    "favorite_sport":"football"
}
```

На этом регистрация завершена.

---

# Полный поток регистрации

```
POST /auth/register

↓

Gin Router

↓

Register()

↓

ShouldBindJSON()

↓

UserRegister DTO

↓

UserService.Register()

↓

TrimSpace()

↓

ToLower()

↓

Repository.GetByEmail()

↓

Repository.GetByPhone()

↓

ValidatePassword()

↓

HashPassword()

↓

Create User Model

↓

Repository.Create()

↓

INSERT INTO users

↓

PostgreSQL

↓

UserResponse DTO

↓

201 Created
```

---

# Пользовательский флоу №2

# Авторизация пользователя

Endpoint

```
POST /auth/login
```

---

## Запрос клиента

```json
{
    "email":"alex@gmail.com",
    "password":"password123"
}
```

---

## Шаг 1

Router вызывает

```
userHandler.Login()
```

---

## Шаг 2

Handler вызывает

```
ctx.ShouldBindJSON()
```

JSON преобразуется в

```
dto.UserLogin
```

Если данные некорректны,

возвращается

```
400 Bad Request
```

---

## Шаг 3

Handler вызывает

```
userService.LoginUser(req)
```

---

## Шаг 4

Service нормализует email.

```
strings.TrimSpace()
```

```
strings.ToLower()
```

Например

```
Alex@Gmail.Com
```

↓

```
alex@gmail.com
```

---

## Шаг 5

Service вызывает

```
repo.GetByEmail(email)
```

Repository выполняет

```sql
SELECT *
FROM users
WHERE email=?
LIMIT 1;
```

Если пользователь отсутствует,

возвращается

```
ErrInvalidCredentials
```

Обратите внимание.

Service специально НЕ сообщает,

что пользователь отсутствует.

Клиент всегда получает одну ошибку

```
invalid credentials
```

Это защищает систему.

Злоумышленник не сможет узнать,

зарегистрирован ли пользователь.

---

## Шаг 6

Если пользователь найден,

получаем

```
PasswordHash
```

Например

```
$2a$10$...
```

---

## Шаг 7

Service вызывает

```
auth.CheckPassword()
```

Внутри используется

```
bcrypt.CompareHashAndPassword()
```

Алгоритм:

Получить соль из сохраненного bcrypt-хеша

↓

Снова вычислить bcrypt

↓

Сравнить два значения

Если совпали,

пароль правильный.

---

Если пароль неверный,

возвращается

```
ErrInvalidCredentials
```

---

## Шаг 8

После успешной проверки вызывается

```
GenerateToken()
```

Создается JWT.

В него записываются Claims

```
user_id

role

exp
```

Например

```
{
    "user_id":15,
    "role":"fan",
    "exp":1783570000
}
```

После этого JWT подписывается

```
HS256
```

с использованием

```
JWT_SECRET
```

Получается строка

```
eyJhbGciOiJIUzI1NiIsInR5...
```

---

## Шаг 9

Handler возвращает

```
200 OK
```

```json
{
    "token":"eyJhbGc..."
}
```

Клиент сохраняет этот токен.

Во всех следующих запросах он отправляет

```
Authorization

Bearer eyJhbGc...
```

---

# Полный поток авторизации

```
POST /auth/login

↓

Gin Router

↓

Login()

↓

ShouldBindJSON()

↓

UserLogin DTO

↓

UserService.Login()

↓

Repository.GetByEmail()

↓

SELECT users

↓

CheckPassword()

↓

GenerateToken()

↓

JWT

↓

200 OK
```

---

# Пользовательский флоу №3

# Получение профиля пользователя

Endpoint

```
GET /users/:id
```

Пример запроса

```
GET /users/15
```

где

```
15
```

— идентификатор пользователя.

---

## Шаг 1

Запрос попадает в Gin Router.

В `main.go` зарегистрирован маршрут

```go
users.GET("/:id", userHandler.GetProfile)
```

Gin вызывает

```
userHandler.GetProfile()
```

---

## Шаг 2

Handler получает параметр маршрута.

```go
idParam := ctx.Param("id")
```

Метод

```
ctx.Param()
```

всегда возвращает строку.

Например

```
"15"
```

---

## Шаг 3

Преобразование строки в число.

```go
id, err := strconv.Atoi(idParam)
```

Получается

```
15
```

Если пользователь отправит

```
GET /users/abc
```

то

```
strconv.Atoi()
```

вернет ошибку.

Handler сразу отвечает

```
400 Bad Request
```

Ответ

```json
{
    "error":"invalid id"
}
```

До Service выполнение не дойдет.

---

## Шаг 4

Handler вызывает

```go
userService.GetProfile(uint(id))
```

---

## Шаг 5

Service вызывает Repository.

```go
repo.GetByID(id)
```

---

## Шаг 6

Repository выполняет

```go
db.Where("id = ?", id).First(&user)
```

GORM генерирует SQL

```sql
SELECT *
FROM users
WHERE id=?
LIMIT 1;
```

---

## Шаг 7

Возможны два варианта.

### Пользователь найден

Repository возвращает

```
models.User
```

↓

Service

↓

Handler

↓

HTTP 200

---

### Пользователь отсутствует

GORM возвращает

```
gorm.ErrRecordNotFound
```

Repository преобразует ее

в

```
ErrUserNotFound
```

Service передает ошибку выше.

Handler возвращает

```
404 Not Found
```

Ответ

```json
{
    "error":"user not found"
}
```

---

## Шаг 8

При успешном выполнении Handler вызывает

```go
ctx.JSON(http.StatusOK,user)
```

Ответ

```json
{
    "id":15,
    "name":"Alex",
    "email":"alex@gmail.com",
    "phone":"+79991234567",
    "role":"fan",
    "favorite_sport":"football",
    "created_at":"...",
    "updated_at":"..."
}
```

Пароль отсутствует.

Это происходит благодаря тегу

```go
json:"-"
```

у поля

```
PasswordHash
```

---

# Полный поток получения профиля

```
GET /users/15

↓

Router

↓

GetProfile()

↓

ctx.Param()

↓

strconv.Atoi()

↓

Service.GetProfile()

↓

Repository.GetByID()

↓

SELECT users

↓

models.User

↓

HTTP 200
```

---

# Пользовательский флоу №4

# Обновление пользователя

Endpoint

```
PUT /users/:id
```

---

Пример запроса

```json
{
    "name":"Alexander",
    "favorite_sport":"Basketball"
}
```

Все поля необязательные.

Можно отправить только одно поле.

---

## Почему все поля указатели?

DTO

```go
type UserUpdate struct {
    Name *string
}
```

Использует указатели.

Это позволяет понять,

отправил пользователь поле

или

не отправил.

Например

```json
{}
```

↓

```
Name == nil
```

А

```json
{
    "name":"Alex"
}
```

↓

```
Name != nil
```

---

## Шаг 1

Router вызывает

```
UpdateUser()
```

---

## Шаг 2

Handler получает ID.

```
ctx.Param("id")
```

↓

```
strconv.Atoi()
```

---

## Шаг 3

Handler получает JSON.

```
ShouldBindJSON()
```

↓

```
dto.UserUpdate
```

---

## Шаг 4

Handler вызывает

```
Service.UpdateUser()
```

---

## Шаг 5

Service получает пользователя.

```
repo.GetByID(id)
```

SQL

```sql
SELECT *
FROM users
WHERE id=?
LIMIT 1;
```

Если пользователя нет,

возвращается

```
ErrNotFoundUser
```

---

## Шаг 6

Обновление имени.

Если

```
req.Name != nil
```

значит пользователь действительно хочет изменить имя.

Service

```
TrimSpace()

↓

проверяет длину

↓

записывает новое значение
```

---

## Шаг 7

Обновление Email.

Сначала выполняется

```
TrimSpace()

↓

ToLower()
```

После этого Service проверяет

не используется ли новый email другим пользователем.

Вызывается

```
repo.GetByEmail()
```

Если email занят,

возвращается

```
ErrAlreadyExistUserWithThisEmail
```

---

## Шаг 8

Обновление телефона.

Если телефон присутствует,

Service выполняет

```
TrimSpace()
```

↓

проверяет длину

↓

записывает новое значение.

---

## Шаг 9

Обновление FavoriteSport.

Если поле существует,

Service удаляет лишние пробелы

и записывает новое значение.

---

# Смена пароля

Самая сложная часть UpdateUser.

---

Если

```
OldPassword == nil

NewPassword == nil
```

пароль вообще не меняется.

---

Если присутствует только один пароль,

например

```
OldPassword
```

без

```
NewPassword
```

возвращается

```
ErrRequiredNewAndOldPassword
```

---

Если присутствуют оба,

Service выполняет

```
CheckPassword()
```

Проверяется,

правильно ли пользователь ввел текущий пароль.

Если нет,

возвращается

```
ErrOldPasswordIsntCorrect
```

---

После этого новый пароль проходит

```
ValidatePassword()
```

Если пароль не соответствует требованиям,

возвращается

```
ErrNewPasswordIsntCorrect
```

---

После успешной проверки

новый пароль снова проходит

```
HashPassword()
```

Получается новый bcrypt-хеш.

Именно он записывается

в

```
PasswordHash
```

---

## Шаг 10

Repository выполняет

```go
db.Model(&models.User{}).
Where("id = ?",user.ID).
Updates(user)
```

SQL

```sql
UPDATE users
SET ...
WHERE id=?
```

---

## Шаг 11

Handler возвращает

```
200 OK
```

Ответ

```json
{
    "message":"updated"
}
```

---

# Полный поток обновления

```
PUT /users/15

↓

Router

↓

UpdateUser()

↓

ShouldBindJSON()

↓

UpdateUser DTO

↓

Repository.GetByID()

↓

Проверка Email

↓

Проверка Password

↓

HashPassword()

↓

Repository.Update()

↓

UPDATE users

↓

200 OK
```

---

# Пользовательский флоу №5

# Удаление пользователя

Endpoint

```
DELETE /users/:id
```

---

Удаление пользователя защищено JWT.

Перед выполнением Handler получает

данные,

которые Middleware извлек из JWT.

---

JWT содержит

```
user_id

role
```

Например

```
{
    "user_id":15,
    "role":"fan"
}
```

Middleware сохраняет их

в

```
gin.Context
```

---

## Шаг 1

Handler получает

```
user_id
```

```go
ctx.GetUint("user_id")
```

---

## Шаг 2

Получает роль.

```go
ctx.GetString("role")
```

Например

```
admin
```

или

```
fan
```

---

## Шаг 3

Получает id пользователя,

которого нужно удалить.

```
DELETE /users/20
```

↓

```
20
```

---

## Шаг 4

Проверка прав.

```go
if role != models.RoleAdmin &&
uint(id)!=userIDFromToken
```

Это означает

если

пользователь

не администратор

и

пытается удалить чужой аккаунт,

то доступ запрещается.

Возвращается

```
403 Forbidden
```

Ответ

```json
{
    "error":"cannot delete other users"
}
```

---

Администратор

может удалить любого пользователя.

Обычный пользователь

может удалить только самого себя.

---

## Шаг 5

После проверки прав

должен вызываться

```
userService.DeleteUser()
```

который вызывает

```
Repository.Delete()
```

Repository выполняет

```sql
DELETE FROM users
WHERE id=?
```

или,

если используется Soft Delete,

GORM обновляет

```
deleted_at
```

---

## Шаг 6

После успешного удаления

Handler возвращает

```
200 OK
```

или

```
204 No Content
```

в зависимости от выбранной реализации API.

---

# Полный поток удаления

```
DELETE /users/15

↓

JWT Middleware

↓

Извлечь user_id

↓

Извлечь role

↓

DeleteUser()

↓

Проверка прав

↓

Repository.Delete()

↓

DELETE users

↓

200 OK
```

---

# Важно

В текущей версии проекта метод

```
DeleteUser()
```

в Handler **не завершен**.

После проверки прав отсутствует вызов

```
userService.DeleteUser()
```

и не формируется HTTP-ответ.

Для завершения реализации необходимо:

```
↓

userService.DeleteUser()

↓

Repository.Delete()

↓

ctx.JSON(...)
```

Иначе пользователь фактически не будет удален.

# Работа JWT

JWT (JSON Web Token) используется для аутентификации пользователей.

После успешной авторизации пользователь получает токен.

В дальнейшем сервер не хранит информацию о сессии.

Каждый запрос содержит JWT.

---

## Что находится внутри JWT

При успешном входе вызывается

```
auth.GenerateToken(user.ID, user.Role)
```

Создаются Claims

```go
claims := jwt.MapClaims{
    "user_id": userID,
    "role": role,
    "exp": time.Now().Add(24*time.Hour).Unix(),
}
```

В JWT записывается

```
user_id

role

exp
```

Например

```json
{
    "user_id":15,
    "role":"fan",
    "exp":1783570000
}
```

---

## Подписание токена

После создания Claims вызывается

```
jwt.NewWithClaims()
```

После этого выполняется

```
SignedString(secret)
```

где

```
secret
```

берется из

```
JWT_SECRET
```

например

```
JWT_SECRET=my_super_secret_key
```

Получается строка

```
eyJhbGciOiJIUzI1NiIsInR5...
```

Именно она отправляется клиенту.

---

## Использование JWT

После входа клиент хранит токен.

При каждом запросе он отправляет

```
Authorization

Bearer eyJhbGc...
```

Middleware:

- проверяет подпись;
- проверяет срок действия;
- извлекает user_id;
- извлекает role;
- сохраняет данные в gin.Context.

После этого Handler получает

```
ctx.GetUint("user_id")

ctx.GetString("role")
```

и может определить,

кто выполняет запрос.

---

# Работа bcrypt

bcrypt используется для хранения паролей.

Пароль никогда не записывается в базу данных.

---

## Регистрация

Пользователь вводит

```
password123
```

Service вызывает

```
HashPassword()
```

↓

```
bcrypt.GenerateFromPassword()
```

↓

```
$2a$10$...
```

↓

Repository.Create()

↓

PostgreSQL

---

## Авторизация

Пользователь вводит

```
password123
```

Service получает

```
PasswordHash
```

из базы.

Вызывает

```
CheckPassword()
```

↓

```
bcrypt.CompareHashAndPassword()
```

Если вычисленный bcrypt совпадает,

авторизация успешна.

---

# Работа Repository

Repository является единственным слоем,

который работает с PostgreSQL.

Он содержит методы

```
Create()

GetByID()

GetByEmail()

GetByPhone()

Update()

Delete()
```

Service никогда не выполняет SQL.

Handler никогда не выполняет SQL.

---

## Create

Используется

```
db.Create()
```

SQL

```sql
INSERT INTO users(...)
VALUES(...);
```

---

## GetByID

Используется

```
Where()

↓

First()
```

SQL

```sql
SELECT *
FROM users
WHERE id=?
LIMIT 1;
```

---

## GetByEmail

SQL

```sql
SELECT *
FROM users
WHERE email=?
LIMIT 1;
```

---

## GetByPhone

SQL

```sql
SELECT *
FROM users
WHERE phone=?
LIMIT 1;
```

---

## Update

Используется

```
Updates()
```

SQL

```sql
UPDATE users
SET ...
WHERE id=?
```

---

## Delete

Используется

```
Delete()
```

При использовании

```
gorm.Model
```

работает Soft Delete.

Поле

```
deleted_at
```

заполняется автоматически.

---

# Работа GORM

GORM является ORM.

ORM (Object Relational Mapping)

позволяет работать с таблицами

как со структурами Go.

Например

```go
user := models.User{}
```

вместо

```sql
SELECT * FROM users
```

используется

```go
db.First(&user)
```

GORM самостоятельно генерирует SQL.

---

# Работа моделей

Модель

```
models.User
```

соответствует таблице

```
users
```

Поля модели

```
ID

Name

Email

Phone

Role

PasswordHash

FavoriteSport

CreatedAt

UpdatedAt

DeletedAt
```

становятся столбцами таблицы PostgreSQL.

---

# Работа DTO

DTO используются только

для обмена данными.

Они никогда не используются

для хранения данных в PostgreSQL.

Используются четыре DTO.

```
UserRegister

UserLogin

UserUpdate

UserResponse
```

---

## UserRegister

Используется

```
POST /auth/register
```

---

## UserLogin

Используется

```
POST /auth/login
```

---

## UserUpdate

Используется

```
PUT /users/:id
```

Почти все поля являются указателями.

Это позволяет понять,

какие поля пользователь действительно изменяет.

---

## UserResponse

Используется

для формирования ответа клиенту.

PasswordHash отсутствует.

---

# Работа Handler

Handler отвечает только за HTTP.

Он умеет

- читать JSON;
- получать параметры URL;
- вызывать Service;
- возвращать JSON.

Handler никогда

не выполняет SQL.

Handler никогда

не генерирует JWT.

Handler никогда

не хеширует пароль.

---

# Работа Service

Service содержит

всю бизнес-логику.

Именно Service

- проверяет email;
- проверяет телефон;
- проверяет пароль;
- вызывает bcrypt;
- вызывает JWT;
- принимает решения;
- вызывает Repository.

Service ничего не знает

про HTTP.

---

# Работа Config

Config отвечает

за подключение PostgreSQL.

Последовательность

```
Load .env

↓

Получить DB_HOST

↓

Получить DB_USER

↓

Получить DB_PASS

↓

Получить DB_NAME

↓

Сформировать DSN

↓

gorm.Open()

↓

Ping()

↓

AutoMigrate()

↓

Вернуть *gorm.DB
```

---

# Работа Docker

Используется Multi Stage Build.

Первый этап

```
golang:1.26-alpine
```

Используется

для сборки приложения.

Второй этап

```
alpine
```

используется

только для запуска бинарного файла.

В итоговый образ

не попадают

- исходный код;
- компилятор Go;
- зависимости.

Поэтому итоговый образ получается значительно меньше.

---

# Полный жизненный цикл HTTP-запроса

Например

```
POST /auth/login
```

```
Клиент

↓

HTTP

↓

Gin Router

↓

Handler

↓

ShouldBindJSON()

↓

DTO

↓

Service

↓

Repository

↓

PostgreSQL

↓

Repository

↓

Service

↓

JWT

↓

Handler

↓

JSON

↓

HTTP Response
```

Абсолютно любой запрос

проходит одинаковый путь.

Меняется только бизнес-логика Service.

---

# Принципы проекта

При разработке используются следующие принципы.

## Single Responsibility Principle

Каждый слой отвечает только за одну задачу.

Handler

↓

HTTP

Service

↓

Бизнес-логика

Repository

↓

PostgreSQL

---

## Dependency Injection

Все зависимости создаются

в

```
main.go
```

Repository

↓

Service

↓

Handler

Благодаря этому

код легко тестировать.

---

## Разделение ответственности

Каждый слой

ничего не знает

о деталях работы остальных.

Например

Repository не знает

что приложение использует Gin.

Handler не знает

как устроена PostgreSQL.

Service не знает

какой HTTP Framework используется.

---

# Итоговая архитектура

```
                           HTTP Request
                                │
                                ▼
                        Gin Router
                                │
                                ▼
                     UserHandler (Transport)
                                │
                                ▼
                    UserService (Business Logic)
                                │
            ┌───────────────────┼───────────────────┐
            ▼                   ▼                   ▼
     Password (bcrypt)      JWT Service      UserRepository
                                                    │
                                                    ▼
                                                  GORM
                                                    │
                                                    ▼
                                               PostgreSQL
```

---

# Заключение

Данный сервис реализует полноценный REST API управления пользователями.

В проекте применяются распространенные практики разработки Go-приложений:

- Layered Architecture;
- Dependency Injection;
- Repository Pattern;
- DTO;
- GORM;
- PostgreSQL;
- JWT-аутентификация;
- bcrypt для безопасного хранения паролей;
- Docker Multi-stage Build.

Основная бизнес-логика сосредоточена в слое Service, доступ к данным инкапсулирован в Repository, а слой Transport отвечает исключительно за обработку HTTP-запросов и формирование ответов. Такое разделение упрощает сопровождение проекта, делает код более тестируемым и позволяет независимо изменять отдельные части системы без влияния на остальные слои.