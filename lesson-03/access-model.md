# Модель доступа проекта study-bot

Проект: бот для конспектов курса. Код бота и заметки лежат отдельно, чтобы автор заметок не менял программу, а разработчик не правил чужие конспекты.

Скрипт `setup-team.sh` запущен в виртуальной машине. После снятия вывода учебные учётные записи удалены.

## Роли

| Роль | Учётная запись | Группы | Может | Не может |
|---|---|---|---|---|
| руководитель | anna | bot-dev, bot-qa, sudo | код, заметки и администрирование | — |
| разработчик | boris | bot-dev | писать код в `app` | открывать `notes`, вызывать sudo |
| редактор конспектов | vera | bot-qa | писать заметки в `notes` | открывать `app`, вызывать sudo |
| наблюдатель | oleg | только личная группа oleg | свой домашний каталог | `app`, `notes`, sudo |

## Каталоги

| Каталог | Группа | Права | Кто работает |
|---|---|---|---|
| `/srv/study-bot/app` | bot-dev | `2770` · `drwxrws---` | anna, boris |
| `/srv/study-bot/notes` | bot-qa | `2770` · `drwxrws---` | anna, vera |

## Вывод id

```text
uid=1006(anna) gid=1010(anna) groups=1010(anna),27(sudo),1008(bot-dev),1009(bot-qa)
uid=1007(boris) gid=1011(boris) groups=1011(boris),1008(bot-dev)
uid=1008(vera) gid=1012(vera) groups=1012(vera),1009(bot-qa)
uid=1009(oleg) gid=1013(oleg) groups=1013(oleg)
```

## Вывод ls -l /srv/study-bot

```text
drwxrws--- 2 root bot-dev 4096 Oct 25 18:04 app
drwxrws--- 2 root bot-qa  4096 Oct 25 18:04 notes
```

![Запуск setup-team.sh](screenshots/10-setup.png)

## Пять тестов

| № | Кто | Команда | Ожидается | Почему |
|---|---|---|---|---|
| 1 | anna | создать файл в `app` | да | anna в `bot-dev`, у каталога `2770` |
| 2 | boris | создать файл в `app` | да | boris в `bot-dev` |
| 3 | boris | открыть `notes` | нет | boris не в `bot-qa`, для остальных `---` |
| 4 | vera | дописать файл в `notes` | да | vera в `bot-qa` |
| 5 | oleg | открыть `app` | нет | oleg не входит в проектные группы |
