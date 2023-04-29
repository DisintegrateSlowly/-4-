# Кейс ГринАтом

- [Подготовка](#подготовка)
  - [Установка виртуальных машин](#установка-виртуальных-машин)
    - [Установка master ВМ (ВМ01)](#установка-master-вм-вм01)
    - [Установка minion ВМ (ВМ02)](#установка-minion-вм-вм02)
  - [Установка Slatstack](#установка-slatstack)
    - [Добавление Salt репозитория](#добавление-salt-репозитория)
    - [Установка salt-master](#установка-salt-master)
    - [Установка salt-minion](#установка-salt-minion)
  - [Настройка Slatstack](#настройка-slatstack)
    - [Настройка `master`](#настройка-master)
    - [Настройка `minion`](#настройка-minion)
    - [Обмен ключами](#обмен-ключами)
- [Устанавливаем nginx через Salt](#устанавливаем-nginx-через-salt)
  - [Подготавливаем пакет](#подготавливаем-пакет)
  - [Инициализируем Salt State файлы (`.sls`)](#инициализируем-salt-state-файлы-sls)

## Подготовка

### Установка виртуальных машин

Для того чтобы у нас у каждой VM был свой ip-адрес в NAT-сети, мы должны создать новую в настройках сети в VirtualBox (нажать на кнопку `Create`)

По итогу создаётся сеть `10.x.x.x/24` с включённым DHCP ():

![Screenshot from 2023-04-29 20-17-04](https://user-images.githubusercontent.com/40892927/235315377-28711d75-e88f-4c92-9a86-71b3af1cbaf4.png)

#### Установка master ВМ (ВМ01)

1. Создаём VM
2. Указываем в настройках VM (`VM -> Settings -> Network -> Adapter`) `NAT Network` и выбираем из списка нашу сеть (`NatNetwork`)
![Screenshot from 2023-04-29 20-46-48](https://user-images.githubusercontent.com/40892927/235316782-91b80982-8441-40e7-b129-7e10dfe11f47.png)
3. Устанавливаем Astra Linux - просто проходим все шаги установки
3.1. При установке указываем hostname (название сервера) `astra-master`

#### Установка minion ВМ (ВМ02)

1. Клонируем `astra-master` VM и называем новую VM как `astra-minion`
2. Меняем hostname черзез команду `hostnamectl set-hostname astra-minion`
3. Перезагружаемся

### Установка Slatstack

#### Добавление Salt репозитория

Для установки из репозитория Salt нужно ввести следующае команды (для добавления репозитория) на всех машинах:

```sh
# Создаём директорию для GPG ключей (для проверки подписей пакетов из APT репозиториев)
sudo mkdir /etc/apt/keyrings

# Скачиваем ключ для Salt репозитория
sudo curl -fsSL -o /etc/apt/keyrings/salt-archive-keyring.gpg https://repo.saltproject.io/salt/py3/debian/11/amd64/SALT-PROJECT-GPG-PUBKEY-2023.gpg && \
# Добавлляем запись о Salt репозитории отдельным list файлом 
echo "deb [signed-by=/etc/apt/keyrings/salt-archive-keyring.gpg arch=amd64] https://repo.saltproject.io/salt/py3/debian/11/amd64/3006 bullseye main" | sudo tee /etc/apt/sources.list.d/salt.list

# Обновляем сведения о репозиториях и пакетах (их список, версии и т.д.) в них 
sudo apt-get update
```

#### Установка salt-master

Устанавливаем `salt-master` на BM01 (мы назвали её `astra-master`)

```sh
# Обновляем сведения о репозиториях и пакетах (их список, версии и т.д.) в них 
sudo apt-get update
# Устанавливаем salt-master (параметр -y нужен для установки без подтверждения)
sudo apt-get install -y salt-master
```

Далее включаем (чтобы он стартовал всегда при запуске) сервис salt-master и запускаем его:

```sh
sudo systemctl enable salt-master && sudo systemctl start salt-master
```

Проверяем, что сервис запущен (`systemctl status salt-master`):

![Screenshot from 2023-04-29 19-25-52](https://user-images.githubusercontent.com/40892927/235314352-706d0c0a-3f8e-4bdc-b49f-3d2d9bf19bf6.png)

#### Установка salt-minion

Установка подобна тому, что было в [предыдущем разделе](#установка-salt-master), только меняется название пакета с `salt-master` на `salt-minion`

В итоге надо ввести такие команды на BM02 (`astra-minion`):

```sh
sudo apt-get update
sudo apt-get install -y salt-minion
sudo systemctl enable salt-minion && sudo systemctl start salt-minion
```

![Screenshot from 2023-04-29 21-14-46](https://user-images.githubusercontent.com/40892927/235318012-3220e55f-c029-4c9d-88f9-b8da989c5f2f.png)

> При установка подразумется, что репозитории добавлены, как описано в разделе [`Добавление Salt репозитория`](#добавление-salt-репозитория)

### Настройка Slatstack

#### Настройка `master`

На `astra-master` создаём сетевой конфиг для salt-master (`/etc/salt/master.d/network.conf`):

> Мы указали адрес, который выдал нам DHCP-сервер в сети [`NatNetwork`](#установка-master-вм-вм01)

```yaml
# Указываем ip-адрес интерфейса к которому "забиндится" master
interface: 10.0.2.15
```

> Если сервис `salt-master` уже запущен (как и в нашем случае), то для применения изменений надо выполнить команду: \
> `sudo systemctl restart salt-master`

#### Настройка `minion`

На `astra-minion` создаём конфиг для подключения к `master` (`/etc/salt/minion.d/master.conf`):

```yaml
# Указываем ip-адрес master'а
master: 10.0.2.15
```

И создаём конфиг для идентификации нашей машины (`/etc/salt/minion.d/id.conf`):

```yaml
id: astra_1
```

> Если сервис `salt-minion` уже запущен (как и в нашем случае), то для применения изменений надо выполнить команду: \
> `sudo systemctl restart salt-minion`

#### Обмен ключами

После установки и настройки `master` и `minion`, для безопасности, нужно подтвердить, что подключаемый `minion` наш. Это можно сделать через утилиту `salt-key`

На `master` вводим команду `salt-key`:

```shell
user@astra-master:~$ sudo salt-key 
Accepted Keys:
Denied Keys:
Unaccepted Keys:
astra_1
Rejected Keys:
```

Видим, что ключ нашего `minion` неподтверждённый. Чтобы подтвердить вводим команду `salt-key -a astra_1`:

![Screenshot from 2023-04-29 21-41-50](https://user-images.githubusercontent.com/40892927/235319198-383faf07-6265-4446-bf12-2edf4a162371.png)

Проверяем, что установка и конфигурация прошла успешно и `master` видит наш `minion` (`astra_1`):

![Screenshot from 2023-04-29 21-44-09](https://user-images.githubusercontent.com/40892927/235319191-e689c734-3c9e-4f58-8600-86e4a0e25d9d.png)

## Устанавливаем nginx через Salt

### Подготавливаем пакет

Скачаем предварительно `deb-пакет` `nginx` с [официального сайта](http://nginx.org/packages/debian/pool/nginx/n/nginx/) (для Debian `bullseye`) на `astra-master`

```sh
# Предварительно создаём папку, где будем хранить nginx
sudo mkdir -p /srv/salt/nginx

sudo curl -fsSL -o /srv/salt/nginx/nginx.deb http://nginx.org/packages/debian/pool/nginx/n/nginx/nginx_1.24.0-1~bullseye_amd64.deb
```

### Инициализируем Salt State файлы (`.sls`)

Теперь нам надо написать "инструкцию" как установить наш `nginx` пакет

Напишем `state` для передачи файла на наш `minion` (`/srv/salt/nginx/init.sls`):

```yaml
# Путь сохранения
/tmp/nginx.deb:
  file.managed:
    # Путь на master относительно "root folder" (/srv/salt)
    - source: salt://nginx/nginx.deb
    # Права доступа к файлу
    - user: root
    - group: root
    - mode: 644
```

После создаём конфигурацию запуска наших "states" (пока одного) для определённых `minions` (`/srv/salt/top.sls`):

```yaml
base:
  # Применяем для всех minion'ов с названием astra
  'astra*':
    - nginx
```

Теперь проверим, что всё работает. Запустим наш `state` командой `sudo salt astra_1 state.apply`:

![Screenshot from 2023-04-29 23-08-55](https://user-images.githubusercontent.com/40892927/235322347-a0077203-7c7f-4c7a-99ef-bc2188514500.png)

И мы можем убедиться, что файл был передан на наш `minion`:

![Screenshot from 2023-04-29 23-09-49](https://user-images.githubusercontent.com/40892927/235322354-a37a7004-239a-4cf4-adf9-1a6ed924d660.png)
