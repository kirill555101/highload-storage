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

Количество RPS считается по формуле
`V = (K * 10e6) / (24 * 3600)`, 
где:
* V - RPS
* K - среднее количество действий пользователей

Количество RPS по типам запросов для всех пользователей представлено в таблице 7.

Таблица 7 - Количество RPS по типам запросов для всех пользователей
Тип запроса             | RPS
----------------------- | -----
Авторизация             | 3
Просмотр списка файлов  | 810
Просмотр файлов         | 9 005
Загрузка файлов         | 22 454
Предоставление доступа  | 49


## 3. Логическая схема <a name="part-3"></a>
Логическая схема представлена на рисунке 1.

![Логическая схема](https://github.com/kirill555101/highload-storage/blob/main/logic-diagram.png)

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
* role - роль пользователя
* expire_date - дата истечения доступа

Таблица "files" содержит информацию обо всех файлах и состоит из 7 полей:
* id - первичный ключ
* user_id - внешний ключ, содержащий id пользователя
* size - размер файла
* name - имя файла
* path - путь к файлу
* created_at - дата создания файла
* updated_at - дата последнего изменения файла

Таблица "subscriptions" содержит информацию обо всех подписках пользователя и состоит из 5 полей:
* id - первичный ключ
* user_id - внешний ключ, содержащий id пользователя
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


## 4. Физическая схема <a name="part-4"></a>


## 5. Технологии <a name="part-5"></a>


## 6. Схема проекта <a name="part-6"></a>


## 7. Список серверов <a name="part-7"></a>


## 8. Список использованных источников <a name="part-8"></a>
1. https://ru.wikipedia.org/wiki/Dropbox
2. https://earthweb.com/dropbox-statistics/
3. https://www.similarweb.com/ru/website/dropbox.com
4. https://www.dropbox.com/plans
5. https://expandedramblings.com/index.php/dropbox-statistics/
