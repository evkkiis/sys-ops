# Пользователи, группы и sudo · отчёт

ФИО: Булаева Ева Игоревна
Группа: 9/2-РПО-25/1
Дата: 25.10.2026
Среда: Ubuntu 26.04.1 LTS, VirtualBox, пользователь `evkis`

## Учётная запись

Команды:

```bash
id
getent passwd evkis
```

![Учётная запись](screenshots/01-id.png)

Результат: `uid=1001(evkis) gid=1001(evkis)`, дополнительные группы `adm` и `sudo`. Строка passwd: `evkis:x:1001:1001::/home/evkis:/bin/bash`.

Подпишите семь полей своей строки /etc/passwd.

Вывод: 1) имя `evkis`. 2) пароль `x` — хеш лежит в shadow. 3) UID `1001`. 4) GID `1001`. 5) комментарий пустой. 6) домашний каталог `/home/evkis`. 7) оболочка `/bin/bash`.

## passwd и shadow

Команды:

```bash
ls -l /etc/passwd /etc/shadow
```

![passwd и shadow](screenshots/02-shadow.png)

Результат: `/etc/passwd` — `-rw-r--r--`, владелец `root`, группа `root`. `/etc/shadow` — `-rw-r-----`, владелец `root`, группа `shadow`.

Почему shadow закрыт для остальных пользователей.

Вывод: в shadow лежат хеши паролей. Читать его могут только root и группа `shadow`. В passwd на месте пароля стоит `x`.

## Группы

Команды:

```bash
groups
getent group sudo
```

![Группы](screenshots/03-groups.png)

Результат: `groups` вывел `evkis adm sudo`. Группа sudo: `sudo:x:27:evkis`.

Где основная группа, где дополнительные.

Вывод: основная группа — `evkis`, GID 1001. Она в четвёртом поле passwd и в `gid=` у `id`. `adm` и `sudo` дополнительные. Новые файлы получают основную группу, если каталог без setgid.

## Правила sudo

Команды:

```bash
sudo -l
```

![Правила sudo](screenshots/04-sudo.png)

Результат: на машине `evkis` пользователю `evkis` разрешено `(ALL : ALL) ALL`.

Какое правило даёт вам права администратора.

Вывод: правило `(ALL : ALL) ALL` разрешает любую команду от имени любого пользователя. Оно действует, потому что `evkis` входит в группу `sudo`.

## Команда mobile-app

Команды:

```bash
getent group mobile-dev mobile-qa sudo
id lead
id dev
id qa
id guest
```

![Команда mobile-app](screenshots/05-team.png)

Результат: `mobile-dev` — lead и dev, `mobile-qa` — lead и qa, в `sudo` есть `evkis` и lead. У guest в `id` только группа `guest`.

Совпадают ли группы со схемой ролей.

Вывод: совпадают. lead работает с кодом и отчётами и администрирует систему. dev только в `mobile-dev`. qa только в `mobile-qa`. guest не входит ни в проектные группы, ни в sudo.

## Каталоги проекта

Команды:

```bash
ls -l /srv/mobile
```

![Каталоги проекта](screenshots/06-dirs.png)

Результат: `src` — `drwxrws---`, группа `mobile-dev`. `reports` — `drwxrws---`, группа `mobile-qa`.

Что означает каждая цифра в 2770.

Вывод: `2` — setgid, новый файл наследует группу каталога. `7` — владельцу rwx. `7` — группе rwx. `0` — остальным ничего. Поэтому dev пишет в `src` и не открывает `reports`.

## Тесты доступа

Команды:

```bash
sudo -u lead touch /srv/mobile/src/lead.txt
sudo -u lead ls /srv/mobile/reports
sudo -u dev touch /srv/mobile/src/main.kt
sudo -u dev ls /srv/mobile/reports
sudo -u qa touch /srv/mobile/reports/test-1.md
sudo -u qa ls /srv/mobile/src
sudo -u guest ls /srv/mobile/src
sudo -u guest ls /srv/mobile/reports
```

![Тесты доступа](screenshots/07-access.png)

Результат: lead создал файл в `src` и открыл `reports`. dev создал `main.kt` и получил `Permission denied` на `reports`, код 2. qa создал `test-1.md` и получил `Permission denied` на `src`. guest получил `Permission denied` на оба каталога.

Результаты таблицы тестов доступа.

Вывод: фактический результат совпал с ожидаемым по схеме ролей. Таблица целиком — в `access-tests.md`.

## Проверка

Команды:

```bash
sudo bash ~/Downloads/team-lab-check.sh
```

![Проверка](screenshots/08-check.png)

Результат: все 21 пункт `OK`, строка `Итог: 21 из 21`.

Итог проверки; какие пункты потребовали исправления.

Вывод: исправлять схему не пришлось. Группы, учётные записи, членство и права `2770` сошлись с первого запуска.

## Отказ sudo в журнале

Команды:

```bash
sudo -n -u guest sudo -n -l
```

![Отказ sudo в журнале](screenshots/09-denied.png)

Результат: `sudo: Sorry, user guest may not run sudo on evkis.`

Кто, когда и какую команду пытался выполнить.

Вывод: guest не входит в sudo, поэтому список прав ему не показывают. В Ubuntu 26.04 текст отказа — `Sorry, user guest may not run sudo on evkis.`

## Почему постоянная работа под root опасна

1. Ошибка в команде удаления под root не останавливается правами каталога и может стереть систему.
2. Чужая root-сессия открывает `/etc/shadow` и любую учётную запись. Обычному пользователю shadow закрыт.
3. Общий вход под root не оставляет в журнале имя человека. `sudo` пишет, кто запустил команду, поэтому отказ guest виден отдельно.

## Свой проект

Модель бота конспектов, скрипт и пять тестов — в `access-model.md` и `setup-team.sh`. Снимок вывода `id` и `ls -l` после запуска скрипта:

![setup-team.sh](screenshots/10-setup.png)

## Практика-тест

https://algorthimization-course-vvodnoe.vercel.app/op05-operacionnye-sistemy-i-sredy/testy/tema-04-05/index.html

Код:

## Перед сдачей

- [+] Проверка: 21 из 21.
- [+] access-tests.md заполнен.
- [ ] Учебные учётные записи удалены.
