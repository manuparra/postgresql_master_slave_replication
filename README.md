# Environment

- Two server:
  - Master-Server: IP 192.168.10.130
  - Slave-Server:  IP 192.168.10.131
- Ubuntu 16.04
- PostgreSQL 9.4

# Setting-up Master and Slave

```
sudo apt-get -y update
sudo apt-get -y upgrade
reboot
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install postgresql-9.4
```

If you have BIND error, try to change: 

```
vi /etc/postgresql/9.4/main/postgresql.conf
```

Change bind to 'localhost' and restart postgresql: `service postgresql restart`

Set the hostname in `master-server`:

```
hostnamectl set-hostname master-server
```

and in `slave-server`:

```
hostnamectl set-hostname slave-server
```

Change host file in each server and add in `/etc/hosts/`:

```
vi /etc/hosts
```	

Add to the file:

```
192.168.10.130	master-server
192.168.10.131	slave-server
```

Execute this command in both servers:	
	
```
passwd postgres
```

Then change to the `postgres` user:

```
su - postgres
psql
\conninfo
```

# Configuration of the Master-Server

In this step, we will configure the 'master server' with IP address '192.168.10.130'. 
We will create a new user/role with special permission to perform the replication, then we edit the PostgreSQL configuration file to enable the hot standby replication mode.
From the root privileges, switch to the PostgreSQL user with the su command:

```
su - postgres
```

Access the Postgres shell with the psql command and type in this PostgreSQL query to create the new user/role:

```
psql
CREATE USER replica REPLICATION LOGIN ENCRYPTED PASSWORD 'replicauser@';
```

Check new replica user with PostgreSQL command below:
```
\du
```
New replica user has been created.

Next, go to the PostgreSQL directory '/etc/postgresql/9.4/main' to edit the configuration file:

```
vi postgresql.conf
```

Change the following lines (uncomment if needed):

```
listen_addresses = 'localhost,192.168.10.130'
...
wal_level = hot_standby
...
checkpoint_segments = 8
...
archive_mode = on
archive_command = 'cp -i %p /var/lib/postgresql/9.4/main/archive/%f'
...
max_wal_senders = 3
wal_keep_segments = 8
...
```

Save and exit. 

Change to postgres user:

```
su - postgres
```
And create the next folder:

´´´
mkdir -p /var/lib/postgresql/9.4/main/archive/
´´´

Edit `/etc/postgresql/9.4/main/pg_hba.conf` and add to the end, your Slave Servers:

```
host    replication     replica      192.168.10.131/24            md5
```
Remember, your slave server is `192.168.10.131`.

Save and exit, and restart postgresql with:

```
service posrgresql restart
```

# Configuration of the Slave-Server

Change to postgres user and edit configuration file:

```
su - postgres
vi /etc/postgresql/9.4/main/postgresql.conf
```

Change the following lines (or uncomment):

```
listen_addresses = 'localhost,192.168.1.248'
...
wal_level = hot_standby
...
checkpoint_segments = 8
...
max_wal_senders = 3
...
wal_keep_segments = 8
...
hot_standby = on
```

Save and exit.

#Sycn data from the master to the slave

In this step, we will move the PostgreSQL data directory '/var/lib/postgresql/9.4/main' to a backup folder and then replace it with the latest master data with 
the command `pg_basebackup` command.

On the SLAVE SERVER, stop PostgreSQL Service:

```
service postgresql stop
```

Now login with postgres user:

```
su - postgres
```
and execute:

```
mv 9.4/main 9.4/main_original
```

Run the following command to copy data from the master to the slave:

```
pg_basebackup -h 192.168.10.130 -D /var/lib/postgresql/9.4/main -U replica -v -P
```
The IP 192.168.10.130 of your Master-Server. This command will ask your password (for replication).

Edit the following file:

```
vi /var/lib/postgresql/9.4/main/recovery.conf
```

and paste the following:

```
standby_mode = 'on'
primary_conninfo = 'host=192.168.1.230 port=5432 user=replica password=replicauser@'
restore_command = 'cp //var/lib/postgresql/9.4/main/archive/%f %p'
trigger_file = '/tmp/postgresql.trigger.5432'
```
Save the file and restart the services:

```
service postgresql restart
```

# Testing the environment




