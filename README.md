## Master and Slave:
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
````

and in `slave-server`:

```
hostnamectl set-hostname slave-server
````

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

#Configuration of the Master-Server

In this step, we will configure the 'master server' with IP address '192.168.10.130'. 
We will create a new user/role with special permission to perform the replication, then we edit the PostgreSQL configuration file to enable the hot standby replication mode.
From the root privileges, switch to the PostgreSQL user with the su command:

```
su - postgres
````

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

````
listen_addresses = 'localhost,192.168.10.130'
...
````




Then:

mkdir -p /var/lib/postgresql/9.4/main/archive/
