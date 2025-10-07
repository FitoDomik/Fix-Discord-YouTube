### Быстрый старт

1. Запустите `service.bat` от имени администратора (ПКМ → Запуск от имени администратора).
2. В меню выберите пункт `1. Install Service`.
3. Укажите нужный профиль (`general*.bat`).
4. Для проверки состояния выберите `3. Check Status`.
5. Чтобы выключить/удалить службу, выберите `2. Remove Service`.

## Zapret Discord/YouTube (Windows, WinDivert) — Документация проекта

### 1. Структура проекта

```text
c:\Users\Slava\Downloads\1\
  ├─ bin\
  │  ├─ cygwin1.dll
  │  ├─ quic_initial_www_google_com.bin
  │  ├─ tls_clienthello_www_google_com.bin
  │  ├─ WinDivert.dll
  │  ├─ WinDivert64.sys
  │  └─ winws.exe
  ├─ general (ALT).bat
  ├─ general (ALT2).bat
  ├─ general (ALT3).bat
  ├─ general (ALT4).bat
  ├─ general (ALT5).bat
  ├─ general (ALT6).bat
  ├─ general (FAKE TLS ALT).bat
  ├─ general (FAKE TLS AUTO ALT).bat
  ├─ general (FAKE TLS AUTO ALT2).bat
  ├─ general (FAKE TLS AUTO).bat
  ├─ general (FAKE TLS).bat
  ├─ general (МГТС).bat
  ├─ general (МГТС2).bat
  ├─ general.bat
  ├─ lists\
  │  ├─ ipset-all.txt
  │  ├─ ipset-all.txt.backup
  │  └─ list-general.txt
  └─ service.bat
```

### 2. Описание файлов

- **bin\winws.exe**: основной исполняемый файл фильтрации/обхода DPI (работает поверх WinDivert).
  - **Назначение**: перехват/модификация сетевых пакетов по заданным правилам.
  - **Связи**: загружается из батников `general*.bat`; использует драйвер `WinDivert64.sys` и DLL `WinDivert.dll`; может подмешивать подготовленные бинарные шаблоны `*.bin`.

- **bin\WinDivert64.sys, WinDivert.dll**: драйвер и библиотека WinDivert.
  - **Назначение**: kernel/user-space перехват/инъекция пакетов в Windows.
  - **Связи**: обязательны для работы `winws.exe`.

- **bin\cygwin1.dll**: среда Cygwin, нужна `winws.exe`.

- **bin\quic_initial_www_google_com.bin, tls_clienthello_www_google_com.bin**: шаблоны «фальшивых» пакетов QUIC/TLS для техник dpi-desync.

- **service.bat**: интерактивный менеджер установки/удаления сервиса, обновлений и режимов.
  - **Назначение**: 
    - Установка Windows‑сервиса `zapret` из выбранного батника (парсит аргументы `winws.exe` из файла).
    - Проверка статуса (`status_zapret`), диагностика конфликтующих программ/служб, обновление списков.
    - Переключение игровых портов (`game filter`) и режима ipset, скачивание свежего `ipset-all.txt` с GitHub.
  - **Ключевые функции/секции**: `:service_install`, `:service_remove`, `:service_status`, `:service_diagnostics`, `:service_check_updates`, `:game_switch_status`/`:game_switch`, `:ipset_switch_status`/`:ipset_switch`, `:ipset_update`.
  - **Связи**: вызывается батниками `general*.bat` перед запуском; управляет файлами в `lists\` и маркером `bin\game_filter.enabled`.
  - **Краткая логика**: при установке сервиса читает выбранный `general*.bat`, извлекает все аргументы после `winws.exe`, подставляет абсолютные пути, переменные (`%BIN%`, `%LISTS%`, `%GameFilter%`), создаёт и запускает службу `zapret` с этими аргументами.

- **general (ALT3).bat** (и подобные `general*.bat`): сценарии запуска `winws.exe` в фоне с конкретными наборами правил.
  - **Назначение**: запуск обхода DPI/блокировок для HTTP/HTTPS/QUIC, Discord/STUN и игровых портов.
  - **Ключевые опции**: `--wf-tcp`, `--wf-udp`, `--filter-tcp`, `--filter-udp`, `--hostlist`, `--ipset`, `--filter-l7`, `--dpi-desync=*`, `--new`.
  - **Связи**: используют списки из `lists\`, бинарники из `bin\`, переменную `%GameFilter%`, которую выставляет `service.bat`.
  - **Краткая логика**: добавляет правила для доменов из `list-general.txt` и IP‑сетей из `ipset-all.txt`, применяя техники «desync» (fake/fakedsplit/multisplit и т.п.) на TCP 80/443 и UDP 443/игровые диапазоны.

- **lists\list-general.txt**: домены, к которым применяются host‑based правила (Discord, YouTube и сопутствующие).

- **lists\ipset-all.txt**: список подсетей/адресов для ipset‑режима. Пустой режим помечается строкой `0.0.0.0/32` (ipset выключен).

- **lists\ipset-all.txt.backup**: резервная копия полного списка ipset (большой набор префиксов CDN/облачных сетей и т.д.). Используется для включения ipset.

### 3. Архитектура проекта

- **Общая структура**: 
  - Скрипты `general*.bat` — профили запуска `winws.exe` с разными наборами правил.
  - `service.bat` — единая точка управления: проверка/диагностика, установка сервиса, переключение режимов, обновления списков.
  - `bin\` — исполняемые и драйверы фильтрации (WinDivert + вспомогательные .bin).
  - `lists\` — доменные и сетевые списки для таргетирования правил.

- **Потоки данных**:
  1. Пользователь запускает `general*.bat` или ставит сервис через `service.bat`.
  2. Скрипт вызывает: проверку статуса, проверку обновлений, загрузку статуса игрового фильтра.
  3. `winws.exe` поднимает правила WinDivert и начинает перехват пакетов.
  4. Фильтрация применяется к трафику, который совпадает с доменами (`hostlist`) или IP‑сетями (`ipset`), а также к L7 (Discord/STUN) и портовым диапазонам.
  5. DPI‑обход реализован через техники «desync»: вставка фальшивых пакетов QUIC/TLS, манипуляции TTL/последовательностями/разбиениями.

### 4. Библиотеки/фреймворки

- **WinDivert**: драйвер/библиотека для фильтрации и инъекции пакетов в Windows.
  - **Где используется**: `winws.exe` (вызов из `general*.bat`/сервиса).
- **Cygwin**: `cygwin1.dll` — среда выполнения для `winws.exe`.

### 5. Точка входа и запуск

- **Ручной запуск**: любой из профилей `general*.bat`.
  - Выполняется: проверка статуса `zapret`, проверка обновлений (с GitHub), загрузка статуса game filter, затем `start /min winws.exe ...` с правилами.
- **Сервисный режим**: через `service.bat` → Install Service → выбор нужного `general*.bat`.
  - Первым выполняется создание Windows‑сервиса `zapret` с аргументами, распарсенными из выбранного профиля; затем сервис стартует `winws.exe` при загрузке ОС.

### 6. Архитектурные особенности

- **Парсинг батников при установке сервиса**: `service.bat` извлекает аргументы из выбранного `general*.bat`, подставляя абсолютные пути и значения переменных. Это даёт единый способ запускать одинаковые профили как в интерактивном, так и в сервисном режимах.
- **Два режима таргетирования трафика**: по доменам (`hostlist`) и по адресным префиксам (`ipset`). Переключение ipset делается заменой `lists\ipset-all.txt` ↔ `ipset-all.txt.backup`.
- **Game filter toggle**: включение игровой фильтрации создаёт файл‑флаг `bin\game_filter.enabled` и расширяет порты на диапазон `1024-65535`.
- **DPI desync техники**: комбинации `fake`, `fakedsplit`, `multisplit`, `autottl`, `fooling=md5sig|badseq`, заготовки QUIC/TLS. Разные профили `general*.bat` варьируют параметры для устойчивости под разных провайдеров.

### 7. Примеры ключевых команд и фрагментов

Фрагмент профиля запуска (из `general (ALT3).bat`):

```14:22:c:\Users\Slava\Downloads\1\general (ALT3).bat
start "zapret: %~n0" /min "%BIN%winws.exe" --wf-tcp=80,443,%GameFilter% --wf-udp=443,50000-50100,%GameFilter% ^
--filter-udp=443 --hostlist="%LISTS%list-general.txt" --dpi-desync=fake --dpi-desync-repeats=6 --dpi-desync-fake-quic="%BIN%quic_initial_www_google_com.bin" --new ^
--filter-udp=50000-50100 --filter-l7=discord,stun --dpi-desync=fake --dpi-desync-repeats=6 --new ^
--filter-tcp=80 --hostlist="%LISTS%list-general.txt" --dpi-desync=fake,multisplit --dpi-desync-autottl=2 --dpi-desync-fooling=md5sig --new ^
--filter-tcp=443 --hostlist="%LISTS%list-general.txt" --dpi-desync=fakedsplit --dpi-desync-split-pos=1 --dpi-desync-autottl --dpi-desync-fooling=badseq --dpi-desync-repeats=8 --new ^
--filter-udp=443 --ipset="%LISTS%ipset-all.txt" --dpi-desync=fake --dpi-desync-repeats=6 --dpi-desync-fake-quic="%BIN%quic_initial_www_google_com.bin" --new ^
--filter-tcp=80 --ipset="%LISTS%ipset-all.txt" --dpi-desync=fake,multisplit --dpi-desync-autottl=2 --dpi-desync-fooling=md5sig --new ^
--filter-tcp=443,%GameFilter% --ipset="%LISTS%ipset-all.txt" --dpi-desync=fakedsplit --dpi-desync-split-pos=1 --dpi-desync-autottl --dpi-desync-fooling=badseq --dpi-desync-repeats=8 --new ^
--filter-udp=%GameFilter% --ipset="%LISTS%ipset-all.txt" --dpi-desync=fake --dpi-desync-autottl=2 --dpi-desync-repeats=10 --dpi-desync-any-protocol=1 --dpi-desync-fake-unknown-udp="%BIN%quic_initial_www_google_com.bin" --dpi-desync-cutoff=n2
```

Список доменов (`lists\list-general.txt`) для host‑фильтра:

```1:10:c:\Users\Slava\Downloads\1\lists\list-general.txt
cloudflare-ech.com
dis.gd
discord-attachments-uploads-prd.storage.googleapis.com
discord.app
discord.co
discord.com
discord.design
discord.dev
discord.gift
discord.gifts
```

Маркер выключенного ipset (`lists\ipset-all.txt`):

```1:2:c:\Users\Slava\Downloads\1\lists\ipset-all.txt
0.0.0.0/32
```


### 8. Предупреждения

- Возможно вмешательство сетевых фильтров/антивирусов/VPN. Используйте диагностику в `service.bat`.

- Требуются права администратора для установки сервиса и работы WinDivert.

