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

    KDC_IP_ADDRESS    kdc.insat.tn       kdc
    PG_SERVER_ADDRESS    pg.insat.tn        pg
    CLIENT_ADDRESS    client.insat.tn    client

*We can find these IP addreses by running the command : `hostname -I`*

Once the setup is done, we should restart the three machines.

## Steps to Configure PostgreSQL
