# Reverse Proxy 
I assume that the host machine has no reverse proxy of its own. I will be using nginx as the reverse proxy and certbot to obtain ssl certificates for the entire host machine.

1. It can be used to route all traffic for the machine. Though, this repo will be using it only for nextcloud traffic.
2. Place the **reverse-proxy** folder preferably inside **/opt**  (as this is meant for this type of services) or on any place of your choice on the host. Folder placement will not affect the working.
3. Nginx config for nextcloud is placed inside:
    >/reverse-proxy/nginx/conf.d/

    Note: If you want to host other services on the same host, you can simply place their nginx files inside the above dir.

    The nextcloud.conf is taken from the official docs, it is for the urls of form *http(s)://cloud.example.com/*. If you'll be using url of the form *http(s)://cloud.example.com/nextcloud/* then refer to the officail docs:
    > https://docs.nextcloud.com/server/stable/admin_manual/installation/nginx.html
    
    with a few tweeks :
    
    ```
    upstream php-handler {
        server nextcloud:9000;
    }
    ```
    ```
    # Path to the root of your installation
    root /var/www/html;
    ```

    ```
    replace 'nextcloud.kart1kg.com' with the domain you wish to use for nextcloud
    ```


4. Also create the **letsencrypt** folder inside **reverse-proxy** for cerbot to manage ssl certificates.

5. The **compose.yml** defines an internal network **proxy** managed by **nginx** and other services will connect to it and **not create a new network**.

```
Folder structure:

/opt/reverse-proxy/
├── compose.yml
├── nginx/
│   ├── conf.d/
│   │   └── nextcloud.conf
└── letsencrypt/
```

# Database
Nextcloud requires a database to track its user information. Any standard RDBMS would do the job, here I choose Mariadb.

Here is the folder structure (again, you may place **/service** anywhere instead of **/opt**):
```
/opt/services/mariadb
├── compose.yml
└── .env
```

1.  The following two command options are from the Nextcloud's recommendation:

    * --transaction-isolation=READ-COMMITTED
    * --binlog-format=ROW 

2. Create a **.env** inside **/mariadb** with the following fields and choose some **strong password**.
```
MYSQL_ROOT_PASSWORD=some-very-long-random-password
MYSQL_PASSWORD=another-very-long-random-password
```

3. This database will only talk to nextcloud and hence we only expose it to **nextcloud** network which will be created later (in the compose file for nextcloud itself).

Note: SQLite works fine just like other RDBMSs with a caveat that it **doesn't** support concurrent write operations like othes. This may lead to scalability issues.