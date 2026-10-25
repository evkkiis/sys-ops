# Процессы, сигналы и коды завершения

## Цепочка от PID 1 до оболочки

Команда `pstree -s -p $$` в терминале Ptyxis:

```text
systemd(1)---systemd(16122)---ptyxis(20961)---ptyxis-agent(20977)---bash(21019)---bash(21139)---pstree(21141)
```

| Звено | Что это |
|---|---|
| systemd (1) | первый процесс системы, его родитель — ядро, PPID 0 |
| systemd (16122) | менеджер сессии пользователя |
| ptyxis (20961) | окно терминала. В Ubuntu 26.04 это Ptyxis, не GNOME Terminal |
| ptyxis-agent (20977) | помощник терминала, от него запускается оболочка |
| bash (21019) | текущая оболочка |
| bash (21139) и pstree (21141) | сама команда, которой снята цепочка, а не постоянные родители |

![Цепочка процессов](screenshots/12-chain.png)

## Опыт 1 — graceful.sh и SIGTERM

```bash
~/lesson_05/graceful.sh &
G=$!
kill -TERM $G
wait $G
echo $?
```

Скрипт создал `/tmp/graceful-21106.tmp`. После TERM он написал `TERM получен, удаляю /tmp/graceful-21106.tmp`, `wait` вернул `0`, повторный `ls` ответил `No such file or directory`.

![TERM](screenshots/10-term.png)

## Опыт 2 — graceful.sh и SIGKILL

```bash
~/lesson_05/graceful.sh &
G=$!
kill -9 $G
wait $G
echo $?
```

Код завершения `137`. Файл `/tmp/graceful-21120.tmp` остался, права `-rw-rw-r--`, владелец `evkis`, размер 35 байт. После снимка файл удалён.

![KILL](screenshots/11-kill.png)

## Коды завершения

| Код | Формула | Сигнал |
|---|---|---|
| 130 | 128 + 2 | SIGINT, тот же сигнал, что Ctrl+C |
| 143 | 128 + 15 | SIGTERM |
| 137 | 128 + 9 | SIGKILL |

Эти три кода получены отдельно: `kill -INT` у `sleep 100` дал 130, `kill` дал 143, `kill -9` дал 137. Снимок — в отчёте, `screenshots/06-exit.png`.

## Разница между TERM и KILL

TERM можно перехватить через `trap`. Скрипт успевает удалить временный файл и выйти с кодом 0. KILL ядро не отдаёт процессу: обработчик не запускается, файл остаётся, код равен `128 + 9`.
