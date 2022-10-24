# Проектирование облачного файлового хранилища


## Содержание
1. [Тема и целевая аудитория](#part-1)
2. [Расчет нагрузки](#part-2)
3. [Логическая схема](#part-3)
4. [Физическая схема](#part-4)
5. [Технологии](#part-5)
6. [Схема проекта](#part-6)
7. [Список серверов](#part-7)
8. [Список использованных источников](#part-8)


## 1. Тема и целевая аудитория <a name="part-1"></a>
Облачное файловое хранилище позволяет пользователям хранить их файлы удаленно и предоставляет доступ на их просмотр и скачивание. В данной работе в качестве примера будет реализован функцоинал сервиса Dropbox - файлового хостинга компании Dropbox Inc., включающего персональное облачное хранилище, синхронизацию файлов и программу-клиент [1].

MVP сервиса будет включать следующее:
* Загрузка файлов в хранилище
* Скачивание файлов из хранилища
* Просмотр загруженных файлов через браузер
* Предоставление доступа к файлам третьим лицам

На текущий момент в Dropbox зарегистрировано более 700 млн пользователей [2]. При этом [3]:
* В США - 35.40% пользователей - 247.8 млн человек
* В Великобритании - 5.30% пользователей - 37.1 млн человек
* В Японии - 5.22% пользователей - 36.54 млн человек
* В Германии - 3.7% пользователей - 25.9 млн человек
* В Австралии - 3.54% пользователей - 24.99 млн человек
* В остальных странах - 46.84% пользователей - 327.88 млн человек


## 2. Расчет нагрузки <a name="part-2"></a>
### Продуктовые метрики
Dropbox ежемесячно посещает около 153.7 млн пользователей [3]. Это в среднем 5.12 млн человек в день.

Размер хранилища составляет [4]:
* Для бесплатных пользователей - 2 Гб
* Для платных пользователей на самой дорогой подписке - не ограничено

Пусть около 5% посещений происходит с запросом авторизации, тогда ежедневно выполняется 256 тысяч запросов на авторизацию.

Согласно статистике за 2016 год в Dropbox ежедневно загружают порядка 1 200 млн файлов [5]. Т.к. в 2016 году аудитория составляла 433 млн человек, а сейчас составляет более 700 млн человек [2], то возьмём количество загрузок файлов равное 1 940 млн.

На первой странице сервиса отображается список всех файлов. Предположим, что каждый десятый хотя бы раз попадает на эту страницу. Пусть таких запросов 70 млн в день.

Предположим, что пользователи просматривают половину своих файлов в день. Согласно статистике за 2018 год всего было загружено 400 000 млн файлов. Т.к. в 2018 году аудитория составляла 500 млн человек, а сейчас составляет более 700 млн человек [2], то получим 560 000 млн файлов. Это в среднем просмотр 778 млн файлов в день.

Согласно статистике за 2015 год каждый час создается 100 тысяч новых публичных папок и файлов [5]. Т.к. в 2015 году аудитория составляла 400 млн человек, а сейчас составляет более 700 млн человек [2], то возьмем, что каждый час создается 175 тысяч шареных файлов. Тогда в день таких файлов создается порядка 4.2 млн.

Среднее количество действий одного пользователя считается по формуле
`Kд.пол-я = Kд.пол-й / count`,
где:
* Kд.пол-я - среднее количество действий одного пользователя
* Kд.пол-й - среднее количество действий пользователей
* count = 5.12 млн человек - среднее количество посещений в день

Среднее количество действий пользователей представлено в таблице 1, среднее количество действий одного пользователя представлено в таблице 2.

Таблица 1 - Среднее количество действий пользователей
Тип запроса             | Среднее количество (млн/день)
----------------------- | -----------------------------
Авторизация             | 0.256
Загрузка файлов         | 1 940
Просмотр списка файлов  | 70
Просмотр файлов         | 778
Предоставление доступа  | 4.2

Таблица 2 - Среднее количество действий одного пользователя
Тип запроса             | Среднее количество (млн/день)
----------------------- | -----------------------------
Авторизация             | 0.05
Загрузка файлов         | 379
Просмотр списка файлов  | 14
Просмотр файлов         | 152
Предоставление доступа  | 1

### Технические метрики
На текущий момент в Dropbox зарегистрировано более 700 млн пользователей, из которых 15.5 млн - платные пользователи [2].

Размер хранилища составляет [4]:
* Для бесплатных пользователей - 2 Гб
* Для платных пользователей на самой дорогой подписке - не ограничено

Предположим, что:
* Бесплатные пользователи используют 1 Гб
* Платные пользователи используют 256 Гб

В таком случае общий размер хранилища будет равен (684.50 * 10e6) чел * 1 Гб + (15.5 * 10e6) чел * 256 Гб = 684 500 Тб + 3 968 000 Тб = 4 652 500 Тб.

Предположим, что:
* Пользователи хранят в облаке фотографии или текстовые документы
* Средний размер файла - 8 Мб
* В пике средняя нагрузка увеличина в 2 раза

Суточный объем считается по формуле
`Vсут = (K * 10e6 * V1) / 2e20`,
где:
* Vсут - суточный объем
* K - среднее количество действий пользователей/одного пользователя
* V1 - объем 1 запроса

Пиковое потребление считается по формуле:
`Vпик = (2 * K * 10e6 * V1) / (8 * 24 * 3600 * 2e20)`,
где:
* Vпик - пиковое потребление
* K - среднее количество действий пользователей/одного пользователя
* V1 - объем 1 запроса

Входящий трафик представлен в таблице 3, входящий трафик на одного пользователя представлен в таблице 4.

Таблица 3 - Входящий трафик
Тип запроса             | Объём 1 запроса (Кб) | Суточный объём (Гб/сутки) | Пиковое потребление (Гбит/с)
----------------------- | -------------------- | ------------------------- | --------------------------------------------
Авторизация             | 1                    | 0.24                      | 7e-7
Загрузка файлов         | 8192                 | 15 156 250                | 44
Предоставление доступа  | 1                    | 4                         | 1e-5

Таблица 4 - Входящий трафик на одного пользователя
Тип запроса             | Объём 1 запроса (Кб) | Суточный объём (Гб/сутки) | Пиковое потребление (Гбит/с)
----------------------- | -------------------- | ------------------------- | --------------------------------------------
Авторизация             | 1                    | 0.05                      | 1e-7
Загрузка файлов         | 8192                 | 2 960 938                 | 8.57
Предоставление доступа  | 1                    | 0.95                      | 3e-6

Исходящий трафик представлен в таблице 5, исходящий трафик на одного пользователя представлен в таблице 6.

Таблица 5 - Исходящий трафик
Тип запроса             | Объём 1 запроса (Кб) | Суточный объём (Гб/сутки) | Пиковое потребление (Гбит/с)
----------------------- | -------------------- | ------------------------- | --------------------------------------------
Просмотр списка файлов  | 512                  | 34 180                    | 10e-2
Просмотр файлов         | 8192                 | 6 078 125                 | 17.59

Таблица 6 - Исходящий трафик на одного пользователя
Тип запроса             | Объём 1 запроса (Кб) | Суточный объём (Гб/сутки) | Пиковое потребление (Гбит/с)
----------------------- | -------------------- | ------------------------- | --------------------------------------------
Просмотр списка файлов  | 512                  | 6 836                     | 2e-2
Просмотр файлов         | 8192                 | 1 187 500                 | 3.44

Количество запросов в секунду считается по формуле
`RPS = (K * 10e6) / (24 * 3600)`,
где:
* RPS - количество запросов в секунду
* K - среднее количество действий пользователей

RPS по типам запросов для всех пользователей представлено в таблице 7.

Таблица 7 - RPS по типам запросов для всех пользователей
Тип запроса             | RPS
----------------------- | -----
Авторизация             | 3
Просмотр списка файлов  | 810
Просмотр файлов         | 9 005
Загрузка файлов         | 22 454
Предоставление доступа  | 49


## 3. Логическая схема <a name="part-3"></a>
Логическая схема представлена на рисунке 1.

![Логическая схема](https://github.com/kirill555101/highload-storage/blob/main/images/logic-diagram.png)
<p align="center">Рисунок 1 - Логическая схема</p>

Таблица "users" содержит информацию обо всех пользователях и состоит из 5 полей:
* id - первичный ключ
* name - имя пользователя
* email - электронная почта пользователя
* region - регион пользователя

Таблица "accesses" содержит информацию обо всех доступах к файлам и состоит из 5 полей:
* id - первичный ключ
* user_id - внешний ключ, содержащий id пользователя
* file_id - внешний ключ, содержащий id файла
* read_only - флаг доступа только для чтения
* expire_date - дата истечения доступа

Таблица "files" содержит информацию обо всех файлах и состоит из 7 полей:
* id - первичный ключ
* user_id - внешний ключ, содержащий id пользователя
* size - размер файла
* name - имя файла
* path - путь к файлу
* created_at - дата создания файла
* updated_at - дата последнего изменения файла

Таблица "links" содержит информацию обо всех ссылках к файлам и состоит из 5 полей:
* id - первичный ключ
* file_id - внешний ключ, содержащий id файла
* read_only - флаг доступа только для чтения
* uuid - универсально уникальный идентификатор
* expire_date - дата истечения доступа по ссылке

Таблица "subscriptions" содержит информацию обо всех подписках и состоит из 6 полей:
* id - первичный ключ
* type - тип подписки
* name - имя подписки
* space - пространство, доступное в подписке
* price - стоимость подписки
* description - описание подписки

Таблица "user_subscriptions" содержит информацию обо всех подписках пользователя и состоит из 5 полей:
* id - первичный ключ
* user_id - внешний ключ, содержащий id пользователя
* subscription_id - внешний ключ, содержащий id подписки
* active - флаг активности подписки
* auto_renewal - флаг автопродления подписки
* created_at - дата активации подписки
* expire_date - дата истечения подписки

Таблица "payments" содержит информацию обо всех способах оплаты пользователя и состоит из 5 полей:
* id - первичный ключ
* user_id - внешний ключ, содержащий id пользователя
* card_number - номер карты
* expire_date - дата истечения карты
* card_user - фамилия и имяпользователя карты

Таблица "transactions" содержит информацию обо всех оплатах подписок пользователя и состоит из 5 полей:
* id - первичный ключ
* subscription_id - внешний ключ, содержащий id подписки
* payment_id - внешний ключ, содержащий id способа оплаты
* price - стоимость подписки
* created_at - дата оплаты

Таблица "sessions" содержит информацию обо всех сессиях и состоит из 2 полей:
* user_id - внешний ключ, содержащий id пользователя
* hash - хэш сессии


## 4. Физическая схема <a name="part-4"></a>
Для хранения большинства данных будем использовать СУБД PostgreSQL. Рассчитаем, сколько памяти нужно для каждой из таблиц.

Всего зарегистрировано 700 млн пользователей, значит для таблицы users получаем:
* 1 запись = 4 + 50 * 3 + 25 = 179 (байт)
* все записи = 700 * 10e6 * 179 = 117 (Гб)

Для поиска в таблице users создадим индексы по: id, email.

Предположим, что каждый пользователь имеет доступ к 50 документам, значит для таблицы accesses получаем:
* 1 запись = 4 * 3 + 1 + 8 = 21 (байт)
* все записи = 50 * 21 = 1050 (байт)

Для поиска в таблице accesses создадим индексы по: (user_id, file_id).

Согласно пункту 2, всего загружено 560 000 млн файлов, значит для таблицы files получаем:
* 1 запись = 4 * 3 + 50 * 2 + 2 * 8 = 128 (байт)
* все записи = 560 * 10e9 * 128 = 65 (Тб)

Для поиска в таблице files создадим индексы по: id, user_id, name, path.

Предположим, что четверть файлов имеет 1 ссылку, значит для таблицы links получаем:
* 1 запись = 4 * 2 + 1 + 16 + 8 = 33 (байт)
* все записи = 560 * 10e9 * 0.25 * 33 = 4 (Тб)

Для поиска в таблице links создадим индексы по: (file_id, uuid).

В Dropbox присутствует 5 видов подписок [4], значит для таблицы subscriptions получаем:
* 1 запись = 4 * 3 + 5 + 50 + 400 = 467 (байт)
* все записи = 5 * 467 = 2 (Кб)

Для поиска в таблице subscriptions создадим индексы по: id.

Всего платящих 15.5 млн пользователей, предпложим, что у каждого было по 2 подписки, значит для таблицы user_subscriptions получаем:
* 1 запись = 4 * 3 + 1 * 2 + 8 * 2 = 30 (байт)
* все записи = 2 * 15.5 * 10e6 * 30 = 887 (Мб)

Для поиска в таблице user_subscriptions создадим индексы по: id, (user_id, subscription_id).

Предположим, что у каждого платящего пользователя введена одна карта, значит для таблицы payments получаем:
* 1 запись = 4 * 2 + 20 + 8 + 25 = 61 (байт)
* все записи = 15.5 * 10e6 * 61 = 873 (Мб)

Для поиска в таблице payments создадим индексы по: id, user_id.

Предположим, что на одну подписку приходится 2 транзакции, значит для таблицы transactions получаем:
* 1 запись = 4 * 4 + 8 = 24 (байт)
* все записи = 2 * 2 * 15.5 * 10e6 * 24 = 1.4 (Гб)

Для поиска в таблице transactions создадим индексы по: (subscription_id, payment_id).

Для хранения таблицы sessions будем использовать СУБД Redis.

Предположим, что у каждого пользователя есть ровно одна сессия, значит для таблицы sessions получаем:
* 1 запись = 10 + 20 = 30 (байт)
* все записи = 700 * 10e6 * 30 = 20 (Гб)

Для оптимизации работы с БД будем использовать шардирование по регионам. Проверка региона может осуществлятся на основе выставленного языка, указанного региона при регистрации, либо IP-адреса.


## 5. Технологии <a name="part-5"></a>
Выбранные технологии и сферы их применения представлены в таблице 8.

Таблица 8 - Выбранные технологии и сферы их применения
Технология        | Применение
----------------- | --------------------------
React             | Frontend
Nginx             | Web server, load balancer
Golang            | Backend
Redis, PostgreSQL | Database

### Frontend
Для написания клиентской части будет использован фреймворк React, который обладает высокой популярностью и поддержкой.

### Web server, load balancer
Для балансировки на Backend и отдачи статических файлов будет использован веб-сервер Nginx, который активно применяется в этих сферах.

### Backend
Для написания будет использован язык программирования Go, который набирает популярность в качестве языка для написания высоконагруженных веб-приложений.

### Database
В качестве СУБД будут использованы Redis и PostgreSQL, которые обладают хорошими функциональностью и поддержкой.


## 6. Схема проекта <a name="part-6"></a>
Схема проекта представлена на рисунке 2.

![Схема проекта](https://github.com/kirill555101/highload-storage/blob/main/images/project-diagram.png)
<p align="center">Рисунок 2 - Схема проекта</p>

Для балансировки применяется:
* DNS-балансировка 
* L7-балансировка

Backend будет состоять из 3-м микросервисов:
* Auth microservice - микросвервис для работы с автоирзацией, который будет подключен к Redis для реализации сессий
* Payment microservice - микросервис для работы с оплатой
* Access microservice - микросервис для работы с доступом, который будет находиться на сервере с файлами и установленным Nginx для их передачи


## 7. Список серверов <a name="part-7"></a>
Чтобы покрыть основные регионы проживания потенциальных клиентов, предлагается создать 3 датацентра:
* В Северной Америке (Нью-Йорк)
* В Европе (Франкфурт)
* В Юго-Восточной Азии (Токио)

Для хранения данных предлагается использовать RAID 10. Напомним, что RAID 10 - это комбинация RAID 1 + RAID 0. Для RAID 1 предлагатеся использовать 5 дисков, для RAID 0 - 6 дисков. Это повысит надёжность благодаря полному дубированию данных, а также скорость обработки из-за распараллеливания операций по дискам.

Количество серверов в датацентре представлено в таблице 9.

Таблица 9 - Количество серверов в датацентре
Тип                  | Количество основных | Количество резервных
-------------------- | ------------------- | --------------------
Nginx                |                     | 
Auth microservice    |                     | 
Payment microservice |                     | 
Access microservice  |                     | 
Redis                |                     | 
PostgreSQL           |                     | 

Конфигурация каждого сервера представлена в таблицах 10-14.

Таблица 10 - Конфигурация сервера для Nginx
CPU | RAM (Гб) | Тип диска | Объем диска (Гб) | Пропускная способность (Гб/с)
--- | -------- | --------- | ---------------- | -----------------------------
32  | 256      | SSD       | 256              | 20

Таблица 11 - Конфигурация сервера для Auth microservice и Payment microservice
CPU | RAM (Гб) | Тип диска | Объем диска (Гб) | Пропускная способность (Гб/с)
--- | -------- | --------- | ---------------- | -----------------------------
32  | 256      | SSD       | 256              | 20

Таблица 12 - Конфигурация сервера для Access microservice
CPU | RAM (Гб) | Тип диска | Объем диска (Тб) | Пропускная способность (Гб/с)
--- | -------- | --------- | ---------------- | -----------------------------
32  | 256      | HDD       |                  | 20

Таблица 13 - Конфигурация сервера для Redis
CPU | RAM (Гб) | Тип диска | Объем диска (Гб) | Пропускная способность (Гб/с)
--- | -------- | --------- | ---------------- | -----------------------------
16  | 128      | SSD       | 128              | 20

Таблица 14 - Конфигурация сервера для PostgreSQL
CPU | RAM (Гб) | Тип диска | Объем диска (Тб) | Пропускная способность (Гб/с)
--- | -------- | --------- | ---------------- | -----------------------------
32  | 512      | HDD       |                  | 20


## 8. Список использованных источников <a name="part-8"></a>
1. https://ru.wikipedia.org/wiki/Dropbox
2. https://earthweb.com/dropbox-statistics/
3. https://www.similarweb.com/ru/website/dropbox.com
4. https://www.dropbox.com/plans
5. https://expandedramblings.com/index.php/dropbox-statistics/
