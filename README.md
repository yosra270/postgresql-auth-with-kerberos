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

![Principals List](img/principals_list_add_root.png)

[Useful video tutorial](https://www.youtube.com/watch?v=vx2vIA2Ym14)

## Steps to Configure PostgreSQL
