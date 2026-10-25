# Права доступа, владельцы и umask · отчёт

ФИО: Булаева Ева ИГоревна
Группа: 9/2-РПО-25/1
Дата: 25.10.2026
Среда: Ubuntu 26.04.1 LTS, VirtualBox, пользователь `evkis`

## Строка ls -l

Команды:

```bash
ls -l
stat -c '%A %a %U %G %n' report.txt docs /etc/shadow
ls -ld /tmp ~
```

![Строка ls -l](screenshots/01-ls-l.png)

Результат: у `report.txt` права `644`, владелец и группа `evkis`. `docs` — `755`. `/etc/shadow` — `640`, группа `shadow`. Домашний каталог `/home/evkis`: `drwxr-x---`. `/tmp`: `drwxrwxrwt`.

Подпишите поля одной строки: тип, три тройки, владелец, группа.

Вывод: у `report.txt` первый знак `-` — обычный файл. Тройки `rw-` `r--` `r--`: владелец читает и пишет, группа и остальные только читают. Владелец и группа — `evkis`. Буква `t` в конце `/tmp` — sticky-бит.

## Каталог без x и без r

Команды:

```bash
chmod u-x docs
ls docs
cat docs/notes.txt
cd docs
chmod u+x docs
chmod u-r docs
ls docs
cat docs/notes.txt
```

![Каталог без x и без r](screenshots/02-dir-rx.png)

Результат: без `x` `ls docs` показал имя `notes.txt`, а `cat` и `cd` ответили `Permission denied`. Без `r`, но с `x`, `ls` написал `cannot open directory`, а `cat docs/notes.txt` вывел `Meeting notes`.

Объясните каждую ошибку: что запрещено без x и без r.

Вывод: `x` у каталога — право прохода. Без него известное имя ещё видно, но открыть файл и войти нельзя. `r` — право прочитать список имён. Без него `ls` каталог не открывает, но файл по известному имени читается.

## Удаление и право w

Команды:

```bash
chmod u-w docs
touch docs/new.txt
rm docs/notes.txt
chmod u+w docs
chmod a-w report.txt
rm -f report.txt
```

![Удаление и право w](screenshots/03-dir-w.png)

Результат: без `w` на `docs` и `touch`, и `rm docs/notes.txt` получили `Permission denied`. После `chmod a-w` файл стал `-r--r--r--`. `rm -f` всё равно удалил его: `ls report.txt` ответил `No such file or directory`.

Почему удаление файла решает каталог, а не файл.

Вывод: имя файла лежит в каталоге. Удаление меняет каталог, поэтому нужно `w` на каталог, а не на сам файл. Запрет записи на `report.txt` его не спас.

## chmod буквами и числом

Команды:

```bash
chmod -v go-r db.env
chmod -v u=rwx,g=rx,o= backup.sh
chmod -v g+w report.txt
chmod -v 640 report.txt
chmod -v 700 private
chmod -v 755 backup.sh
chmod -v 600 db.env
```

![chmod буквами и числом](screenshots/04-chmod.png)

Результат: `go-r` перевёл `db.env` из `0644` в `0600`. `u=rwx,g=rx,o=` поставил `backup.sh` в `0750`. `g+w` сделал `0664`. Число `640` вернуло `rw-r-----`, `700` закрыло `private`, `755` открыло скрипт. `600` для `db.env` уже стояло: `retained`.

Переведите три своих примера из букв в число и обратно.

Вывод:

| Буквы | Число | Что изменилось |
|---|---|---|
| `go-r` от `rw-r--r--` | `600` · `rw-------` | у группы и остальных снято чтение |
| `u=rwx,g=rx,o=` | `750` · `rwxr-x---` | владелец всё, группа чтение и запуск, остальным ничего |
| `640` | `rw-r-----` | владелец 6, группа 4, остальные 0 |

Обратный перевод `755` — `rwxr-xr-x`.

## Владелец и группа

Команды:

```bash
chown anna report.txt
sudo chown anna report.txt
sudo chown anna:devteam db.env
chgrp adm backup.sh
chgrp shadow backup.sh
```

![Владелец и группа](screenshots/05-chown.png)

Результат: `chown anna` без sudo — `Operation not permitted`. С sudo владелец `report.txt` стал `anna`, у `db.env` владелец `anna` и группа `devteam`. `chgrp adm backup.sh` прошёл, группа стала `adm`. `chgrp shadow` — снова `Operation not permitted`.

Почему chown без sudo запрещён, а chgrp на свою группу разрешён.

Вывод: смену владельца разрешает только root. Сменить группу своего файла можно на ту группу, в которой сам состоишь. `evkis` входит в `adm`, поэтому `chgrp adm` принят. В `shadow` он не входит, поэтому система отказала.

## umask

Команды:

```bash
umask
touch new-file.txt
mkdir new-dir
ls -ld new-file.txt new-dir
(umask 027; touch team.txt; mkdir team-dir; ls -ld team.txt team-dir)
```

![umask](screenshots/06-umask.png)

Результат: текущая маска `0002`. Новый файл `-rw-rw-r--`, каталог `drwxrwxr-x`. С маской `027` файл стал `-rw-r-----` (`640`), каталог `drwxr-x---` (`750`). После подоболочки маска снова `0002`.

Права файла и каталога при маске 0027 — расчёт по тройкам.

Вывод: у файла база `666`, у каталога `777`. Маска `027` снимает у группы `w`, у остальных `rwx`.

| | Владелец | Группа | Остальные | Итог |
|---|---|---|---|---|
| файл `666` | `rw-` | `r--` | `---` | `640` |
| каталог `777` | `rwx` | `r-x` | `---` | `750` |

## Каталоги проекта

Команды:

```bash
ls -l /srv/shop
id alex
id bella
id chris
```

![Каталоги проекта](screenshots/07-shop-dirs.png)

Результат: `releases` — `drwxrwsr-x`, `src` — `drwxrws---`, `uploads` — `drwxrws--T`, группа всех трёх `shop-dev`. alex и bella в `shop-dev`, у chris только группа `chris`.

Что означает каждая буква в drwxrws--T.

Вывод: `d` — каталог. `rwx` — владельцу всё. `rws` — группе чтение, запись и проход, `s` вместо `x` значит setgid. `--T` — остальным ничего, заглавная `T` значит sticky без `x` для остальных. Число `3770`: `3` = setgid `2` плюс sticky `1`, затем `7`, `7` и `0`.

## Тесты доступа

Команды:

```bash
sudo -u alex bash -c 'umask 002; echo "print(1)" > /srv/shop/src/app.py'
echo 'print(2)' | sudo -u bella tee -a /srv/shop/src/app.py
sudo -u alex bash -c 'echo TOKEN=lab > /srv/shop/src/config.env; chmod 600 /srv/shop/src/config.env'
sudo -u bella cat /srv/shop/src/config.env
sudo -u alex bash -c 'echo release-1 > /srv/shop/releases/v1.txt'
sudo chown bella /srv/shop/releases/v1.txt
sudo -u chris cat /srv/shop/releases/v1.txt
sudo -u chris ls /srv/shop/src
sudo -u bella touch /srv/shop/uploads/photo.jpg
sudo -u alex rm /srv/shop/uploads/photo.jpg
```

![Тесты доступа](screenshots/08-shop-access.png)

Результат: bella не прочитала `config.env` — `Permission denied`. chris прочитал `release-1` и не открыл `src`. `rm` файла bella в `uploads` спросил подтверждение, потому что каталог со sticky не даёт удалить чужой файл.

Какой бит или право объясняет каждый результат.

Вывод: дописать `app.py` позволили `umask 002` (файл `664`, группа `shop-dev`) и членство bella в `shop-dev`. `config.env` закрыт числом `600`. chris читает сборку, потому что у `releases` для остальных стоит `r-x`, и не входит в `src`. alex не удалил `photo.jpg` из-за sticky.

## Проверка

Команды:

```bash
sudo bash ~/Downloads/perms-lab-check.sh
```

![Проверка](screenshots/09-shop-check.png)

Результат: `Итог: 14 из 14`.

Итог проверки; какие пункты потребовали исправления.

Вывод: проверка прошла целиком. Файл `app.py` сразу создан с `umask 002`, поэтому группа смогла дописать вторую строку.

## Перед сдачей

- [+] Проверка: 14 из 14.
- [+] Под каждым снимком есть вывод.
- [ ] Учебные каталоги и учётные записи удалены.

Таблица прав своего проекта — в `permissions.md`. Скрипт каталогов shop — `setup-shop.sh`.
