# diploma-ansible

Плейбук установки Docker и Docker Compose на виртуальную машину дипломного проекта.

Часть итоговой работы курса Netology «DevOps-инженер». Общее описание проекта — в репозитории [diploma-devops](https://github.com/Maxim-Moskov/diploma-devops).

---

## Структура

```
.
├── ansible.cfg            # настройки подключения
├── inventory.ini.example  # шаблон инвентаря
└── playbook.yml           # установка Docker
```

Рабочий `inventory.ini` с реальным адресом машины в репозиторий не попадает — он отсекается `.gitignore`. Публичный IP меняется при каждом пересоздании ВМ, поэтому хранить его в коде смысла нет.

---

## Что делает плейбук

1. Ставит зависимости для работы с apt-репозиториями (`ca-certificates`, `curl`, `gnupg`, `apt-transport-https`)
2. Создаёт каталог `/etc/apt/keyrings` и загружает туда GPG-ключ Docker
3. Подключает официальный репозиторий Docker
4. Устанавливает `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`
5. Включает и запускает службу `docker`
6. Добавляет пользователя в группу `docker` — чтобы работать без `sudo`
7. Создаёт каталог `/opt/diploma-app` для файлов приложения
8. Выводит установленные версии

Каталог `/opt/diploma-app` создаётся с владельцем `ubuntu` не случайно: именно туда CI-пайплайн копирует `compose.yaml` по SSH, и без прав на запись деплой бы не прошёл.

---

## Требования

- Ansible core 2.14+
- Доступ по SSH к целевой машине с ключом без парольной фразы
- Целевая ОС: Ubuntu 22.04 LTS

---

## Запуск

```bash
# Подставить реальный IP виртуальной машины
sed 's/CHANGE_ME/<IP>/' inventory.ini.example > inventory.ini

# Проверить связь
ansible app -m ping

# Проверить синтаксис
ansible-playbook playbook.yml --syntax-check

# Применить
ansible-playbook playbook.yml
```

Ожидаемый результат:

```
TASK [Показать установленные версии]
ok: [diploma-app] =>
  msg:
  - Docker version 29.8.0, build 88096ef
  - Docker Compose version v5.5.1

PLAY RECAP
diploma-app : ok=13  changed=5  unreachable=0  failed=0
```

---

## Идемпотентность

Повторный запуск не вносит изменений:

```
PLAY RECAP
diploma-app : ok=13  changed=0  unreachable=0  failed=0
```

Плейбук можно прогонять сколько угодно раз — состояние системы не поменяется.

---

## Особенности реализации

**GPG-ключ загружается через `command`, а не `get_url`.** Модуль `get_url` в Ansible 2.14 несовместим с Python 3.12 и падает с `HTTPSConnection.__init__() got an unexpected keyword argument 'cert_file'`. Параметр `creates` сохраняет идемпотентность: если файл ключа уже есть, задача пропускается.

**Ставится `docker-compose-plugin`, а не `docker-compose`.** Первый — современная команда `docker compose` через пробел, второй — устаревший отдельный бинарник на Python. Задание допускает оба варианта.

**`host_key_checking = False` в `ansible.cfg`.** После `terraform destroy` и повторного `apply` у новой машины другой отпечаток SSH, и без этой настройки Ansible отказался бы подключаться.

**`pipelining = True`.** Ускоряет выполнение примерно вдвое за счёт сокращения числа SSH-операций.

---

## Известное ограничение: `--check` не работает

Пробный прогон завершается ошибкой:

```
TASK [Установить права на GPG-ключ]
fatal: [diploma-app]: FAILED! => msg: file (/etc/apt/keyrings/docker.gpg) is absent, cannot continue
```

Это не дефект плейбука. В режиме проверки Ansible не выполняет реальных действий, поэтому GPG-ключ физически не создаётся, и следующая задача не находит файл. Так ведёт себя любой плейбук, который ставит ПО с нуля: задачи зависят от результатов предыдущих.

Для предварительной проверки используется `--syntax-check`, который работает корректно.
