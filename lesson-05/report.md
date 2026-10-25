# Процессы: PID, состояния и сигналы · отчёт

ФИО: Булаева Ева Игоревна
Группа: 9/2-РПО-25/1
Дата: 25.10.2026
Среда: Ubuntu 26.04.1 LTS, VirtualBox, пользователь `evkis`

## Программа и процесс

Команды:

```bash
ls -l /usr/bin/sleep
sleep 300 &
sleep 301 &
pgrep -a sleep
ps -p $! -o pid,ppid,user,stat,etime,cmd
head -9 /proc/$!/status
readlink /proc/$!/exe
```

![Программа и процесс](screenshots/01-process.png)

Результат: `/usr/bin/sleep` — символьная ссылка на `../lib/cargo/bin/coreutils/sleep`. Запущены два процесса: PID `20220` (`sleep 300`) и PID `20221` (`sleep 301`). У второго PPID `20217`, пользователь `evkis`, состояние `S (sleeping)`. `readlink /proc/20221/exe` дал `/usr/lib/cargo/bin/coreutils/sleep`.

Подпишите PID, PPID и состояние; чем процесс отличается от файла программы.

Вывод: файл на диске — это программа, его никто не выполняет, пока не запустят. Процесс — уже запущенный экземпляр. У него свой PID `20221`, родитель с PID `20217` и состояние `S`: он спит до конца таймера. Два запуска `sleep` получили два разных PID, хотя файл программы один.

## Дерево процессов

Команды:

```bash
echo $$
bash -c 'sleep 200; echo done' &
pstree -s -p $!
ps -p 1 -o pid,ppid,user,cmd
```

![Дерево процессов](screenshots/02-tree.png)

Результат: `$$` внутри скрипта был `21056`. `pstree` вывел

```text
systemd(1)---systemd(16122)---ptyxis(20961)---ptyxis-agent(20977)---bash(21019)---bash(21056)---bash(21058)---sleep(21060)
```

PID 1 — `/usr/lib/systemd/systemd`, PPID `0`, пользователь `root`.

Подпишите цепочку от оболочки до sleep: кто чей родитель.

Вывод: `sleep` `21060` запустил `bash -c` `21058`. Его родитель — скрипт `21056`, а того запустила оболочка терминала `21019`. Дальше вверх: агент Ptyxis `20977`, окно Ptyxis `20961`, пользовательский systemd `16122` и systemd с PID 1. Если родитель завершается, потомка забирает процесс выше по дереву, в конце концов PID 1.

## Состояния

Команды:

```bash
yes > /dev/null &
sleep 500 &
bash -c 'sleep 1 & exec sleep 300' &
sleep 2
kill -STOP $S
ps -o pid,ppid,stat,cmd -p ...
ps -o pid,ppid,stat,cmd --ppid ...
```

![Состояния](screenshots/03-states.png)

Результат: `yes` PID `21068` — `R+`, `sleep 500` PID `21069` после STOP — `T+`, родительский `sleep 300` PID `21070` — `S+`, его потомок PID `21072` — `Z+` `[sleep] <defunct>`. Сводка по первой букве STAT: были и `R`, и `S`, и `T`, и `Z`.

Объясните каждое из четырёх состояний и почему появился зомби.

Вывод: `R` — процесс считается выполняющимся, `yes` крутится без остановки. `S` — спит, как `sleep`, пока не кончится время. `T` — остановлен сигналом STOP, планировщик его не берёт. `Z` — процесс уже завершился, но родитель не вызвал `wait`, поэтому запись с кодом завершения осталась. Здесь родитель сделал `sleep 1 &` и сразу заменил себя на долгий `sleep`, так что короткий потомок стал зомби.

## Фон и передний план

Команды:

```bash
sleep 1000
jobs -l
bg %1
jobs -l
fg %1
```

![Фон и передний план](screenshots/04-jobs.png)

Результат: `sleep 1000` занял терминал. Ctrl+Z остановил задание `[1]+ 21090`. `jobs -l` показал `Stopped`. `bg %1` продолжил его в фоне, в `jobs` появилось `Running` и знак `&`. `fg %1` снова поставило его на передний план, следующий Ctrl+Z опять показал `Stopped`.

Что делает с заданием каждое действие.

Вывод: без `&` задание на переднем плане, оболочка ждёт его и не читает новые команды. Ctrl+Z посылает STOP и возвращает приглашение. `bg` шлёт CONT и оставляет задание в фоне. `fg` возвращает его на передний план, оболочка снова ждёт. Номер задания в этой сессии был `%1`, PID `21090`.

## STOP, CONT и trap

Команды:

```bash
kill -l | head -3
sleep 900 &
kill $!; wait $!; echo $?
sleep 901 &
kill -9 $!; wait $!; echo $?
yes > /dev/null &
kill -STOP $!
kill -CONT $!
bash -c 'trap "echo got TERM, keep working" TERM; while true; do sleep 1; done' &
kill $!
kill -9 $!
```

![STOP, CONT и trap](screenshots/05-signals.png)

Результат: в списке сигнал 2 — SIGINT, 9 — SIGKILL, 15 — SIGTERM. `wait` после TERM дал `143`, после KILL — `137`. `yes` PID `20261` был `R`, после STOP стал `T`, после CONT снова `R`. Процесс с `trap` PID `20265` написал `got TERM, keep working` и остался в состоянии `S`. `kill -9` его снял, повторный `ps` уже пустой.

Почему процесс пережил TERM и не пережил KILL.

Вывод: TERM — обычный сигнал, программа может поставить на него `trap` и не завершаться. KILL процессорная ловушка не получает: ядро убирает процесс само. STOP и CONT при этом не завершают процесс, а только ставят на паузу и снимают с неё.

## Коды завершения

Команды:

```bash
true; echo $?
false; echo $?
ls /nope; echo $?
sleep 100 &
kill -INT $!; wait $!; echo $?
sleep 100 &
kill $!; wait $!; echo $?
sleep 100 &
kill -9 $!; wait $!; echo $?
```

![Коды завершения](screenshots/06-exit.png)

Результат: `true` — `0`, `false` — `1`, `ls /nope` — `2`. `kill -INT` у `sleep 100` дал `130`. `kill` дал `143`. `kill -9` дал `137`, в строке задания было `Killed`.

Объясните коды 130, 143 и 137.

Вывод: обычный успех — 0, своя ошибка программы — небольшое число вроде 1 или 2. Если процесс снят сигналом, bash показывает `128 + номер сигнала`. 130 = 128 + 2, это SIGINT, тот же сигнал, что шлёт Ctrl+C. 143 = 128 + 15, это SIGTERM. 137 = 128 + 9, это SIGKILL.

## Набор процессов

Команды:

```bash
bash ~/Downloads/procs-lab-setup.sh
python3 ~/procs-lab/bin/manager.py &
sleep 2
pstree -p $M
ps -o pid,ppid,stat,%cpu,cmd --ppid $M
ps -o pid,ppid,stat,cmd --ppid $(pgrep -f lab-parent)
```

![Набор процессов](screenshots/07-lab-tree.png)

Результат: диспетчер `python3` PID `20760`. Потомки: worker `20762` `S`, hog `20763` `R` и 99.5% CPU, stubborn `20764` `S`, paused `20765` `T`, lab-parent `20767` `S`. Его потомок `20769` — `Z`.

Таблица «процесс — состояние» для набора.

Вывод: таблица с этими PID записана в `states.md`. Буквы совпали с ролями набора: спящие worker и stubborn, занятый hog, остановленный paused и зомби у lab-parent.

## Сигналы по заданиям

Команды:

```bash
kill -STOP hog; ps
kill -CONT hog; kill hog
kill -CONT paused; kill paused
kill stubborn; cat ~/procs-lab/stubborn.log
kill -9 stubborn
kill -9 zombie; ps
kill lab-parent
kill worker
cat ~/procs-lab/exit-codes.log
pgrep -af procs-lab
```

![Сигналы по заданиям](screenshots/08-lab-signals.png)

Результат: hog `20763` после STOP был `T`, в журнале `hog 20763 signal 15`. stubborn записал `TERM ignored at 18:23:39` и затем `stubborn 20764 signal 9`. Зомби `20769` после `kill -9` остался `Z` и исчез вместе с lab-parent (`signal 15`). paused `20765` и worker `20762` тоже `signal 15`. `pgrep` вывел `NONE`, диспетчер сам дописал журнал и вышел.

Какой сигнал выбран для каждого процесса и почему.

Вывод: hog, paused и worker завершены TERM, сигнал 15: они его не перехватывают, KILL не нужен. stubborn от TERM отказался, поэтому снят KILL, сигнал 9. Зомби сигналом не убирается, его снимает завершение родителя. Диспетчер дождался всех потомков и вышел сам.

## Проверка

Команды:

```bash
bash ~/Downloads/procs-lab-check.sh
```

![Проверка](screenshots/09-lab-check.png)

Результат: `Итог: 11 из 11`.

Итог проверки; какие пункты потребовали исправления

Вывод: набор, журнал и сигналы сошлись сразу: hog записан как signal 15, stubborn как signal 9, процессов набора не осталось. Первый запуск проверки показал 10 из 11 из-за чужого зомби `[xdg-terminal-ex]` у окна терминала, не у lab-parent. В новом окне этого зомби не было, и тот же скрипт по тем же журналам вывел 11 из 11. Исправлять сигналы набора не пришлось.

## Перед сдачей

- [+] Проверка: 11 из 11.
- [+] Под каждым снимком есть вывод.
- [+] states.md заполнен.
- [+] Для кодов 130, 143 и 137 указан сигнал.
- [+] Процессов набора не осталось.

Скрипт `graceful.sh`, коды двух его запусков и цепочка от PID 1 — в `processes.md`.
