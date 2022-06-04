# Настройка отказоустойчивой БД.
Требуется: DataGrip, Ubuntu, Python3.
***
## Настройка Master.

Установим postgresql.

    sudo apt-get update
    sudo apt-get install postgresql postgresql-contrib

Репликацию будем проводить с пользователя postgres, который создается автоматически при установке.

Так как используется DataGrip, который при подключении к базе данных требует пароль от пользователя, то создание пароля обязательно. Пустой пароль вызовет ошибку.

Создадим пароль для postgres.

    su - postgres
    psql -c "ALTER ROLE postgres PASSWORD 'сюда нужно указать Ваш пароль'"

Теперь нужно разрешить этому пользователю подключаться из Replica-сервера в Master

Для этого мы отредактируем файл /etc/postgresql/'Ваша версия postgresql'/main/pg_hba.conf.

    nano /etc/postgresql/'Ваша версия postgresql'/main/pg_hba.conf

Теперь нужно найти строку "If you want to allow non-local connections, you need to add more" и добавить после нее следующую строку:

    host    replication    postgres   'IP адрес Replica-сервера'/32    md5

Далее следует указать настройки репликации. Нужно отредакитровать файл /etc/postgresql/'Ваша версия postgresql'/main/postgresql.conf.

    nano /etc/postgresql/'Ваша версия postgresql'/main/postgresql.conf

Требуется найти, раскомментировать и указать следующие значения:

    listen_addresses = 'localhost, 'IP адрес Master-сервера''
    wal_level = hot_standby
    archive_mode = on
    archive_command = 'cd .'
    max_wal_senders = 8
    hot_standby = on

Выполнить перезагрузку postgresql.

    service postgresql restart

## Настройка Replica

Установим postgresql.

    sudo apt-get update
    sudo apt-get install postgresql postgresql-contrib

По аналогии с настройкой Master-сервера, добавляем пароль для пользователя postgres и подключаемся к DataGrip.

    su - postgres
    psql -c "ALTER ROLE postgres PASSWORD 'сюда нужно указать Ваш пароль'"

Для начала, отстановим работу postgresql.

    service postgresql stop

Откроем и настроим файл /etc/postgresql/'Ваша версия postgresql'/main/pg_hba.conf

    nano /etc/postgresql/'Ваша версия postgresql'/main/pg_hba.conf

Добавим в него строку: 

    host    replication    postgres    'IP адрес Master-сервера'/32    md5

Затем открываем файл /etc/postgresql/'Ваша версия postgresql'/main/postgresql.conf, находим, раскомментируем и указываем следующие значения:

    listen_addresses = 'localhost, REPLICA_ВНУТРЕННИЙ_IP'
    wal_level = hot_standby
    archive_mode = on
    archive_command = 'cd .'
    max_wal_senders = 8
    hot_standby = on

*Сейчас настройки обоих серверов одинаковые, отличаются только IP-адреса. Это потому, что при необходимости реплики могут становиться мастером, а вся разница будет в наличии одного лишь файла.*

Прежде чем Replica-сервер сможет начать реплицировать данные, нужно создать новую БД, идентичную Master-серверу. Для этого воспользуемся утилитой pg_basebackup. Она создаст бэкап с Master-сервера и скачает его на Replica-сервер. Эту операцию нужно выполнять от имени пользователя postgres.

Перейдем на пользователя postgres.

    su - postgres

Далее переходим в каталог с базой данных.

    cd /var/lib/postgresql/'Ваша версия postgresql'/

Удалим здесь каталог стандартной базы данных. Создадим ее снова, но уже пустую. Установим требуемые права.

    rm -rf main && mkdir main && chmod go-rwx main

Теперь сделаем копию БД Master-сервера.

    pg_basebackup -P -R -X stream -c fast -h 'IP адрес Master-сервера' -U postgres -D ./main

В этой команде есть важный параметр -R. Он означает, что PostgreSQL-сервер также создаст пустой файл standby.signal. Несмотря на то, что файл пустой, само наличие этого файла означает, что этот сервер — реплика.

Теперь запускаем сервис postgresql.

    service postgresql start

## Проверка репликации.

Будем использовать следующий Python-скрипт:

    def Replication_process(Master_ip, Replica_ip):
        id = 1
        flag_promote = False
        flag_master_drop = False
        while ( True ):
            if ( ping_check(Master_ip) and flag_master_drop == False ):
                print(f"[{id}] Master is connected")
                connection_check(Master_ip)

            if ( not ping_check(Master_ip) and ping_check(Replica_ip) ):
                print(f"[{id}] [Master is not available, connected to Replica]")
                connection_check(Replica_ip)
                flag_master_drop = True
                if (flag_promote == False):
                    os.system(f"ssh postgres@{Replica_ip} '/usr/lib/postgresql/14/bin/pg_ctl promote -D /var/lib/postgresql/14/main > /dev/null'")
                    flag_promote = True
                    print("[Replica is main server now].")

            if ( ping_check(Master_ip) and flag_master_drop ):
                print((f"[{id}] Connection with Master restored!"))
                connection_check(Master_ip)
                try:
                    os.system(f"ssh postgres@{Master_ip} 'rm -rf /var/lib/postgresql/14/main && mkdir /var/lib/postgresql/14/main && chmod go-rwx /var/lib/postgresql/14/main && pg_basebackup -P -X stream -c fast -h 192.168.200.101 -U postgres -D /var/lib/postgresql/14/main && sudo systemctl restart postgresql'")
                    os.system(f"ssh postgres@{Replica_ip} 'rm -rf /var/lib/postgresql/14/main && mkdir /var/lib/postgresql/14/main && chmod go-rwx /var/lib/postgresql/14/main && pg_basebackup -P -R -X stream -c fast -h 192.168.200.100 -U postgres -D /var/lib/postgresql/14/main && sudo systemctl restart postgresql'")
                    flag_master_drop = False
                    flag_promote = False
                    print("Replication successfully!") 
                except (Exception, Error) as error:
                    print(f"Replication error!")
            id+=1
            time.sleep(2)

    def main():

        Master_ip = '192.168.200.100'
        Replica_ip = '192.168.200.101'
    
        Replication_process(Master_ip, Replica_ip)

    if __name__ == "__main__":
        main()

Он позволяет проверить связь с Master-сервером и Replica-сервером, проверить доступность БД, при потери соединения с Master-сервером переключиться на Replica-сервер, при восстановлении соединения с Master-сервером переключиться на Master-сервер и клонировать данные с Replica-сервера. 

Результат работы скрипта и тестирование репликации:

![unknown_2022_06_04-00_12_Trim](https://user-images.githubusercontent.com/97679190/171991264-e379a0c7-c189-45ff-bd35-7141fe6fa7d6.gif)












