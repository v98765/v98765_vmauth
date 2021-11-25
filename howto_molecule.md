## установка

* venv
* ansible4
* molecule + docker driver
* плагины для vim

Версии ansible меняются часто,
поэтому вместо обновления будет проще создать новое окружение у пользователя или пользователя с новым окружением.
Нужно примерно несколько гигабайт свободного места на диске для софта и тестов.

По [документации](https://molecule.readthedocs.io/en/latest/examples.html) на молекулу у пользователя должны быть определенные привилегии.
В противном случае нужно будет запускать молекулу, которая стартует докер, от рута. Тут будет с sudo запуск молекулы. non-root не знаю.

```sh
groupadd -r deploy
usermod -aG deploy ansible4
echo '%deploy  ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/deploy
```
Переподключиться пользователем, выполнить `sudo whoami`. Ответит `root` без пароля.

Создать и активировать окружение а4. После активации в промпте будет отображаться `(a4)`
```sh
python3 -m venv a4
source a4/bin/activate
echo 'source a4/bin/activate' >> ~/.bashrc
```
Обновить pip
```sh
a4/bin/python3 -m pip install --upgrade pip
```
Установить необходимое для работы. бинарники докера установлены ранее из пакетов.
```text
pip install wheel
pip install scp ansible ansible-lint yamllint molecule molecule-docker
```
Проверка `ansible --version`
```text
ansible [core 2.11.6]
  config file = None
  configured module search path = ['/home/myname/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/myname/a4/lib/python3.8/site-packages/ansible
  ansible collection location = /home/myname/.ansible/collections:/usr/share/ansible/collections
  executable location = /home/myname/a4/bin/ansible
  python version = 3.8.10 (default, Jun  2 2021, 10:49:15) [GCC 10.3.0]
  jinja version = 3.0.2
  libyaml = True
```
Плагины vim
```sh
git clone --depth 1 https://github.com/dense-analysis/ale.git ~/.vim/pack/git-plugins/start/ale
git clone https://github.com/Yggdroot/indentLine.git ~/.vim/pack/vendor/start/indentLine
```
Добавить в `.config/yamllint/config` настройки для yamllint, чтоб не проверял длину строк
```text
extends: relaxed

rules:
  line-length: disable
```
Настроить vim для редактирования файлов yml, yaml
```text
set mouse-=a
syntax on
autocmd FileType yaml setlocal ts=2 sts=2 sw=2 expandtab
autocmd FileType yml setlocal ts=2 sts=2 sw=2 expandtab
let g:ale_echo_msg_format = '[%linter%] %s [%severity%]'
let g:ale_sign_error = 'x'
let g:ale_sign_warning = '!'
let g:ale_lint_on_text_changed = 'never'
```
Каталог для ролей
```sh
mkdir -p .ansible/roles
cd .ansible/roles
```

## init role

```sh
molecule init role v98765_vmauth --driver-name=docker
```
Результат
```text
INFO     Initializing new role v98765_vmauth...
No config file found; using defaults
- Role v98765_vmauth was created successfully
INFO     Initialized role in /home/ansible4/.ansible/roles/v98765_vmauth successfully.
```
tree
```text
.
└── v98765_vmauth
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── molecule
    │   └── default
    │       ├── converge.yml
    │       ├── molecule.yml
    │       └── verify.yml
    ├── README.md
    ├── tasks
    │   └── main.yml
    ├── templates
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml

11 directories, 11 files
```
Т.к. роль не предполагается размещать в galaxy, а `molecule lint`
выдаст ошибку из-за дефолтового `meta/main.yml`, то удалить этот каталог сразу.
```sh
cd v98765_vmauth
rm -rf meta
molecule lint
```
При первом успешном запуске будут установлены недостающие зависимости
```text
INFO     default scenario test matrix: dependency, lint
INFO     Performing prerun...
INFO     Added ANSIBLE_LIBRARY=/home/myname/.cache/ansible-compat/26e08d/modules:/home/myname/.ansible/plugins/modules:/usr/share/ansible/plugins/modules
INFO     Added ANSIBLE_COLLECTIONS_PATH=/home/myname/.cache/ansible-compat/26e08d/collections:/home/myname/.ansible/collections:/usr/share/ansible/collections
INFO     Added ANSIBLE_ROLES_PATH=/home/myname/.cache/ansible-compat/26e08d/roles:/home/myname/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles
INFO     Running default > dependency
INFO     Running ansible-galaxy collection install -v community.docker:>=1.8.0
WARNING  Skipping, missing the requirements file.
WARNING  Skipping, missing the requirements file.
INFO     Running default > lint
COMMAND: set -e
yamllint .
ansible-lint .

Loading custom .yamllint config file, this extends our internal yamllint config.
``` 
Скопировать/заменить содержимое в [molecule/default/molecule.yml](https://github.com/v98765/v98765_vmauth/blob/main/molecule/default/molecule.yml).
Для теста нужны права и рабочее окружение. Пока так. Если все корректно, то тест пройдет.
```sh
sudo -- bash -c 'source /home/ansible4/a4/bin/activate ; molecule test'
```

## handlers

Типичный для рестарта и релоада [handlers/main.yml](https://github.com/v98765/v98765_vmauth/blob/main/handlers/main.yml)

## tasks

Привык к такой структуре:

* [tasks/main.yml](https://github.com/v98765/v98765_vmauth/blob/main/tasks/main.yml) где включаются отдельно `preinstall.yml`, `install.yml`, `configure.yml` с тегами.
* [tasks/preinstall.yml](https://github.com/v98765/v98765_vmauth/blob/main/tasks/preinstall.yml) где создается учетная запись.
* [tasks/install.yml](https://github.com/v98765/v98765_vmauth/blob/main/tasks/install.yml) где создаются рабочие каталоги и разархивируется бинарник на хост.
* [tasks/configure.yml](https://github.com/v98765/v98765_vmauth/blob/main/tasks/configure.yml) создание конфигурационных и прочих файлов, на которые может повлиять изменение переменных.

## templates

Шаблон для старта сервиса [templates/vmauth.service.j2](https://github.com/v98765/v98765_vmauth/blob/main/templates/vmauth.service.j2).
Если в какой-то каталог понадобиться сервису писать,
то нужны права на запись и в параметрах `[Service]` шаблона для systemd указать `ReadWriteDirectories={{ service_some_dir }}`.
В зависимости от кол-ва файлов/процессов изменить/добавить `LimitNOFILE=`.
Все сервисы victorimetrics умеют работать с переменными, поэтому в общем случае нет необходимости каждый раз править `*.service`-файл.

Шаблон для переменных [environment-variables](https://docs.victoriametrics.com/#environment-variables)
в [templates/vmauth.j2](https://github.com/v98765/v98765_vmauth/blob/main/templates/vmauth.j2).

Шаблон для конфигурационного файла [templates/vmauth.yml.j2](https://github.com/v98765/v98765_vmauth/blob/main/templates/vmauth.yml.j2).

## vars

Используемые в tasks, templates переменные описываются в [defaults/main.yml](https://github.com/v98765/v98765_vmauth/blob/main/defaults/main.yml).

## дистрибутив vmutils

```sh
mkdir -p /var/tmp/archive
cd /var/tmp/archive
wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.69.0/vmutils-amd64-v1.69.0.tar.gz
```

## Тест

из каталога `~/.ansible/roles/v98765_vmauth` 
```sh
sudo -- bash -c 'source /home/ansible4/a4/bin/activate ; molecule converge'
```
Исправить ошибки, если есть. Если нет, то залогиниться, проверить
```sh
sudo -- bash -c 'source /home/ansible4/a4/bin/activate ; molecule login'
```
Вывод для примера
```text
INFO     Running default > login
[root@centos8 /]# ps ax | grep vma
   3259 ?        Ssl    0:00 /usr/local/bin/vmauth -envflag.enable -envflag.prefix=VMAUTH_
   3291 pts/0    S+     0:00 grep --color=auto vma
[root@centos8 /]# 
```
Поделать команды, посмотреть логи
```sh
journalctl -e -u vmauth
curl -s -u 'fullaccess:demo' -g 'http://localhost:8427/metrics'
curl -s -u 'readonly:demo' -g 'http://localhost:8427/-/reload'
remocurl -s -u 'readonly:demo' -g 'http://localhost:8427/-/reload?authKey=RkU7FhjinrCv7N7f'
```
Править роль и повторять `molecule converge` до получения нужного результата.
После чего `molecule destroy` и тест в т.ч. на идемпотентность `molecule test`.
