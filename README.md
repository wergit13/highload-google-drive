# Проектирование высоконагружнных приложений

Курсовая работа в рамках 3-го семестра программы по Веб-разработке ОЦ VK x МГТУ им. Н.Э. Баумана (ex. "Технопарк") по
дисциплине "Проектирование высоконагруженных сервисов"

#### Содержание:

1. [Тема и целевая аудитория](#1)
2. [Расчёт нагрузки](#2)
3. [Глобальная балансировка нагрузки](#3)
4. [Локальная балансировка нагрузки](#4)
5. [Логическая схема базы данных](#5)
6. [Физическая схема базы данных](#6)
7. [Алгоритмы](#7)
8. [Технологии](#8)
9. [Обеспечение надежности](#9)
10. [Схема проекта](#10)
11. [Список серверов](#11)

## Тема и целевая аудитория <a name="1"></a>

Тема - "Проектирование сервиса по типу Google Drive"

Функционал (MVP) :
- загрузка файла
- удаление файла
- скачивание файла
- поиск по диску
- создание общей папки, файла
- история изменения папок, файлов

Целевая аудитория:
- 2.6B Посещений в месяц по всему миру
- 85.5M Посещений в день
- Среднее время визита 6 min
- Среднее количество страниц за визит 5
- Bounce Rate(Cредний процент посетителей, которые просматривают только одну страницу) - 29%
- Основные пользователи офисные сотрудники и студенты, возраст 20-40 лет
- Распредение по странам
  | Страна         | Процент от всех юзеров   |
  | -------------- | ------------------------ |
  | США            | ~25 %   |
  | Бразилия       | ~6 %    |
  | Индия          | ~4.5 %  |
  | Индонезия      | ~3 %    |
  | Мексика        | ~3 %    |
  | Япония         | ~3%     |
Источники: [hypestat](https://hypestat.com/info/drive.google.com), [simularweb](https://www.similarweb.com/website/drive.google.com/#demographics)

## Расчет нагрузки <a name="2"></a>

### Объем хранилища и типы файлов
В стандартном тарифном плане объем предоставлемого хранилища 15 гб для компаний и платных пользоватей 2тб. Платных пользоватей у Google drive 8mln. Вообщем пользоватей примерно 2.6bln. На основании собственного опыта и опроса знакомых диск в среднем заполнеy на 2/3. Примерно по пропорции можно закладывать 18GB в среднем на пользователя.

**Общий размер всего хранилища**: ```18GB * 2.6bln ~ 41_600PiB.```

По опросам пользователей, 80% используют диск только для видео и фото. Для простоты будем счиать что остальные файлы это документы. Поскольку основной алгоритм использования это загрузка на диск фото и видео с телефона для долгострочного хранения. Соотношение фото к видео у меня и знакомых на телефоне примерное 8:1, будем брать такую статистику в расчетах.

  | Тип файла         |Процент от всех файлов в штуках    |
  | -------------- | ------------ |
  | Фото           | 71 %        |
  | Видео          | 9 %         |
  | Документы      | 20 %       |


Оценим средний размер PNG картинки ```( 1920px * 1080 px * 4b + 33b (header) ) * 0.66 (среднее сжатие DEFLATE) ~= 5.12 MiB```. Также есть информация что средний размер фотографии на iPhone в 2023 году был 2-8mb, что примерно бьется с расчетами.
Продолжая ориентироватся что большая часть /фото видео снимается на iPhone возьмем средний размер 1080p/30fps видео снятого на последний - это ```60MiB/min```, средняя продолжительность видео на самом попярном видеохостинге ```11.7min```. Исходя из этого средний обьем видео:```60 * 11.7 = 700MiB```
Размер документа docx/pdf   ```10 kB + 3kB/page```, будем брать среднем 30KiB.
Итого:

  | Тип файла      |Средний размер  |
  | -------------- | -------------- |
  | Фото           | 5.12MiB        |
  | Видео          | 700MiB         |
  | Документы      | 30KiB          |



### Среднее количество действий пользователя день:

Приведем статистику которая есть от похожего сервиса - [Dropbox](https://marketsplash.com/dropbox-statistics/#link4).
 - 1.2bln загружаемых файлом в день при DAU 4mln, в пересчете 300 файлов в день на пользователя
 - 100k создаваемых ссылок на папки, файлы при DAU 4mln в пересчете 0.6 ссылок в день на пользователя
   

Просмотр файла также будем считать за скачивание, так как пользователь пользователь просматривая картинку/видео/документ его скачивает.
Пусть около 5% посещений происходит с запросом авторизации. Среднее количество посещаемых страниц равно 5, а поскольку на диске только папки являются страницами будем брать это как просмотр содержимого папки. Будем считать что просматривают пользователи в 2 раза больше чем загружают, к-во удалений тоже возьмем сравнимое с загрузкой.


| Действие    | Среднее количество дейстивий в день на пользователя |
| --------    | ------- |
| Аутентификация    | 0.05 |
| Аутентификация по куке | 1 |
| Загрузка на диск | 300 |
|Просмотр/скачивание с диска| 600|
| Удаление | 100 |
|Создание ссылки| 0.6 |
| Просмотр директории | 5 |
| Поиск по диску | 5 |


### RPS
DAU = 85.5 M
RPS = количесво действий на пользователя * DAU / 86 400

Файлы будут загрузаться чанками по 10 MiB с клиента, чанкование возможно в основном только для видео в нашем случае это ```700/10 = 70``` запросов на загрузку видео. Количество загрузок видео ```300*9% =27```.

```RPS = DAU * (0.05 + 300*91% + 27 * 70 + 600 + 100 + 0.6 + 5 + 5) / 86400 = 2873.65 * DAU /86400 ~ 2_850_000```

Общее
- **RPS** ~ 2_850_000.
- **Пиковый RPS (x1.5)** ~ 4_250_000.

**RPS по типам запросов**
| Запрос    | RPS | Пиковый RPS |
|---|---|--|
| Аутентификация  | 50 | 75 |
| Аутентификация по куке | 1_000| 1_500|
| Загрузка на диск | 300_000 |450_000 |
| Скачивание с диска | 600_000 |900_000|
| Создание сслыки/изменение доступа |600| 900|
| Просмотр директории | 5_000 |7_500|
| Поиск по диску | 5_000 |7_500|

### Расчет сетевого траффика
DAU = 85.5 M

Скачивание будет производится с s3 так что посчитаем его отдельно. Все запросы не связанные с файлами будем считать размером в 5KiB

**Общий трафик**
```Дневной трафик = DAU * sum(count*size) * 8```

```DAU * ((0.05+600+100+0.6 + 5 + 5) * 5KiB + 300*20%*30KiB + 27*700MiB + 300*71%*5.12MiB) * 8 = DAU * (3.47MiB + 1.76MiB + 18900MiB + 1090.56MiB) * 8 ~ 12_738 Pbit/day```

``` Трафик с секунду = 12_738 Pbit/day / 86400 = 151 Tbit/sec```


**Скачивание**

``` Скачивание в день = DAU * 600 * (0.2 * 30KiB + 0.09 * 700MiB + 0.71 * 5.12MiB) * 8 = 25_491 Pbit/day ```

``` Скачивание в секунду =  25_491 Pbit/day / 86400 = 302Tbit/sec```

|Запрос| Трафик/s |
|--|--|
| Аутентификация  | 1 Мbit|
| Аутентификация по куке | 41 Mbit |
| Загрузка на диск | 155 Gbit |
| Скачивание с диска | 310 Gbit |
| Создание сслыки/изменение доступа |25 Mbit|
| Просмотр директории |  400 Mbit|
| Поиск по диску | 200 Mbit |

### Технические метрики:
| Метрика    | Значение |
| -------- | ------- |
| Общий размер хранилища  | 41_600PiB |
| Траффик в день | 38_230 Pbit/day |
| Траффик в секунду | 453 Tbit/s |
| Пиковый трафик с секунду|680 Tbit/s|
| RPS | 2_850_000 |
| Пиковый RPS | 4_250_000 |

## Глобальная балансировка <a name="3"></a>

### Выбор расположения датацентров

Большая часть аудитории находится в США (28 %), другие же основные места это северная часть Южной Америки(Мексика, Бразилия) и Азия(Индия, Япония), остальной трафик равномерно приходит с Европы. 

Карта расположения ЦОД'ов Google
<img width="895" alt="image" src="https://github.com/wergit13/highload-google-drive/assets/102697969/ea7f896a-d22f-44d5-b8c7-c331e8445093">

Карта плотности населения
<img width="895" alt="image" src="https://github.com/wergit13/highload-google-drive/assets/102697969/949f0d94-6f22-4d2d-9341-8b58b2868e64">

Предлагаемое расположение ЦОД'ов для диска
<img width="895" alt="image" src="https://github.com/wergit13/highload-google-drive/assets/102697969/c07ff675-a601-4453-a5f3-af62e2941db9">

По США установим ЦОД'ы в Портланде, Техасе, Нью-Йорке и Калилифорнии как в наиболее населенных местах, в Техасе чтоб еще доставать до Мексики, а с Портланда до Канады. Еще двумя в Бельгии и в Финляндии покроем западную и северную Европы. ЦОД в Японии покроет саму Японию и восточную Россию. Сингапур же отвечает за Индию, юг Азии и Океанию. И еще один в Чили покрывает аудиторию Южной Америки.

Стоит отметить что хоть и для файлового хранилища неприятны длинные пути данных, но задержка не является критическим фактором для системы. Например ping от довольно удаленных важных городов НьюДели и Мельбурна до ближайшего ЦОД'а(Сингапура) будет 80 и 90ms соответсвенно, что является приемлемым значением.

Сайты статистики не предоставляют точных данных по странам однако если считать что насение европы прорционально США пользуются диском исходя из похожих культур получается что на один ЦОД будет приходится от 7 до 18% пользовательской аудитории. Самыми нагруженными ЦОД'адами будут 2 европейских. Тогда в самом нагруженном случае на один ЦОД будет приходится
**Примерная нагрузка на один ЦОД**

| <!-- -->      | <!-- -->        | <!-- -->      |
|:-------------:|:---------------:|:-------------:|
| Траффик в день | 6_900 Pbit/day |
| Траффик в секунду | 81.5 Tbit/s |
| Пиковый трафик с секунду|122 Tbit/s|
| RPS | 513_000 |
| Пиковый RPS | 765_000 |

### Выбор способа глобальная балансировки

Для балансировки клиентов между регионами(Америка, Европа, Азия) будем использовать Geo Based DNS (например Amazon Route 53). 

Для уменьшения задержки в пределах регионом будем использовать BGP Anycast для выбора нужного ЦОД'а. В США будет использоваться один IP для AS, который будет сопостовлять с IP настоящего ЦОД'а.
Например в случае США(самого нагруженного региона) разобьем все линки BGP сети условно на 4 группы, каждая из которых будет соответствовать одному из четырех ЦОД'ов. При загруженности (или при полном отказе одного из ЦОД'ов) мы можем скорректировать BGP веса для снижения нагрузки на ЦОД.

## Локальная балансировка <a name="4"></a>

### Балансировка

Балансировку будем делать посредством роутинга и L7 балансировщиков.

После приземления клиента в цод маршрут выбирается на основании BGP Routing'а поскольку у диска можеть быть большой входящий трафик стоит использовать сетевое оборудование. Будем использовать хэширование по IP, чтобы пакеты с одного роутера пользователя попадали на один и тот же балансировщик. 

В качестве L7 балансировщика будем использовать nginx, он же будет и отвечать за SSL терминацию.

### Отказоустойчивать

1. Роутеры должны поддержитьвать VRRP чтоб в случае отказа заменять друг друга
2. Использование keepalived для проверки доступности нод бекэендов и балансировщиков. Он будет сообщать Nginx'у о падении/подъеме новых бекэндов, а также резервировать сами балансировщики через CARP 

Все балансеры ставим попарно в 2 раза больше чем нужно для пикового трафика.

## Логическая схемы данных <a name="5"></a>
### Общая логическая схема

Схема без учета избавления от join и разбиенения на базы
<img width="1394" alt="image" src="https://github.com/wergit13/highload-google-drive/assets/102697969/e801977f-1d8a-4bf9-acd1-ae429592c0c0">

**share_id** - id файла или папки для доступа по ссылке, если файлом поделились, размер взят такой же как у GoogleDrive в сслылках.

**pemission** - тип доступа чтение/запись к файлу/папке

### Таблица пользователей

  * uid - id пользователя ( 16 B )
  * username - имя пользователя ( 60 B )
  * email - email пользователя ( 60 B )
  * password_hash - хеш пароля ( 32 B )
  * created_at - дата создания аккаунта ( 4 B )
  * diskroot - корневая папка диска ( 16 B)

```188 B * 2.6 bil  = 525.2 GiB ```
  
### Таблица Сессий

  * cookie - кука с которой пользователй залогинен ( 32 B )
  * expiers - дата истечения действия куки ( 4 B )
  * user_uid - пользователь который зашел с такой кукой ( 16 B )

Допустим пользователь в среднем залогинен с двух устройств

``` 52 B * 2 * 2.6 bil =  96 GiB ```

### Таблица папок

  * uid ( 16 B )
  * owner_uid  - владелец файла ( 16 B )
  * parent - родительская папка ( 16 B) 
  * name - имя папки ( 60 B )
  * created - дата создания ( 4 B )
  * share_id? ( 50 B )

Пусть у пользователя в среднем 60 папок

``` 112 B * 60 * 2.6 bil = 16 TiB ```

### Таблица файлов

  * uid ( 16 B )
  * owner_uid  - владелец файла ( 16 B )
  * parent - родительская папка ( 16 B) 
  * name - имя файла ( 60 B )
  * last_changed - дата последнего изменения ( 4 B )
  * last_version - id последней версии ( 16 B )
  * created - дата создания ( 4 B )
  * s3_url - s3 ссылка на последнюю версию ( 256 B )
  * redactor? - пользователь который послений изменил ( 16 B ) 
  * share_id? ( 50 B )

``` 446 B * 3200 ( среднее кол-во файлов у пользователя ) * 2.6 bil = 3.2 PiB ```

### Таблица версий файлов

Версий больше одной бывает только у документов которых мы считаем около 10%, пусть у документа 3 версии в среднем  

  * uid ( 16 B )
  * file_uid - файл к которому относится версия  ( 16 B )
  * user_uid - пользователь что загрузил версию ( 16 B )
  * s3_url - ссылка на s3 ( 256 B )
  * uploaded - дата изменения ( 4 B )
  * hash - хеш файла ( 32 B )

``` 328 B * 3200 * 1.2 * 2.6 bil  = 2.8 PiB ```

### Таблица прав доступа

  * share_id ( 50 B )
  * user_uid -  владелец файла ( 16 B )
  * file_uid - файл или папка к которому дается доступ ( 16 B )
  * Array(<user_uid, permisson>) - список пользователей имеющих права  ( (16 B + 1 B) * elem )

Есть следующая статистика которую можно использовать для оценки количества ссылок
 * 3200 файлов на человека
 * 0.6 создаваемых ссылок в день на человек
 * 300 создаваемых файлов в день на человека

По пропорции получаем 6.4 ссылки на человека, будем считать что пользователь делится с двумя людьми в среднем, так что нам надо хранить в списке 2 пользователя

``` (50 B + 16 B * 2 + (16 B + 1 B) * 2) * 6.4 * 2.6 bil = 1.75 TiB ```

## Физическая схема данных <a name="6"></a>
### PostgreSql

В postgreSQL будем хранить талблицы files  и folders

Индексы для таблиц Folders и Files:
- id - Hash unique index - для доступа как по ключу
- parent - B-tree index для поиска всех потомков

### Clickhouse

Ипользовать clickhouse будем для хранения версий файлов, которые всегда пишутся, но редко читаются

Индекс для таблицы Versions:
- id - Hash unique index
- file - B-tree index для поиска версий файла

### Tarantool

Для таблиц users, acl и sessions будем использовать Tarantool с фреймворком Cartridge

Для users ключем будет uuid

Для sessions ключем будет кука

Для acl ключем будет являтся ссылка доступа, значением будет список пар (пользователь, права) или просто общий тип доступа.

Также через Tarantool будет работать сервис координатор который будет хранить пары user:shard и определять на каком шарде postgres'а хранятся данные диска пользователя.

### S3

Сами файлы будем хранить в S3 хранилище. S3 бакеты разделяем по uuid пользователя.

### Общая схема

<img width="899" alt="image" src="https://github.com/wergit13/highload-google-drive/assets/102697969/231e2d11-89ee-499c-a736-b592dc92bf92">

Для отказоустойчивости есть запасные реплики на чтение, а также запасной мастер на случай падения основного. 

С запасных реплик, поскольку они не под нагрузкой можно снимать дампы.

## Расчет количества шардов и нагрузки на базы данных

### FileDB

**Количесво шардов**

В базе данных FileDB мы храним таблицы files и folders.

Всего файлов ``` 3200 * 2.6bln = 8320bln ``` записей. Папок ``` 60 * 2.6bln = 156bln ```

Будем расчитывать количесво записей на шард так чтобы индексы влезли в RAM.

Создадим таблицу из пары значений uuid, заполним случайными значениями, построим b-tree и hash индексы и посмотрим размер

![image](https://github.com/wergit13/highload-google-drive/assets/102697969/b6df65df-874c-4c6c-88db-c4cae04299e8)

Итого:
  * Unique hash index ``` 0.515 MiB / 10k ```
  * B-tree index ``` 0.256 MiB / 10k ```

На 10k записей в таблицу files или folders надо 0.771 MiB

Индексы по файлам и папкам весят одинакого так что можем их сложить, на один датацентр приходится 18% нагрузки.

Т.е. один датацентр должен хранить ``` 1275 bln ``` записей файлов/папок

Тогда размер индексов будет ``` 1275 bln / 10k * 0.771 MiB = 93.7 TiB ```

Будем считать что один сервер базы данных имеет 2 Tib RAM, для запаса будем держать только 85% заполнеными 

Тогда на ЦОД нужно  ``` 93.7 / (2 * 0.85) = 55 ``` шардов

На одноном шарде будет хранится ``` 2 TiB * 0.85 / 0.771 Mib/10k = 23 bln ``` записей

Которые будут весить(посчитаем по файлам) ``` 23 bln * 446 B = 9.4 TiB ```

**Количество реплик**

Нагрузка на один шард (будем считать как ```нагрузка * 18% / 55``` )
Будем считать что в одной папке 50 файлов

| Запрос   | RPS | ТИП|
|---|---|--|
| Загрузка на диск | 950 |запись|
| Скачивание с диска | 1_950 |чтение|
| Просмотр директории | 20 |чтение|

Нагрузка чтение получилась великовата и при пиковых нагрузках Postgres может не выдержать, будет логичным читать не конкретный файл, а массивами по 50(Столько у Google Drive) читать содержимое папки. Так можно будет сильно снизить RPS. Информация о всех файлах пользователя будет весить ```1200 * 446 B =  0.53 MiB ``` что спокойно сможет обработать фронтенд, например, для некоторых запросов поиска. В rps на скачиваение оставим только чтение от пользователей по ссылке. Количество запросов на чтение будет больше чем количество просморов директории, домножим чтение директории на 3. Тогда получим.

| Запрос   | RPS | ТИП|
|---|---|--|
| Загрузка на диск | 950 |запись|
| Скачивание с диска | 10 | чтение|
| Просмотр директории | 60 |чтение|

Размер текста самого запроса возьмем за 200 B.

Сеть(прием): ```950 * 446 B + (10 + 60 + 950 * 200B) = 5 Mbit```

Сеть(отдача): ```(60 * 50 + 10) * 446 B = 11 Mbit ```

Теперь нагрузка на один шард не такая большая. Для отказоустойчивости каждому шарду будем хранить одну master реплику и 2 slave которые будут держать нагрузку на чтение.

Итого размер на один ЦОД на FileDB выходит(учитывая реплики) ``` 2_068 TiB ```


**Tarantool coordinator**
Координатор будет хранить ключ значение пары **user:(shard,root_folder)**, каждая такая запись занимает ``` 16 B + 16 B + 8 B = 40B ``` 

Тогда на одном инстансе надо будет хранить ** 40B * 2.6 bln = 100 GiB **

Нагрузка же на координатор будет только при заходе пользователя на сайт ```~200rps``` что довольно мало для такой базы как tarantool


### UserDB
**Пользователи**

| Характеристика   | Значение |
|---|---|
| Объем | 525 GiB |
| Запросы на чтение| 200 |
| Запросы на запись | 10 |

**Сессии**
| Характеристика   | Значение |
|---|---|
| Объем | 96 GiB |
| Запросы на чтение| 200 |
| Запросы на запись | 10 |

**ACL**

Просмотр по ссылке посчитам как ```% файлов со ссылкой * rps просмотра```.

| Характеристика   | Значение |
|---|---|
| Объем | 1.75 TiB |
| Запросы на чтение| 216 |
| Запросы на запись | 10 |

Тогда размер UserDB  на один ЦОД выходит ```~11.25TiB```

Каждый Tarantool реплицируем аналогично Postgres. Делаем запасной master и набор из 3 slave на чтение.

### Обеспечение надежности

1. В случае баз Tarantool(Users, Shards, ACL, Session) размер довольно мал и мы можем себе позволить помимо просто реплицирования мастера хранить в каждом ЦОДе все данные этих баз, что даст надежность при отказе всего ЦОДа.
2. В случае баз версий, логов, файлов и папок размер уже намного больше и стоит для надежности условно раз в день, отправлять дампы этих баз в соседний ЦОД, чтобы защититься от отказа. Clickhouse например автоматически умеет асинхронно реплицировать таблицы в разных кластерах.
3. S3 также реплицируем на соседние ЦОД

## Алгоритмы <a name="7"></a>

### Миграция

Пусть нам надо перенести пользователя с одного шарда на другой, для балансировки шардов или для уменьшения latency для пользователя. Это можно сделать копированием данных пользователя со скрытой реплики(для не нагрузки прода) на нужный шард и сменой записи в координаторе распредения по шардам. При ошибке чтения(когда файлы были удалены с шарда, но запись в координаторе не сменилась) бекэнд просто может проверить с того ли шарда он читает. Для исключения ошибок записи, при начале миграции будет оправлятся сигнал о пользователях, которых переносят, после записи бекэнд будет проверять не писал ли он во время миграции.

### Баланс шардов

Для избежания неравноменой нагрузки нужен алгоритм балансировки активных пользователей по шардам. Для этого можно для каждого пользователя раз некоторое время собирать по логам из ClickHouse'a статистику связанную с его активностью, нампример ```среднее_количество_запросов_в_день + среднее_количество_чтений_файлов_по_ссылкам_пользователя``` и подобрать эту статистику как чтобы от суммы статистик пользователей на одном шарде линейно зависел его среднедревный rps. Формулу этой статистики можно будет эмпирически подобрать методами машинного обучения. Скорей всего влияние этой статистики на нагрузку будет заметно только для очень активных пользователей, таких как коропоративные аккаунты.

По выведенной статистике можно будет балансировать нагрузку на шарды, перещая пользователей между ними так чтобы сумма статистик на них была примерно одинакова.

## Технологии <a name="8"></a>


| Технология        |                  Область применения                   |                                                    Мотивация                                                                        |
|-------------------|:-----------------------------------------------------:|:-----------------------------------------------------------------------------------------------------------------------------------:|
| Golang            |                       ЯП Backend'a                    | Встроенный паралелизим, популярен в индустрии как язык Backend'a                                                                    |
| React             |                       Frontend                        |                         Стандарт индустррии, большое количесво специалистов                                                         |
| Nginx             |                     L7 Балансировщий, ssl терминация  | Большое количество модулей, open-source                                                                                             |
| PostgreSQL        |                       База данных                     |                                                Самая популярная SQL база данных, репликация и контроль соединений через pgbouncer   |
| Kafka         | Асинхронный стриминговый сервис, брокер сообщений | Надежный, производительней по сравнению с RabbitMQ, отложенное эффективное выполнение задач, партицирование из коробки   |
| ClickHouse        | Хранение логов, версий файлов                         | Ориентирован на большой поток на запись, масштабируемость,стандарт как СУБД для хранения статистики                                                  |
| Tarantool         |           Key-value СУБД                              |                               Мастштабироение через Cartridge, в отличии от Redis устойчив к падениям сервера            |
| MinIO             |         Хранилище файлов                              |                                  OpenSource(Знанит можно будет доработать) реализация S3          |
| Memcached         |      Кеш                                              |            Ориентированность на работу именно с кешированием в отличие от Redis/Tarantool                                      |

## Схема проекта <a name="9"></a>

![image](https://github.com/wergit13/highload-google-drive/assets/102697969/909bc216-998b-484b-b154-069706ce885c)

 **API gateway** - выполняет роль шлюза, держит websocket сессии с пользователями, асинхронно логирует все запросы

**Auth service** - отвечает за авторизацию пользователей

**UserService** - отвечает за работу с пользователями

**FileService** - отвечает за работу с файлами и права доступа

**Migration Worker** - процесс балансировки пользователей по шардам

В кеше будем хранить, самые популярные файловые запросы: корневые каталоги дисков пользователей, списки файлов популярных общих папок.

Загрузка файлов в S3 проходит асинхронно, через брокер сообщений

## Обеспечение надежности <a name="10"></a>

### БД
- Реплицирование и шардирование баз данных
- Хранение копий файло X2 на других ДЦ
### Отказоустойчивость  балансировка:
- Rate limits ограничение зарпосов на nginx
- VRRP/CAPR резервирование балансировщиков
### Надежность сервисов
- Максимальное покрытие Unit/Integration/E2E-тестами.
- Graceful degradation(при высоких нагрузках отключение таких функций как: просмотр версий, поиск и прочих не основных)
- Резервирование железа
- Kafka сохранение сообщений на диске, гарантированная доставка.
- Кеш на частные запросы
### Observalvability
- Сбор метрик, логов с сервисов
- Настройка alert для 500 и падении сервисов

## Список серверов <a name="11"></a>

### Конфигурация для одного ЦОД

#### Балансировка

| Балансировщик          | RPS           | Net         |CPU|
|------------------------|---------------|-------------|-|
| Main                   | 255_000       | 40 Tbit/s   |510|
| Download               | 510_000       | 81 Tbit/s   |1020|

#### Сервисы

| Сервис                 | Нагрузка, RPS | Net         |CPU|
|------------------------|---------------|-------------|-|
| Auth                   | 300           | 2 Mbit/s    |4|
| User                   | 210           | 1.5 Mbit/s  |4|
| FileService            | 2500          | 40 Tbit/s   | 25|
| API Gateway            | 84_000        | 1 Gbit/s    |64|

#### БД

| База Данных            | Нагрузка, RPS |  Net        |  RAM|  Drive|CPU|
|------------------------|---------------|-------------|-----|-------|-|
| Postgres(на шард)      | 1500          | 20 Mbit/s   | 2 TB|  10 TB|64|
| User                   | 210           | 1.5 Mbit/s  |  700 GB  | 1TB  |4|
| Shards                 | 200           | 0.5 Mbit/s  | 128GB|  128GB |4|
| ACL                    | 230           | 0.5 Mbit/s  | 2 TB|  2TB  |4|
| Sessions               | 84_000        | 2 Mbit/s    |  128 GB | 128 GB |4|
| Clickhouse(Versions)   | 82_000        | 1.1 Gbit/s  |  -  |  0.6 PB   |-|
| Clickhouse(Logs)       | 250_000       | 4 Gbit/s    |  -  |  76 PB   |-|
| S3                     | 700_000       | 120 Tbit/s  |  -  |  8000 PB   |-|

 Объем расчитан на одну реплику и один ЦОД.
 Объем БД логов 




