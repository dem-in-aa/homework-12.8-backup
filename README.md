# Домашнее задание к занятию «Резервное копирование баз данных» Андрей Дёмин


### Задание 1. Резервное копирование

### Кейс
Финансовая компания решила увеличить надёжность работы баз данных и их резервного копирования. 

Необходимо описать, какие варианты резервного копирования подходят в случаях: 

1.1. Необходимо восстанавливать данные в полном объёме за предыдущий день.

1.2. Необходимо восстанавливать данные за час до предполагаемой поломки.

1.3.* Возможен ли кейс, когда при поломке базы происходило моментальное переключение на работающую или починенную базу данных.

*Приведите ответ в свободной форме.*

Ответ:

1.1 Дифференциальный бэкап справится с этой задачей быстрей всех остальных методов. Дифференциальная резервная копия позволяет быстрее восстанавливать данные по сравнению с инкрементным резервным копированием, поскольку для этого требуется всего две части резервной копии: полная резервная копия и последняя дифференциальная резервная копия. 

1.2 Необходимо инкрементное резервное копирование каждый час. Инкрементное резервное копирование можно выполнять так часто, как требуется, так как сохраняются только копии последних изменений. Для ускорения восстановления данных желательно чаще проводить full backup, насколько это позволяют возможности хранилища, например 2-3 раза в неделю или даже ежесуточно.

1.3* Необходимо приобрести оборудование и создать кластер репликации на основе master-slave, DRBD или SAN-кластер. 
Как вариант использовать инструмент [Veeam Explorer for SQL Server или Veeam Explorer for Oracle из комплекта Veem Backup & Replication v.11](https://habr.com/ru/companies/veeam/articles/566274/), предоставляющих  возможность мгновенного восстановления базы данных  (Instant recovery). Подход здесь напоминает всем знакомое мгновенное восстановление виртуальной машины - быстро смонтировать файлы на рабочий сервер, чтобы, например, достать нужные отчеты для бухгалтерии, и в это же время в фоновом режиме спокойно копировать файлы бэкапа и по готовности переключиться на полноценную машину. 
Сам же процесс восстановления проходит так:
1) Veeam Explorer запускает параллельно 2 сессии монтирования файлов бэкапа по iSCSI (iSCSI mount sessions). В ходе одной из них база из бэкапа публикуется на продакшен сервер и аттачится к нужному инстансу SQL Server. (При публикации БД выполняется ее временный аттач к выбранному серверу без собственно восстановления). В ходе второй в фоновом режиме идет копирование файлов из бэкапа на продакшен сервер.

Примечание: Пока пользователи работают с опубликованной базой (“запаской”), все изменения файлов базы сохраняются в кэш на маунт-сервере. 

2) После того, как все файлы из бэкапа скопировались на целевой сервер, происходит синхронизация изменений, накопившихся в кэше.
3) На финальном этапе выполняется переключение на актуальную базу, поднятую из бэкапа на продакшене и успешно синхронизированную. Переключение можно запустить вручную или автоматически.
---

### Задание 2. PostgreSQL

2.1. С помощью официальной документации приведите пример команды резервирования данных и восстановления БД (pgdump/pgrestore).

Ответ:

Создать дамп при помощи утилиты pgdump:
```sql
pg_dump -U user > /tmp/my.dump
```
Восстановление с использованием pgrestore:
```sql
pg_restore -d mydb my.dump
```
2.1.* Возможно ли автоматизировать этот процесс? Если да, то как?

Ответ:

С использованием скрипта:

```bash
#!/bin/sh
PATH=/etc:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin

PGPASSWORD=password
export PGPASSWORD
pathB=/backup
dbUser=dbuser
database=db

find $pathB \( -name "*-1[^5].*" -o -name "*-[023]?.*" \) -ctime +61 -delete
pg_dump -U $dbUser $database | gzip > $pathB/pgsql_$(date "+%Y-%m-%d").sql.gz

unset PGPASSWORD
```
Запуск скрипта по расписанию через crontab: 
```bash
0 0 * * * /scripts/postgresql_dump.sh
```
---

### Задание 3. MySQL

3.1. С помощью официальной документации приведите пример команды инкрементного резервного копирования базы данных MySQL. 

Вариант 1:
```sql
oci mysql backup create 
--db-system-id <DBSystemOCID> 
--backup-type incremental 
--display-name <BackUpName>
--retention-in-days <NumberOfDays>
```
Примечание:

- db-system-id: Укажите OCID системы базы данных.

- backup-type: (Необязательно) Укажите тип резервной копии, ПОЛНОЙ или ДОБАВОЧНОЙ. Значение по умолчанию - ИНКРЕМЕНТНОЕ.

- displayName: (Необязательно) Укажите отображаемое имя системы базы данных. Если вы не определяете отображаемое имя, оно создается для вас в mysqlbackupYYYYMMDDHHMMSS формате.

- retention-in-days: (Необязательно) Укажите количество дней, в течение которых сохраняется резервная копия. Период хранения по умолчанию составляет 365 дней.

Источник:
```
https://docs.oracle.com/en-us/iaas/mysql-database/doc/creating-manual-backup.html#GUID-4DDABC9A-F649-40A4-90C0-12AFFF4C8276
```
Вариант 2:
С помощью утилиты XtraBackup:
```
xtrabackup --backup --target-dir=/data/backups/inc1 --incremental-basedir=/data/backups/full
xtrabackup --backup --target-dir=/data/backups/inc2 --incremental-basedir=/data/backups/inc1
```
3.1.* В каких случаях использование реплики будет давать преимущество по сравнению с обычным резервным копированием?

Ответ:

Репликация имеет преимущество перед резервным копированием в системах высокой доступности, таких как торговые сети и онлайн-бизнес, в которых процесс восстановления из резервной копии может занять несколько дней, что влечет финансовые и репутационные потери. При этом переключение с основной базы на реплику займёт 1-3 минуты. Приложение продолжает работать, простой и потеря данных – минимальные.
Но наиболее грамотный вариант – использование двух технологий в тандеме. Репликацию – для обеспечения непрерывности работы приложения в случае аварии, бэкап – для сохранения целостности и доступности данных, если они повредились из-за поломки дисков, на которых они хранятся, атаки вируса-шифровальщика или других непредвиденных ситуаций.

---
Задания, помеченные звёздочкой, — дополнительные, то есть не обязательные к выполнению, и никак не повлияют на получение вами зачёта по этому домашнему заданию. Вы можете их выполнить, если хотите глубже шире разобраться в материале.
