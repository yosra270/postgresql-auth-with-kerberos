# PostgreSQL Authentication with Kerberos

Enabling GSSAPI / Kerberos authentication in PostgreSQL will allow ***single-sign-on*** – i.e. authentication without using the standard username and password for PostgreSQL clients.

## Steps To Setup Kerberos On UBUNTU

Kerberos is a network *authentication protocol* used to verify the identity of two or more *trusted hosts* across an *untrusted network*. It uses *secret-key cryptography* and a *trusted third party* (Kerberos Key Distribution Center) for authenticating client-server applications. Key Distribution Cente (KDC) gives clients tickets representing their network credentials. The Kerberos ticket is presented to the servers after the connection has been established.

![How does Kerberos work ?](https://upload.wikimedia.org/wikipedia/commons/b/b5/Kerberos-ruggiero.svg)


*!! Since Kerberos protocol has a timestamp involved, all three machines clocks need to be synchronized.*

### Hostname and IP Addresses

We will start by setting hostnames for each machine :
  * KDC machine       
      `hostnamectl --static set-hostname kdc.insat.tn`
  * Service Server machine        
      `hostnamectl --static set-hostname pg.insat.tn`
  * Client machine       
      `hostnamectl --static set-hostname client.insat.tn`
 
*We can check the hostname of a machine by running the command : `hostname`*

Next, we will be mapping these hostnames to their corresponding IP addresses on all three machines using */etc/hosts* file. <br> 
  `sudo vi /etc/hosts`
  
Now, we should set below information to */etc/hosts* **for all three machines** :

    <KDC_IP_ADDRESS>    kdc.insat.tn       kdc
    <PG_SERVER_ADDRESS>    pg.insat.tn        pg
    <CLIENT_ADDRESS>    client.insat.tn    client

*We can find these IP addreses by running the command : `hostname -I`*

Once the setup is done, we should restart the three machines.

### Key Distribution Center Machine Configuration

Following are the packages that need to installed on the KDC machine : <br>
 `sudo apt install krb5-kdc krb5-admin-server krb5-config`
 
During the installation, we will be asked for configuration of :
 * the realm : 'INSAT.TN' (must be *all uppercase*)
 * the Kerberos server : 'kdc.insat.tn'
 * the administrative server : 'kdc.insat.tn'
 
***Realm** is a logical network, similar to a domain, that all the users and servers sharing the same Kerberos database belong to.* 

The master key for this KDC database needs to be set once the installation is complete :
   
    `sudo krb5_newrealm`
    Enter KDC database master key: 
    Re-enter KDC database master key to verify:

*The users and services in a realm are defined as a **principal** in Kerberos.* These principals are managed by an *admin user* that we need to create manually :

    `sudo kadmin.local`
    kadmin.local:  add_principal root/admin
    Enter password for principal "root/admin@INSAT.TN": 
    Re-enter password for principal "root/admin@INSAT.TN": 
    Principal "root/admin@INSAT.TN" created.

[kadmin.local](https://web.mit.edu/kerberos/krb5-1.12/doc/admin/admin_commands/kadmin_local.html) is a KDC database administration program. We used this tool to create a new principal in the INSAT.TN realm (`add_principal`).

We can check if the user *root/admin* was successfully created by running the command : `kadmin.local: list_principals`. We should see the 'root/admin@INSAT.TN' principal listed along with other default principals.

Next, we need to grant all access rights to the Kerberos database to admin principal *root/admin* using the configuration file */etc/krb5kdc/kadm5.acl* . <br>
 `sudo vi /etc/krb5kdc/kadm5.acl`

In this file, we need to add the following line :

    */admin@INSAT.TN    *

For changes to take effect, we need to restart the following service : `sudo systemctl restart krb5-admin-server`

Once the admin user who manages princpals is created, we need to create the principals. We will to create principals for both the client machine and the service server machine.

**Create a principal for the client**

    `sudo kadmin.local`
    kadmin.local:  add_principal yosra
    Enter password for principal "yosra@INSAT.TN": 
    Re-enter password for principal "yosra@INSAT.TN": 
    Principal "yosra@INSAT.TN" created.

**Create a principal for the service server**

For the service server we will be creating two principals :
  * A principal which represents the database user and the Linux login user (we will name it *postgres*): <br>
    
         `sudo kadmin.local`
         kadmin.local:  add_principal postgres
         Enter password for principal "postgres@INSAT.TN": 
         Re-enter password for principal "postgres@INSAT.TN": 
         Principal "postgres@INSAT.TN" created.
    

  * A principal instance for Service server (*postgres/pg.insat.tn*) : <br>
    
         `sudo kadmin.local`
         kadmin.local:  add_principal postgres/pg.insat.tn
         Enter password for principal "postgres/pg.insat.tn@INSAT.TN": 
         Re-enter password for principal "postgres/pg.insat.tn@INSAT.TN": 
         Principal "postgres/pg.insat.tn@INSAT.TN" created.
    

*We can check the list of principals by running the command : `kadmin.local: list_principals`* <br>

### Service server Machine Configuration

#### Configuration of Kerberos

##### Installation of Packages  

Following are the packages that need to be installed on the Service server machine : <br>
 `sudo apt install krb5-user libpam-krb5 libpam-ccreds auth-client-config`
 
During the installation, we will be asked for the configuration of :
 * the realm : 'INSAT.TN' (must be *all uppercase*)
 * the Kerberos server : 'kdc.insat.tn'
 * the administrative server : 'kdc.insat.tn'

PS : *We need to enter the same information used for KDC Server.*

##### Preparation of the *keytab file*

We need to extract the service principal from KDC principal database to a keytab file.

1. In the KDC machine run the following command to generate the keytab file in the current folder :

   ```
   $ ktutil 
   ktutil:  add_entry -password -p postgres/pg.insat.tn@INSAT.TN -k 1 -e aes256-cts-hmac-sha1-96
   Password for postgres/pg.insat.tn@INSAT.TN: 
   ktutil:  wkt postgres.keytab
   ktutil:  exit
   ```

2. Send the keytab file from the KDC machine to the Service server machine :

In the Postgres server machine make the following directories :
   
   `mkdir -p /home/postgres/pgdata/data`

In the KDC machine send the keytab file to the Postgres server :

   `scp postgres.keytab postgres@<PG_SERVER_IP_ADDRESS>:/home/postgres/pgdata/data`
   
3. Verify that the service principal was succesfully extracted from the KDC database :

   ```
   postgres@pg:~$ ktutil 
   ktutil:  list
   slot KVNO Principal
   ---- ---- ---------------------------------------------------------------------
   ktutil:  read_kt postgres.keytab 
   ktutil:  list
   slot KVNO Principal
   ---- ---- ---------------------------------------------------------------------
    1    1          postgres/pg.insat.tn@INSAT.TN
   ```

#### Configuration of the service (PostgreSQL)

##### Installation of PostgreSQL 

1. Update the package lists 
 
   `sudo apt update`
 
2. Install necessary packages for Postgres

   `sudo apt install postgresql postgresql-contrib`

3. Ensure that the service is started

   `sudo systemctl start postgresql.service`



##### Create a Postgres Role for the Client 

```
$ postgres

postgres=# create user 'yosra' with encrypted password 'some_password';
```
To ensure the role was successfully created run the following command :

```
postgres=# SELECT username FROM pg_user WHERE username LIKE 'yosra';
```

The user yosra has now a role in Postgres and can access its default database 'yosra'.


##### Update Postgres Configuration files (*postgresql.conf*, *pg_hba.conf* and *pg_ident.conf*)

* Updating *postgresql.conf*

To edit the file run the following command :

`vi .../postgresql.conf`

By default, Postgres Server only allows connections from localhost. Since the client will connect to the Postgres server remotely, we will need to modify *postgresql.conf* so that Postgres Server allows connection from the network :

```
listen_addresses = '*'
```

 We will also need to specify the keytab file location :
 ```
 krb_server_keyfile = '/home/postgres/pgdata/data/postgres.keytab'
 ```


* Updating *pg_hba.conf*

HBA stands for host-based authentication. *pg_hba.conf* is the file used to control clients authentication in PostgreSQL. It is basically a set of records. Each record specifies a **connection type**, a **client IP address range**, a **database name**, a **user name**, and the **authentication method** to be used for connections matching these parameters.

The first field of each record specifies the **type of the connection attempt** made. It can take the following values :

* `local` : Connection attempts using *Unix-domain sockets* will be matched. 
* `host` : Connection attempts using *TCP/IP* will be matched (SSL or non-SSL as well as GSSAPI encrypted or non-GSSAPI encrypted connection attempts).
* `hostgssenc` : Connection attempts using *TCP/IP* will be matched, but only when the connection is made with GSSAPI encryption.

This field can take other values that we won't use in this setup. For futher information you can visit the official [documentation](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html).

Some of the possible choices for the **authentication method** field are the following :

* `trust` : Allow the connection unconditionally. 
* `reject` : Reject the connection unconditionally. 
* `md5` : Perform SCRAM-SHA-256 or MD5 authentication to verify the user's password.
* `gss` : Use GSSAPI to authenticate the user. 
* `peer` : Obtain the client's operating system user name from the operating system and check if it matches the requested database user name. This is only available for *local connections*.

So to allow the user 'yosra' to connect remotely using  Kerberos we will add the following line :

```
# IPv4 local connections:
hostgssenc   yosra     yosra           <IP_ADDRESS_RANGE>         gss
```


* Updating *pg_ident.conf*

### Client Machine Configuration

Following are the packages that need to be installed on the Client machine : <br>
 `sudo apt install krb5-user libpam-krb5 libpam-ccreds auth-client-config`
 
During the installation, we will be asked for configuration of :
 * the realm : 'INSAT.TN' (must be *all uppercase*)
 * the Kerberos server : 'kdc.insat.tn'
 * the administrative server : 'kdc.insat.tn'


PS : *We need to enter the same information used for KDC Server.*

![Principals List](img/principals_list_add_root.png)

[Useful video tutorial](https://www.youtube.com/watch?v=vx2vIA2Ym14)

## Steps to Configure PostgreSQL
