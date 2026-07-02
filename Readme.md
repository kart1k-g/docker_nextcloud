# Structure
```
docker_nextcloud
├── .gitignore
├── Readme.md
├── reverse-proxy
│   ├── .env
│   ├── compose.yml
│   └── nginx
│       └── conf.d
│           └── nextcloud.conf
└── services
    ├── .env
    └── compose.yml
```
# Reverse Proxy 
I assume that the host machine has no reverse proxy of its own. I will be using nginx as the reverse proxy and certbot to obtain ssl certificates for the entire host machine.

1. It can be used to route all traffic for the machine. Though, this repo will be using it only for nextcloud traffic.
2. Place the **reverse-proxy** folder preferably inside **/opt**  (as this is meant for this type of services) or on any place of your choice on the host. Folder placement will not affect the working.

3. Obtain ssl certificates for the same domain using certbot. Run (use the domain you'll be using for nextcloud). Let's say its nc.example.com:
    >sudo apt update  
    >sudo apt install certbot -y  
    >sudo certbot certonly --standalone -d nc.example.com

    To verify, view the dir:
    > sudo ls /etc/letsencrypt/live/nc.example.com


    It must contain these files
    ```
    cert.pem  chain.pem  fullchain.pem  privkey.pem
    ```

4. Nginx config for nextcloud is:
    >/reverse-proxy/nginx/conf.d/nextcloud.conf

    Note: If you want to host other services on the same host, you can simply place their **.conf** files in **/conf.d** dir as well.

    The **nextcloud.conf** is taken from the official nextcloud docs, it is for the urls of form *http(s)://nextcloud.example.com/*. If you'll be using url of the form *http(s)://example.com/nextcloud* then refer to the official docs:
    > https://docs.nextcloud.com/server/stable/admin_manual/installation/nginx.html
    
    A few tweeks are to be done for our setup :
    
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
    #use your domain by replacing every occurance:
    nc.kart1kg.com -> nc.example.com
    ```

5. I'm using a **bind mount** for ssl certificates in the **compose.yml** since these are managed by certbot on the host.

6. The **compose.yml** defines an internal network **proxy** managed by **nginx** and other services will connect to it and **not create a new network**. Run the following command to create **proxy**
    > docker network create proxy

    Note: Docker prefixes network names with directory name. If you create a network in the compose.yml file (by **not** setting **external: true** )it might not resolve correctly.

7. Create **/reverse-proxy/.env** with:
    ```
    #/path/to/store/nextcloud/data/on/host
    NEXTCLOUD_DATA_DIR=$HOME/nextcloud-data
    ```
    Instead of *$HOME/nextcloud-data*, you may choose to store your data anywhere.

# Database
Nextcloud requires a database to track its user information. Any standard RDBMS would do the job, here I choose Mariadb.

You may place **/service** anywhere instead of **/opt** .

1.  In **compose.yml** the following two command options are from the Nextcloud's recommendation:

    * --transaction-isolation=READ-COMMITTED
    * --binlog-format=ROW 

2. Create **/services/.env** with the following fields and choose some **strong password**.
    ```
    #super root mariadb password
    MYSQL_ROOT_PASSWORD=some-very-long-random-password

    #nextcloud user mariadb password
    MYSQL_PASSWORD=another-very-long-random-password
    ```

3. This database will only talk to nextcloud and hence we only expose it to **nextcloud** network. To create it run the following:
    >docker network create nextcloud  

Note: SQLite works fine just like other RDBMSs with a caveat that it **doesn't** support concurrent write operations like othes. This may lead to scalability issues.

# Nextcloud
Having setup the minimum prerequisite to selfhost Nextcloud, we can now move on with its installation. 

1. Since I have nginx , here I've used an **fpm** image as the default nextcloud image ships with apache.
2. I'll be using **bind mount** and **not volumes** to store my data that I upload to Nextcloud. This way it's easy to manage and migrate as it is **managed** by the host(me) and not docker.
3. In the **/services/.env** specify :
    ```
    #/path/to/store/nextcloud/data/on/host
    NEXTCLOUD_DATA_DIR=$HOME/nextcloud-data
    ```
    Keep this path same as the one menioned in **/reverse-proxy/.env**.

# Redis (Optional)
Redis provides in memory caching our content on nextcloud and can provide speedup. To use it, we'll have to let know nextcloud about it.
1. The configuration for the same lies in:
    > $HOME/nextcloud-data/config/config.php  

    Use the path specified in the variable **NEXTCLOUD_DATA_DIR** instead of *$HOME/nextcloud-data*

2. I'll use nano to edit it, run the following:
    > sudo nano $HOME/nextcloud-data/config/config.php  

    Add/modify the following properties:
    ```
    'memcache.local' => '\OC\Memcache\APCu',

    'filelocking.enabled' => true,

    'memcache.locking' => '\OC\Memcache\Redis',

    'redis' => [
        'host' => 'redis',
        'port' => 6379,
    ],

    'trusted_proxies' =>
    array (
        0 => 'nginx',
    ),    
    ```