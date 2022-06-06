# LAB_BGS2
# Лабораторная работа №2
## Предмет: Безопасность гегеоинформационных систем
 *Выполнили студенты группы РИ-571227:*
 
 *- Иванов Роман*
 
 *- Токарева Виолетта*
 
 *- Худкова Елена*
 
<div id="header" align="center">
  <img src="https://media.giphy.com/media/lQ7l3COBczbm8cC0WD/giphy.gif" width="500" height="400"/>
</div>

Настройки были произведены в файле *postgresql.conf*. Лучший результат был достигнут со следующей конфигурацией:

> synchronous_commit = remote_apply

synchronous_commit (enum):
> Определяет, после завершения какого уровня обработки WAL сервер будет сообщать об успешном выполнении операции. Допустимые значения: remote_apply (применено удалённо), on (вкл., по умолчанию), remote_write (записано удалённо), local (локально) и off (выкл.).

Значение *remote_apply* влияет на фиксирование ответов. Фиксирование завершается только после получения ответов от текущих синхронных ведомых серверов, говорящих, что они получили запись о фиксировании транзакции, сохранили её в надёжном хранилище, а также применили транзакцию, так что она стала видна для запросов на этих серверах. (подробнее: https://postgresqlco.nf/doc/ru/param/synchronous_commit/)

Худший результат получен с конфигурацией:

> synchronous_commit = off

Ниже в таблице приведены характеристики различных значений *synchronous_commit*, где наглядно показана эффективность значений:

![image](https://user-images.githubusercontent.com/87654857/172142834-603a05e8-97b4-4553-9018-7661b84f4aaa.png)


## Листинг программы:

```

import os
import psycopg2
from psycopg2 import Error

def Closed(ip_addr_m):
    os.system(f"ssh postgres@{ip_addr_m} 'ifconfig eth0 down'")
    os.system(f"ssh postgres@{ip_addr_m} 'sleep 10'")
    os.system(f"ssh postgres@{ip_addr_m} 'ifconfig eth0 down'") 

def Availability(host):
    response = os.system("ping " + host)
    return response

def Detect(ip_addr_m,ip_addr_s):
    while Availability(ip_addr_m) == 0:
        connection = psycopg2.connect(dbname='Test', user='postgres', password='12345', host = ip_addr_m )
        cur = connection.cursor()
        cur.execute('SELECT COUNT(*) FROM Test;')
        count_m = cur.fetchone()

        connection2 = psycopg2.connect(dbname='Test', user='postgres', password='12345', host = ip_addr_s  )
        cur2 = connection2.cursor()
        cur2.execute('SELECT COUNT(*) FROM Test;')
        count_r = cur2.fetchone()

        print(f"Записей в таблице Test основной БД насчитывается: {count_m[0]}\n Записей в таблице Test реплики БД насчитывается: {count_r[0]}")
        if count_m[0] == count_r[0]:
            print("Все в норме, количество записей в таблице Test совпадает.")
            exit(0)
        if count_m[0] != count_r[0]:
            print(f"Внимание, количества записей в таблице Test не совпадает и насчитывается: {(count_m[0]-count_r[0])}")
            exit(0)

async def Start(ip_addr_m,ip_addr_s):
    connection = psycopg2.connect(dbname='Test', user='postgres', password='12345', host = ip_addr_m )
    try:
        i = 0
        while i < 100000:
            async with connection.transaction():
                await connection.execute(f"INSERT INTO Test (type, numberof) VALUES ('{i}','{i}');")
            if i == 1000:
                Closed()
            i = i + 1
    except:
        pass

def main():

    ip_addr_m = '192.168.200.100'
    ip_addr_s = '192.168.200.101'
    
    while( True ):
        Start(ip_addr_m,ip_addr_s)

if __name__ == "__main__":
    main()

```

