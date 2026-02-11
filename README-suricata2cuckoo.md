# Suricata filestore → Cuckoo Sandbox (OPNsense)

Демон для OPNsense: следит за появлением новых файлов в Suricata filestore (`/var/log/suricata/filestore/00` … `ff`) и отправляет их в Cuckoo Sandbox по REST API (как [CuckooMX](https://github.com/kolixxx/cmxurapiaig)).

## Требования

- OPNsense (FreeBSD) с настроенной Suricata и **file-extraction**
- Cuckoo Sandbox с включённым API (как в CuckooMX)
- Доступ к командной строке OPNsense (SSH или консоль)

## Пошаговая установка

### Шаг 1: Установка зависимостей

Подключитесь к OPNsense по SSH и выполните:

```bash
# Обновление списка пакетов
pkg update

# Установка обязательных Perl-модулей
pkg install p5-libwww p5-HTTP-Message p5-XML-XPath

# Установка опциональных модулей
# p5-File-LibMagic - для определения типа файла по содержимому (рекомендуется)
pkg install p5-File-LibMagic || echo "p5-File-LibMagic не найден, будет использоваться пакет 'exe' по умолчанию"
```

**Важно о p5-IO-KQueue:**

Пакет `p5-IO-KQueue` обычно **недоступен** в стандартных репозиториях OPNsense. Это нормально — скрипт автоматически переключится на режим **polling** (опрос директорий каждые N секунд), который работает без дополнительных зависимостей.

Если вы хотите использовать kqueue (более эффективный мониторинг), можно попробовать установить через CPAN:

```bash
# Установка CPAN (если еще не установлен)
pkg install p5-CPAN

# Установка IO::KQueue через CPAN
cpan IO::KQueue
```

**Примечание:** 
- Если `p5-IO-KQueue` недоступен — скрипт работает в режиме **polling** (указано в конфиге `watch-method=polling` или автоматически определяется)
- Если `p5-File-LibMagic` недоступен — скрипт использует пакет `exe` по умолчанию для всех файлов (Cuckoo всё равно определит тип по содержимому)

### Шаг 2: Скачивание проекта с GitHub

```bash
# Переход в рабочую директорию
cd /tmp

# Скачивание проекта (если git установлен)
pkg install git
git clone https://github.com/kolixxx/osfetcsndbx.git

# Или скачивание через curl/wget (если git недоступен)
# Скачайте архив с GitHub и распакуйте, либо скопируйте файлы вручную
```

### Шаг 3: Копирование файлов в системную директорию

```bash
# Создание директории для скрипта
mkdir -p /usr/local/etc/suricata2cuckoo

# Копирование файлов из скачанного проекта
cp /tmp/osfetcsndbx/suricata2cuckoo.pl /usr/local/etc/suricata2cuckoo/
cp /tmp/osfetcsndbx/suricata2cuckoo.conf /usr/local/etc/suricata2cuckoo/

# Установка прав доступа
chmod +x /usr/local/etc/suricata2cuckoo/suricata2cuckoo.pl
chmod 644 /usr/local/etc/suricata2cuckoo/suricata2cuckoo.conf
chown root:wheel /usr/local/etc/suricata2cuckoo/suricata2cuckoo.pl
chown root:wheel /usr/local/etc/suricata2cuckoo/suricata2cuckoo.conf
```

### Шаг 4: Настройка прав доступа к Suricata filestore

Скрипт должен иметь доступ на чтение файлов из директории Suricata:

```bash
# Проверка текущих прав на filestore
ls -ld /var/log/suricata/filestore

# Если директория принадлежит другому пользователю (например, suricata),
# нужно либо:
# 1. Запускать скрипт от имени этого пользователя (в rc.d скрипте)
# 2. Или добавить пользователя, от которого запускается скрипт, в группу suricata
# 3. Или изменить права на filestore (менее безопасно)

# Вариант 1: Добавление пользователя root в группу suricata (если она существует)
pw groupmod suricata -m root

# Вариант 2: Если скрипт запускается от root (по умолчанию), 
# root обычно имеет доступ ко всем файлам
```

**Важно:** Убедитесь, что скрипт может читать файлы из `/var/log/suricata/filestore/00` … `ff`. Проверьте:

```bash
# Тест доступа (OPNsense использует sh, поэтому используем простой синтаксис)
ls -la /var/log/suricata/filestore/00/ | head -5

# Или просто проверьте, что директория существует и доступна
ls /var/log/suricata/filestore/00/
```

### Шаг 5: Настройка конфигурации

Отредактируйте файл конфигурации:

```bash
vi /usr/local/etc/suricata2cuckoo/suricata2cuckoo.conf
```

Или через nano:

```bash
nano /usr/local/etc/suricata2cuckoo/suricata2cuckoo.conf
```

Настройте следующие параметры:

- **`<path>`** — путь к filestore Suricata (по умолчанию `/var/log/suricata/filestore`)
- **`<watch-method>`** — метод мониторинга:
  - `polling` — опрос директорий (по умолчанию, работает всегда)
  - `kqueue` — эффективный мониторинг через kqueue (требует установленный `IO::KQueue` через CPAN)
- **`<api-url>`** — URL API Cuckoo (например `http://192.168.1.100:8090`)
- **`<api-token>`** — токен из `conf/cuckoo.conf` Cuckoo (если включена аутентификация, иначе оставьте пустым)
- **`<guest>`** — имя машины Cuckoo (например `Cuckoo1`)

Пример конфигурации:

```xml
<filestore>
  <path>/var/log/suricata/filestore</path>
  <watch-method>kqueue</watch-method>  <!-- или "polling" -->
  <poll-interval>5</poll-interval>
  <file-settle-time>2</file-settle-time>
</filestore>

<cuckoo>
  <api-url>http://192.168.1.100:8090</api-url>
  <api-token>your_token_here</api-token>
  <guest>Cuckoo1</guest>
</cuckoo>
```

**Если вы установили IO::KQueue через CPAN** и хотите использовать kqueue:

1. Измените в конфиге:
   ```xml
   <watch-method>kqueue</watch-method>
   ```

2. Проверьте, что модуль доступен:
   ```bash
   perl -MIO::KQueue -e "print 'IO::KQueue доступен\n'"
   ```

3. Если модуль не найден, проверьте путь установки CPAN:
   ```bash
   perl -V | grep @INC
   ```

4. После изменения конфига перезапустите сервис:
   ```bash
   service suricata2cuckoo restart
   ```

5. Проверьте логи — должно быть сообщение "Watching X filestore dirs with kqueue":
   ```bash
   tail -20 /var/log/system.log | grep suricata2cuckoo
   ```

### Шаг 6: Тестовый запуск

Перед запуском как сервис, проверьте работу скрипта в режиме отладки:

```bash
# Запуск в переднем плане (для проверки)
/usr/local/etc/suricata2cuckoo/suricata2cuckoo.pl \
  -c /usr/local/etc/suricata2cuckoo/suricata2cuckoo.conf \
  --no-fork
```

Скрипт должен запуститься и начать мониторинг. Вы должны увидеть сообщение:
```
Starting suricata2cuckoo daemon (filestore=..., method=...)
```

Если есть ошибки — они будут видны сразу. Проверьте логи (на OPNsense могут быть в разных местах):

```bash
# Попробуйте все варианты (один из них должен работать):
tail -f /var/log/system.log | grep suricata2cuckoo
tail -f /var/log/messages | grep suricata2cuckoo
tail -f /var/log/syslog | grep suricata2cuckoo
```

Если всё работает корректно, остановите тестовый запуск (Ctrl+C) и переходите к следующему шагу.

### Шаг 7: Настройка автозапуска через rc.d

Скопируйте пример rc.d скрипта:

```bash
cp /tmp/osfetcsndbx/rc.d.suricata2cuckoo.example /usr/local/etc/rc.d/suricata2cuckoo
chmod +x /usr/local/etc/rc.d/suricata2cuckoo
```

Отредактируйте скрипт и укажите путь к конфигу (если отличается от стандартного):

```bash
vi /usr/local/etc/rc.d/suricata2cuckoo
```

Добавьте в `/etc/rc.conf.local` (или создайте файл, если его нет):

```bash
echo 'suricata2cuckoo_enable="YES"' >> /etc/rc.conf.local
echo 'suricata2cuckoo_config="/usr/local/etc/suricata2cuckoo/suricata2cuckoo.conf"' >> /etc/rc.conf.local
```

Или отредактируйте файл вручную:

```bash
vi /etc/rc.conf.local
```

Добавьте строки:

```
suricata2cuckoo_enable="YES"
suricata2cuckoo_config="/usr/local/etc/suricata2cuckoo/suricata2cuckoo.conf"
```

### Шаг 8: Запуск сервиса

```bash
# Запуск сервиса
service suricata2cuckoo start

# Проверка статуса
service suricata2cuckoo status

# Просмотр логов
tail -f /var/log/system.log | grep suricata2cuckoo
```

### Шаг 9: Проверка работы

```bash
# Проверка, что процесс запущен
ps aux | grep suricata2cuckoo

# Проверка логов (на OPNsense логи могут быть в разных местах)
# Попробуйте все варианты:
grep suricata2cuckoo /var/log/system.log 2>/dev/null | tail -20
grep suricata2cuckoo /var/log/messages 2>/dev/null | tail -20
grep suricata2cuckoo /var/log/syslog 2>/dev/null | tail -20

# Или через dmesg (если есть ошибки при запуске)
dmesg | grep suricata2cuckoo
```

**Если процесс не запущен или логи не видны:**

1. **Проверьте, что rc.d скрипт существует и исполняемый:**
   ```bash
   ls -l /usr/local/etc/rc.d/suricata2cuckoo
   # Должно быть: -rwxr-xr-x
   ```

2. **Проверьте, что путь к скрипту правильный в rc.d:**
   ```bash
   grep command= /usr/local/etc/rc.d/suricata2cuckoo
   # Должно быть: command="/usr/local/etc/suricata2cuckoo/suricata2cuckoo.pl"
   ```

3. **Попробуйте запустить скрипт вручную для диагностики:**
   ```bash
   # Запуск в переднем плане (вы увидите все ошибки)
   /usr/local/etc/suricata2cuckoo/suricata2cuckoo.pl \
     -c /usr/local/etc/suricata2cuckoo/suricata2cuckoo.conf \
     --no-fork
   ```
   
   Если есть ошибки — они будут видны сразу. Типичные проблемы:
   - Не найден конфиг → проверьте путь
   - Не найден модуль Perl → установите зависимости
   - Нет доступа к filestore → проверьте права

4. **Проверьте, что сервис включен в rc.conf:**
   ```bash
   grep suricata2cuckoo /etc/rc.conf.local
   # Должно быть:
   # suricata2cuckoo_enable="YES"
   # suricata2cuckoo_config="/usr/local/etc/suricata2cuckoo/suricata2cuckoo.conf"
   ```

5. **Попробуйте запустить сервис вручную через rc.d:**
   ```bash
   /usr/local/etc/rc.d/suricata2cuckoo start
   # Проверьте вывод на ошибки
   ```

6. **Проверьте, что daemon может создать pidfile:**
   ```bash
   ls -ld /var/run
   # Если нет прав, может быть проблема с pidfile
   ```

## Управление сервисом

```bash
# Запуск
service suricata2cuckoo start

# Остановка
service suricata2cuckoo stop

# Перезапуск
service suricata2cuckoo restart

# Проверка статуса
service suricata2cuckoo status
```

## Мониторинг

- **kqueue** — используется, если установлен `p5-IO-KQueue` и в конфиге указано `watch-method` = `kqueue`. Реакция на появление файлов без опроса диска (рекомендуется).
- **polling** — если kqueue недоступен или в конфиге указан `polling`, директории опрашиваются раз в `poll-interval` секунд.

При появлении нового файла демон выжидает `file-settle-time` секунд (чтобы Suricata успела дописать файл), затем отправляет его в Cuckoo через `POST /tasks/create/file` с правильным расширением файла (например `sample.exe`, `sample.doc`) для корректной работы в гостевой Windows ВМ.

## Логирование

Логи пишутся в syslog:
- Программа: `suricata2cuckoo`
- Facility: из конфига (по умолчанию `daemon`)

Просмотр логов (на OPNsense логи могут быть в разных местах):

```bash
# Попробуйте все варианты (один из них должен работать):
# Вариант 1: system.log
grep suricata2cuckoo /var/log/system.log | tail -20

# Вариант 2: messages (часто используется на FreeBSD/OPNsense)
grep suricata2cuckoo /var/log/messages | tail -20

# Вариант 3: syslog
grep suricata2cuckoo /var/log/syslog | tail -20

# Мониторинг в реальном времени (попробуйте все варианты):
tail -f /var/log/messages | grep suricata2cuckoo
# или
tail -f /var/log/system.log | grep suricata2cuckoo
```

## Устранение проблем

### Скрипт не запускается

1. Проверьте права доступа:
   ```bash
   ls -l /usr/local/etc/suricata2cuckoo/suricata2cuckoo.pl
   # Должно быть: -rwxr-xr-x
   ```

2. Проверьте наличие всех зависимостей:
   ```bash
   perl -MLWP::UserAgent -e "print 'OK\n'"
   perl -MHTTP::Request::Common -e "print 'OK\n'"
   perl -MXML::XPath -e "print 'OK\n'"
   ```

3. Проверьте синтаксис скрипта:
   ```bash
   perl -c /usr/local/etc/suricata2cuckoo/suricata2cuckoo.pl
   ```

### Файлы не отправляются в Cuckoo

1. Проверьте доступность API Cuckoo:
   ```bash
   curl http://IP_CUCKOO:8090/cuckoo/status
   ```

2. Проверьте токен аутентификации в конфиге

3. Проверьте логи на наличие ошибок API (попробуйте все варианты):
   ```bash
   grep "Cuckoo API" /var/log/messages | tail -20
   grep "Cuckoo API" /var/log/system.log | tail -20
   ```

### Нет доступа к filestore

1. Проверьте права на директорию:
   ```bash
   ls -ld /var/log/suricata/filestore
   ```

2. Проверьте доступ к файлам:
   ```bash
   ls /var/log/suricata/filestore/00/ | head -1
   ```

3. Если нужно, запустите скрипт от имени пользователя suricata (измените rc.d скрипт)
