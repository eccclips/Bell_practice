# practice

## Двойной буфер для ВМ

Двойной буфер намного упрощает жизнь при использовании VirtualBox, позволяя копипастить из основной системы и наоборот.

Сетап двойного буфера (Ubuntu 24.04)
1.  `Устройства` -> `Подключить образ Дополнений гостевой ОС`

    На экране появится иконка установочного образа для дополнений в виде диска. 

2.  Снова переходим в `Устройства`, во вкладках `Общий буфер` обмена выбираем `Двунаправленный` и `Разрешить передачу файлов`, во вкладке `Функция Drag and Drop` также выбираем `Двунаправленный`.
3.  Переходим в терминал и вводим следующие команды

```bash
sudo apt-get install dkms build-essential linux-headers-$(uname –r)
sudo apt-get install bzip2
sudo apt-get update
```

4.  Запускаем установку образа и жмем `Install Now`, по завершении загрузки делаем контрольный перезапуск системы и получаем удовольствие.

---


## SSH 
В `настройках сети` ВМ в `virtual box` нужно выбрать `сетевой мост`! и перезагрузить ВМ

1. установить Open SSH: 

```bash
sudo apt install openssh-server openssh-clients
```
2. На виртуальной машине ubuntu и в WSL создать пары ssh ключей:

```bash
ssh-keygen
```
3. На виртуальной машине ubuntu зайти в ssh config:

```bash
sudo nano /etc/ssh/sshd_config
```
4. в этом файле раскомментировать и изменить слудующее (это создание юзера root):

- ```PermitRootLogin yes```
- ```PubkeyAuthentication yes```
- ```AuthorizedKeysFile```
- ```KbdInteractiveAuthentication no```
5. зайти под юзером root и установить для него пароль (passwd вводится уже под root'ом). АНАЛОГ ```sudo passwd```:

```bash
sudo su
passwd
```
6. отправить из WSL ssh pubkey на ВМ:

```bash
ssh-copy-id {username(root)}@{server-ip}

ssh-copy-id -i /home/p0mi/.ssh/key.pem.pub admin@192.168.1.22
```
7. проверить подключение. В WSL написать (```exit``` чтобы отключиться):

```bash
ssh root@{server-ip}
```

---

## Kafka without docker 3.9.0

1. Установка openjdk 21

```bash
sudo apt install -y openjdk-21-jdk
```

2. Скачивание Kafka 3.9.0 и распаковка архива в opt/kafka

```bash
wget https://dlcdn.apache.org/kafka/3.9.0/kafka_2.13-3.9.0.tgz
```
```bash
tar -xzf kafka_2.13-3.9.0.tgz
```
```bash
sudo mv kafka_2.13-3.9.0 /opt/kafka
```

3. Создание и установка переменной среды (в официальной документации этого нету и вроде не обязательно)

```bash
export KAFKA_HOME=/opt/kafka

export PATH=$PATH:$KAFKA_HOME/bin

source ~/.bashrc
```

4. Редактирование файла `server.properties` для `kafka` в режиме `kraft`

```bash
sudo nano /opt/kafka/config/kraft/server.properties
```

что надо отредактировать?

- { key: 'advertised.listeners', value: 'PLAINTEXT://{{ new_ip }}:9092,CONTROLLER://localhost:9093' }
- { key: 'listeners', value: 'PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093' }
- { key: 'log.dirs', value: '/opt/kafka/kafka_2.13-3.9.0/logs/kraft-combined-logs'}

5. Генерация uuid ключа для кластера и установка этого ключа для kafka

```bash
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
```
```bash
bin/kafka-storage.sh format --standalone -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties
```
**Чтобы kafka работала в кластерном режиме uuid должен быть одинаковым!!**

6. Запуск
   
```bash
bin/kafka-server-start.sh config/kraft/server.properties
```
Чтобы не использовать эту команду можно вручную создать сервис и запускать его. для этого нужно:

6.1. Создать по пути `/etc/systemd/system/` файл с расширением *.service*, например `kafka.service` и добавить следующий шаблонную запись:

```
[Unit]
Description=Apache Kafka
After=network.target

[Service]
User=root
Group=root
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64"
ExecStart=/bin/bash -c '/opt/kafka/kafka_2.13-3.9.0/bin/kafka-server-start.sh /opt/kafka/kafka_2.13-3.9.0/config/kraft/server.properties'
ExecStop=/opt/kafka/kafka_2.13-3.9.0/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

7. Базовые команды для взаимодействия с kafka через терминал список всех команд находится в папке `/opt/kafka/bin`:

```
./kafka-topics.sh --bootstrap-server localhost:9092 --create --topic test --partitions 1
```

```
./kafka-topics.sh --bootstrap-server localhost:9092 --list
```

```
./kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic test
```

```
./kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic test
```

```
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```

```
./kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test
```
```
/opt/kafka/kafka_2.13-3.9.0/bin/kafka-metadata-quorum.sh --bootstrap-controller 192.168.1.21:9093 describe --status
```

## Kafka KRaft Cluster без Docker (версия 3.9.0) 

Ansible-плейбук для автоматизированного развёртывания Kafka-кластера (3 узла) в режиме KRaft без использования Docker.  
Подходит для production и обучения.

---

### 🔧 Что делает playbook (по блокам)

#### - name: Jdk install (Debian)
- Устанавливает OpenJDK 21 через apt.

#### - name: Jdk install (RHEL)
- Устанавливает OpenJDK 21 через dnf.

#### - name: Create kafka directory
- Создаёт рабочую директорию /opt/kafka.

#### - name: Download kafka archive
- Скачивает архив Kafka 3.9.0 (Scala 2.13) с официального зеркала.

#### - name: Unpack kafka archive
- Распаковывает архив в /opt/kafka/kafka_2.13-3.9.0.

#### - name: Create systemd kafka.service
- Копирует unit-файл kafka.service в /etc/systemd/system/.

#### - name: Reload systemd daemon
- Перезагружает демон systemd, чтобы подхватить новый сервис.

#### - name: Generate server.properties
- Создаёт или модифицирует файл конфигурации Kafka:
  - process.roles=broker,controller
  - controller.quorum.voters — на основе инвентаря
  - Уникальный broker.id — из переменной BROKER_ID
  - Пути к логам, директориям, портам и другим параметрам

#### - name: Create log dirs
- Создаёт директории логов Kafka, если их нет.

#### - name: Generate cluster.id
- Генерирует UUID (один раз) и сохраняет его в файл cluster.id на контроллере.

#### - name: Sync cluster.id to all nodes
- Распространяет сгенерированный cluster.id на все узлы кластера.

#### - name: Format kafka storage
- Выполняет команду kafka-storage.sh format с заданным cluster.id, если Kafka ещё не инициализирована.

#### - name: Enable kafka systemd service
- Включает автозапуск Kafka через systemctl enable kafka.

#### - name: Start kafka
- Запускает Kafka как systemd-сервис.

#### - name: Restart kafka on config change
- Обеспечивает перезапуск сервиса при изменении конфигурации.

---

## Сборка фронта в докере на node:18.18.0

1. Создаем контейнер с образом node:18.18.0

```
sudo docker run -itd --name node -p 8000:8000 node:18.18.0
```

2. Заходим внутрь контейнера и копи
    