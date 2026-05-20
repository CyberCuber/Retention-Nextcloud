Вот полное описание для вашей базы знаний. Можно копировать целиком.

---

Nextcloud Auto-Retention: автоудаление файлов старше 60 дней (snap-версия без upload_time)

Задача

Удалять файлы, загруженные в Nextcloud более 60 дней назад. Удаление производится с двойной проверкой: по датам в базе данных и по дате создания файла в файловой системе.

Особенности окружения

· Установка: snap-версия Nextcloud
· СУБД: MySQL
· Версия Nextcloud: без поля upload_time в таблице oc_filecache (старая структура БД)
· Пользователь: Admin

Почему нельзя использовать только mtime/ctime на файлах

При копировании старых файлов их mtime (дата изменения) сохраняется исходной. Файл, загруженный сегодня, но с mtime годичной давности, был бы мгновенно удалён скриптом, использующим только mtime.

Логика защиты (двойная проверка)

Скрипт использует два уровня проверки перед удалением:

1. Проверка по БД: поля mtime и storage_mtime в таблице oc_filecache должны быть старше 60 дней.
2. Проверка по ФС: атрибут ctime (время создания файла на диске) также должен быть старше 60 дней.

Файл удаляется только при совпадении обоих условий. Это исключает удаление сегодня загруженных файлов со старыми метаданными.

Пути и переменные

Переменная Значение Описание
NEXTCLOUD_USER Admin Имя пользователя Nextcloud
DATA_PATH /var/snap/nextcloud/common/nextcloud/data/Admin/files Путь к файлам пользователя
OCC /snap/bin/nextcloud.occ Путь к утилите occ
SOCKET /tmp/snap-private-tmp/snap.nextcloud/tmp/sockets/mysql.sock Путь к Unix-сокету MySQL

Полный код скрипта

```bash
#!/bin/bash

NEXTCLOUD_USER="Admin"
DATA_PATH="/var/snap/nextcloud/common/nextcloud/data/$NEXTCLOUD_USER/files"
OCC="/snap/bin/nextcloud.occ"
SOCKET="/tmp/snap-private-tmp/snap.nextcloud/tmp/sockets/mysql.sock"

# Шаг 1: Удаление файлов с двойной проверкой
/usr/bin/mysql -u nextcloud \
    -p'ПАРОЛЬ_БД' \
    -S "$SOCKET" nextcloud -N -B -e "
    SELECT CONCAT('$DATA_PATH', fc.path)
    FROM oc_filecache fc
    JOIN oc_storages s ON fc.storage = s.numeric_id
    WHERE s.id LIKE 'home::%$NEXTCLOUD_USER'
      AND fc.mimetype != 2
      AND fc.mtime > 0
      AND fc.mtime < UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL 60 DAY))
      AND fc.storage_mtime < UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL 60 DAY))
      AND fc.path NOT LIKE 'files_trashbin/%'
      AND fc.path NOT LIKE 'files_versions/%';
" | while read f; do
    if [ -f "$f" ]; then
        FILE_CTIME=$(stat -c %Z "$f" 2>/dev/null)
        CUTOFF=$(date -d "60 days ago" +%s)
        if [ "$FILE_CTIME" -lt "$CUTOFF" ]; then
            rm -f "$f"
        fi
    fi
done

# Шаг 2: Удаление пустых папок
find "$DATA_PATH" -mindepth 2 -type d -empty -delete 2>/dev/null

# Шаг 3: Синхронизация базы данных
$OCC files:scan "$NEXTCLOUD_USER" -n -q
$OCC files:cleanup -n -q
```

Построчное объяснение

Переменные окружения

```bash
NEXTCLOUD_USER="Admin"
```

Имя пользователя Nextcloud. Используется в пути к данным и в SQL-запросе.

```bash
DATA_PATH="/var/snap/nextcloud/common/nextcloud/data/$NEXTCLOUD_USER/files"
```

Абсолютный путь к директории с файлами на диске.

```bash
OCC="/snap/bin/nextcloud.occ"
```

Путь к консольной утилите Nextcloud.

```bash
SOCKET="/tmp/snap-private-tmp/snap.nextcloud/tmp/sockets/mysql.sock"
```

Путь к Unix-сокету MySQL внутри изолированной среды snap.

---

Шаг 1: SQL-запрос и удаление файлов

```bash
/usr/bin/mysql -u nextcloud \
    -p'ПАРОЛЬ_БД' \
    -S "$SOCKET" nextcloud -N -B -e "..."
```

Параметры mysql:

Параметр Описание
-u nextcloud Имя пользователя БД
-p'...' Пароль (без пробела после -p)
-S "$SOCKET" Подключение через Unix-сокет
-N Не выводить заголовки столбцов
-B Пакетный режим
-e "..." Выполнить SQL-запрос

SQL-запрос

```sql
SELECT CONCAT('$DATA_PATH', fc.path)
FROM oc_filecache fc
JOIN oc_storages s ON fc.storage = s.numeric_id
WHERE s.id LIKE 'home::%$NEXTCLOUD_USER'
  AND fc.mimetype != 2
  AND fc.mtime > 0
  AND fc.mtime < UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL 60 DAY))
  AND fc.storage_mtime < UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL 60 DAY))
  AND fc.path NOT LIKE 'files_trashbin/%'
  AND fc.path NOT LIKE 'files_versions/%';
```

Таблицы:

Таблица Назначение
oc_filecache Метаданные файлов: путь, размер, mimetype, mtime
oc_storages Хранилища пользователей (тип home::username)

Условия WHERE:

Условие Назначение
s.id LIKE 'home::%$NEXTCLOUD_USER' Только файлы пользователя Admin
fc.mimetype != 2 Исключить папки
fc.mtime > 0 Исключить записи без даты
fc.mtime < UNIX_TIMESTAMP(...) mtime старше 60 дней
fc.storage_mtime < UNIX_TIMESTAMP(...) storage_mtime старше 60 дней
fc.path NOT LIKE 'files_trashbin/%' Не трогать корзину
fc.path NOT LIKE 'files_versions/%' Не трогать версии файлов

Цикл удаления с проверкой ctime

```bash
| while read f; do
    if [ -f "$f" ]; then
        FILE_CTIME=$(stat -c %Z "$f" 2>/dev/null)
        CUTOFF=$(date -d "60 days ago" +%s)
        if [ "$FILE_CTIME" -lt "$CUTOFF" ]; then
            rm -f "$f"
        fi
    fi
done
```

Команда Описание
while read f Читает путь к файлу из вывода mysql
[ -f "$f" ] Проверка существования файла
stat -c %Z "$f" Получает ctime файла (дата создания в ФС) в формате UNIX timestamp
date -d "60 days ago" +%s Вычисляет timestamp 60-дневной давности
[ "$FILE_CTIME" -lt "$CUTOFF" ] Сравнивает: ctime должен быть старше 60 дней
rm -f "$f" Принудительное удаление

Логика защиты: если сегодня загружен файл со старым mtime, его ctime будет сегодняшним. Условие ctime < 60 дней назад не выполнится, и файл не удалится.

---

Шаг 2: Удаление пустых папок

```bash
find "$DATA_PATH" -mindepth 2 -type d -empty -delete 2>/dev/null
```

Аргумент Описание
-mindepth 2 Не трогать корневую папку files
-type d Только директории
-empty Только пустые
-delete Удалить
2>/dev/null Подавить ошибки

---

Шаг 3: Синхронизация базы данных

```bash
$OCC files:scan "$NEXTCLOUD_USER" -n -q
```

Сканирует файловую систему пользователя и обновляет таблицу oc_filecache. Устраняет расхождения между диском и БД после прямого удаления файлов.

```bash
$OCC files:cleanup -n -q
```

Удаляет из БД записи о файлах, которых больше нет на диске.

---

Установка

```bash
# 1. Создать файл
sudo nano /usr/local/bin/nextcloud_retention.sh

# 2. Вставить код, сохранить

# 3. Права (внутри пароль БД — закрыть от других)
sudo chmod 700 /usr/local/bin/nextcloud_retention.sh

# 4. Тестовый запуск
sudo /usr/local/bin/nextcloud_retention.sh
```

Cron (ежедневно в 3:00)

```bash
sudo crontab -e
```

```
0 3 * * * /usr/local/bin/nextcloud_retention.sh
```

Срок хранения

Изменить INTERVAL 60 DAY в SQL-запросе и 60 days ago в строке CUTOFF:

Срок Замена в SQL Замена в CUTOFF
30 дней INTERVAL 30 DAY 30 days ago
90 дней INTERVAL 90 DAY 90 days ago
1 год INTERVAL 1 YEAR 365 days ago

Структура таблицы oc_filecache (данная версия)

Поле Тип Описание
fileid bigint Первичный ключ
storage bigint ID хранилища
path varchar(4000) Путь к файлу
mimetype bigint Тип (2 = папка)
size bigint Размер
mtime bigint Время изменения файла (UNIX timestamp)
storage_mtime bigint Время изменения в хранилище (UNIX timestamp)

Важно: в данной версии отсутствует поле upload_time, поэтому используется двойная проверка через mtime + storage_mtime + ctime.

Возможные ошибки

Ошибка Причина Решение
ERROR 1054: Unknown column 'upload_time' В БД нет такого поля Использовать исправленный скрипт (выше)
mysql: command not found Не установлен клиент apt install mysql-client
Can't connect through socket Неверный путь к сокету find / -name "mysql.sock" 2>/dev/null
Access denied Неверный пароль БД Проверить в config.php
