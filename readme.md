## Домашнее задание по уроку №30

В этом задании вам нужно будет добиться развертывания (deployment) своего сервиса. 
Для этого можно:

- Использовать площадки и дополнительное ПО, например: GitHub Action, GitLab CI/DI, Jenkins, etc.
- Использовать Docker и все возможные комбинации с ним.
- Написать свой небольшой скрипт для "разворачивания" 
(по сути просто загрузка бинарного файла на сервер) 

В этом задании задача будет сделать это с помощью скрипта. Другие площадки и пайплайны посмотрим в других уроках и в других домашках. 

Сервис, который необходимо развернуть - ваше домашнее задание с бекендом для блога. 

Результат домашнего задания - код сервера в этом репозитории, 
плюс все ваши конфиги сервиса на линукс, скрипт для деплоя и описание того, как этот деплой производить в readme файле

---------

Для подобного скрипта можно придумать много сетапов и реализаций. 

Предложу такой подход:
1) Запускаем на AWS виртуальный сервер, бесплатный тир. 
2) Подключаемся через ssh, проверяем, что все работает, 
что сервер видно из вне и он может отправлять запросы.
3) Создаем новый systemd service в Linux на сервере
   https://medium.com/@benmorel/creating-a-linux-service-with-systemd-611b5c8b91d6

Пример файла сервиса
```
# location of this file
# /lib/systemd/system/sneakers.service
[Unit]
Description=my backend on Golang
After=network.target

[Service]
Type=simple
WorkingDirectory=/ubuntu/myservice-directory/
EnvironmentFile=/ubuntu/myservice-directory/.env
ExecStart=/ubuntu/myservice-directory/myservice
Restart=on-failure
RestartSec=10


[Install]
WantedBy=multi-user.target
```

4) Пишем скрипт для загрузки бинарного билда вашего кода на сервер.
  Примерный план скрипта:  
- Билдим бинарь из го кода `go build -o myservice` для линукс ОС
- Загружаем бинарь на наш сервер через команду `scp` https://linuxize.com/post/how-to-use-scp-command-to-securely-transfer-files/
- Останавливаем Linux Service с нашим новым сервисом
- Подменяем старый бинарь на новый, старый удаляем
- Стартуем Linux Service
- Проверяем что все работает 

Для всего этого скрипта нам понадобиться системная тулза 
Линукс `systemctl` которая позволяет менеджить сервисы линукс.

----
To manage the state of the service you should use the `systemctl` linux tool.

The commands you will usually need:
- `systemctl start myservice.service` - start the service
- `systemctl restart myservice.service`- restart the service
- `systemctl stop myservice.service` - stop the service
---

Пример скрипта:
```shell
env GOOS=linux GOARCH=amd64 go build -o myBinary

server_ip="ec2-13-49-78-215.eu-north-1.compute.amazonaws.com"
server_username="ubuntu"

scp $server_username@$server_ip:~/myServer/myBinary

ssh $server_username@$server_ip "systemctl stop myServer.service"

ssh $server_username@$server_ip "mv ~/myServer/myBinary_new ~/myServer/myBinary"

ssh $server_username@$server_ip "chmod +x ~/myServer/myBinary"

ssh $server_username@$server_ip "systemctl start myServer.service"

ssh $server_username@$server_ip  "systemctl status -l myServer.service"

rm myServer
```

Если не работают команды `scp` & `ssh`, если они отваливаются по ошибке `Permission denied`
используйте фичу ssh с конфигурационным файлом. 

В папке `/.ssh` где лежат все ключи, сосздаем файл с названием `config`
и добавляем туда новый хост, например: 
```ssh
Host your_name_for_host
    HostName ec2-13-49-78-215.eu-north-1.compute.amazonaws.com
    IdentityFile ~/.ssh/tsm-test.pem
    IdentitiesOnly yes
    User ubuntu
    Port 22
```
Тогда в скрипте можно поменять 
- `scp $server_username@$server_ip:~/myServer/myBinary` на
- `scp your_name_for_host:~/myServer/myBinary`
