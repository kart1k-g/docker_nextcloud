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
    ├── compose.yml
    └── php
        └── custom.ini
```
# Storage Type Used
```
MariaDB                 -> named volume  
Redis                   -> named volume  
Nextcloud Installation  -> named volume  
Nextcloud USer Data     -> bind mount  
Nginx conf              -> bind mount  
SSL certs               -> bind mount  
```
# Docker Setup On Host

```
sudo apt update
sudo apt install docker.io docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```
If *docker-compose-plugin* is not found then use *docker-compose-v2* instead.  
**Reboot the instance**, for user joining docker group change to take effect.
# Firewall 
This one often gets me scratching my head as to what the hell is happening!  
Basically we need ports **80 and 443** open on our host for incoming HTTP/HTTPS traffic.  
Do it for your system firewall.
1. For **iptables firewall**, run: 
    ```
    sudo iptables -I INPUT 1 -p tcp --dport 80 -j ACCEPT
    sudo iptables -I INPUT 1 -p tcp --dport 443 -j ACCEPT
    sudo apt install iptables-persistent
    sudo netfilter-persistent save
    ```
2. To verify:
    ```
    sudo iptables -L INPUT -v -n --line-numbers
    ```
    You must get something similar
    > 1       16   896 ACCEPT     6    --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:443  
    >2       11   620 ACCEPT     6    --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80  

    Notice **80** and **443** at the end of lines
    
# Reverse Proxy 
I assume that the host machine has no reverse proxy of its own. I will be using nginx as the reverse proxy and certbot to obtain ssl certificates for the entire host machine.

Reminder: Configure your DNS to point to the host's IP.

1. It can be used to route all traffic for the machine. Though, this repo will be using it only for nextcloud traffic.
2. Place the **reverse-proxy** folder preferably inside **/opt**  (as this is meant for this type of services) or on any place of your choice on the host. Folder placement will not affect the working.

3. Obtain ssl certificates for the same domain using certbot. Run (use the domain you'll be using for nextcloud). Let's say its nc.example.com:
    ```
    sudo apt update  
    sudo apt install certbot -y  
    sudo certbot certonly --standalone -d nc.example.com
    ```

    To verify, view the dir:
    ```
    sudo ls /etc/letsencrypt/live/nc.example.com
    ```

    It must contain these files
    
    >cert.pem  
    chain.pem   
    fullchain.pem  
    privkey.pem
    

4. Nginx config for nextcloud is:
    >/reverse-proxy/nginx/conf.d/nextcloud.conf

    Note: If you want to host other services on the same host, you can simply place their **.conf** files in **/conf.d** dir as well.

    The **nextcloud.conf** is taken from the official nextcloud docs, it is for the urls of form *http(s)://nextcloud.example.com/*. If you'll be using url of the form *http(s)://example.com/nextcloud* then refer to the official docs:
    > https://docs.nextcloud.com/server/stable/admin_manual/installation/nginx.html
    
    A few tweeks were done for the setup :
    
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
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    ```
    **You need to change the below part**:

    ```
    #use your domain by replacing every occurance:
    nc.kart1kg.com -> nc.example.com
    ```

5. I'm using a **bind mount** for ssl certificates in the **compose.yml** since these are managed by certbot on the host.

6. The **compose.yml** defines an internal network **proxy** managed by **nginx** and other services will connect to it and **not create a new network**. Run the following command to create **proxy**
    ```
    docker network create proxy
    ```

    Note: Docker prefixes network names with directory name. If you create a network in the compose.yml file (by **not** setting **external: true** )it might not resolve correctly.

7. Create **/reverse-proxy/.env** with:
    ```
    #/path/to/store/nextcloud/data/on/host
    NEXTCLOUD_DATA=$HOME/nextcloud-data
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
    ```
    docker network create nextcloud  
    ```

Note: SQLite works fine just like other RDBMSs with a caveat that it **doesn't** support concurrent write operations like othes. This may lead to scalability issues.

# Nextcloud
Having setup the minimum prerequisite to selfhost Nextcloud, we can now move on with its installation. 

1. Since I have nginx , here I've used an **fpm** image as the default nextcloud image ships with apache.
2. I'll be using **bind mount** and **not volumes** to store my data that I upload to Nextcloud. This way it's easy to manage and migrate as it is **managed** by the host(me) and not docker.
3. In the **/services/.env** specify :
    ```
    #/path/to/store/nextcloud/data/on/host
    NEXTCLOUD_DATA=$HOME/nextcloud-data
    ```
    Keep this path same as the one mentioned in **/reverse-proxy/.env**.

4. The **compose.yml** file also creates a container **nextcloud-cron** for nextcloud cron specified in the offical documentation: https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/background_jobs_configuration.html?utm_source=chatgpt.com  
For it to take effect:  
a. Open Nextcloud as an admin.  
b. Go to Administration Settings → Basic settings.  
c. Under Background jobs, select Cron.  
5. Create a volume named nextcloud for nextcloud installation.
    ```
    docker volume create nextcloud
    ```
6. Run both the container, use seperate terminal for each to better see their logs:
    ```
    cd /opt/reverse-proxy
    docker compose up

    cd /opt/services
    docker compose up 
    ```
    Wait unitl you see something like:
    >mariadb         | 2026-07-04 10:37:08 0 [Note] mariadbd: ready for connections.  
7. Visit your domain on a web browser and install nextcloud.  
    Note : Domain propagation might take sometime so if it shows bad gatway or something simialr either wait for a while or use host ip in browser instead of domain name.  

    After clicking on install you won't be able to access the url. It requires some further config. Leave the container up for now. 

8. The configuration for nextcloud php config lies inside the nextcloud container at:
    > /var/www/html/config/config.php 
    
    Run:
    ```
    docker network inspect proxy | grep Subnet
    ```

    > "Subnet": "172.18.0.0/16",  

    Get its copy from the container:
    ```
    docker cp nextcloud:/var/www/html/config/config.php .
    nano config.php
    ```
    Add/Modify the following:  
    ```
    array ( 
        0 => 'Your Host IP',
        1 => 'nc.kart1kg.com',
     ),

    # From above "Subnet": "172.18.0.0/16",  
    'trusted_proxies' => array ( 0 => '172.18.0.0/16', ),

    'overwrite.cli.url' => 'https://nc.kart1kg.com',
    'overwriteprotocol' => 'https',
    ```
    *Use your domain above*

    Copy it back to the container and set proper permissions:
    ```
    docker cp config.php nextcloud:/var/www/html/config/config.php

    docker exec nextcloud chown www-data:www-data /var/www/html/config/config.php

    docker exec nextcloud chmod 640 /var/www/html/config/config.php
    ```
    Verify by:
    ```
    docker exec nextcloud ls -l /var/www/html/config/config.php
    ```
    You must get:
    > -rw-r----- 1 www-data www-data 1279 Jul  3 21:32 /var/www/html/config/config.php

9. Server auto maintenance setup: 
    Set its maintenance time such negligible traffic is expected during maintenance. 
    ```
    docker exec -u www-data nextcloud php occ config:system:set maintenance_window_start --type=integer --value=22
    ```
    Value specifies the time in **UTC**, here 22hrs UTC = 3:30 a.m IST

    Then 
    ```
    docker exec -u www-data nextcloud php occ maintenance:repair --include-expensive
    ```
10. Maintenance, run ocasionally and also at the time of setup:
    ```
    docker exec -u www-data nextcloud php occ maintenance:repair --include-expensive
    ```
# PHP Tweeks For Nextcloud
By this time, nextcloud is ready to be used. The purpose of this block is to tweek some of the default setting as per our need. 
These tweeks are specified in:
> /services/php/custom.ini

1. The max file upload size is specified by all of them:

    > /services/php/custom.ini
    ```
    upload_max_filesize = 2G
    post_max_size = 2G
    ```

    >/services/compose.yml
    ```
    PHP_UPLOAD_LIMIT: 2G
    ```

    >/reverse-proxy/nginx/conf.d/nextcloud.conf
    ```
    client_max_body_size 2G;
    ```

    The minimum of all the above values will be used.  
    To verify it, with nextcloud container up,  run:
    ```
    docker exec nextcloud php -i | grep upload_max_filesize
    ```
    You must get:
    > upload_max_filesize => 2G => 2G

# Bring it up baby!
Finally after reading through all of that **you** made it to bringing **Nextcloud** online. Needless to say I too feel happy for reaching this seciton. 

This won't take long just, all thanks to **Docker**. 

To bring the reverse-proxy up, run:
```
cd /opt/reverse-proxy
docker compose up -d
```

For nextcloud, run:
```
cd /opt/services
docker compose up -d
```

**You are good to go!!**

Visit your specified url, here it is: https://nc.example.com

# Redis (Optional)
Redis provides in memory caching our content on nextcloud and can provide speedup. To use it, we'll have to let know nextcloud about it.
1. The configuration for the same lies inside the nextcloud container at:
    > /var/www/html/config/config.php 



2. Run:  
    Get nextcloud php config from container:
    ```
    docker cp nextcloud:/var/www/html/config/config.php .
    nano config.php
    ```
    Add/Modify the following:  
     ```
    'memcache.local' => '\OC\Memcache\APCu',

    'filelocking.enabled' => true,

    'memcache.locking' => '\OC\Memcache\Redis',
    'memcache.distributed' => '\OC\Memcache\Redis',
    'redis' => [
        'host' => 'redis',
        'port' => 6379,
    ],
    ```

    Copy it back to the container and set proper permissions:
    ```
    docker cp config.php nextcloud:/var/www/html/config/config.php

    docker exec nextcloud chown www-data:www-data /var/www/html/config/config.php

    docker exec nextcloud chmod 640 /var/www/html/config/config.php
    ```
    Verify by:
    ```
    docker exec nextcloud ls -l /var/www/html/config/config.php
    ```
    You must get:
    > -rw-r----- 1 www-data www-data 1279 Jul  3 21:32 /var/www/html/config/config.php

# Image Previews For Client (Optional)

Reading it doesn't give the feel of its need but it can quicky become a bottleneck.  
1. Conside a folder **/myimages** containing several images that you uploaded onto your nextcloud. 
The default behaivour of nextcloud is to generate image previews on the fly when you open **/myimages** on the client.
2. For around 50-100 it works fine but with increasing number of images this can eaisly eatup significant compute and choke the server.
3. For this the solution I found is to generate image previews periodically in background and store them on disk.
4. First tweak preview setting int the **nextcloud volume** at:
    >  /var/www/html/config/config.php  

    Get nextcloud php config from container:
    ```
    docker cp nextcloud:/var/www/html/config/config.php .
    nano config.php
    ```
    Add/Modify the following:  
    ```
    'enable_previews' => true,

    'enabledPreviewProviders' => [
        'OC\Preview\PNG',
        'OC\Preview\JPEG',
        'OC\Preview\GIF',
        'OC\Preview\TXT',
        'OC\Preview\MarkDown',
        'OC\Preview\SVG',
        'OC\Preview\HEIC',
    ],

    'preview_max_x' => 1024,
    'preview_max_y' => 1024,
    'jpeg_quality' => 60,
    'preview_max_memory' => 1024,
    ```

    a. You can **turnoff** preview completely by *'enable_previews' => false*  
    b. Specify files for which you want previews in *'enabledPreviewProviders'*  
    c. Specify preview dimensions by *'preview_max_x' => 1024* and *'preview_max_y' => 1024*

    Copy it back to the container and set proper permissions:
    ```
    docker cp config.php nextcloud:/var/www/html/config/config.php

    docker exec nextcloud chown www-data:www-data /var/www/html/config/config.php

    docker exec nextcloud chmod 640 /var/www/html/config/config.php
    ```
    Verify by:
    ```
    docker exec nextcloud ls -l /var/www/html/config/config.php
    ```
    You must get:
    > -rw-r----- 1 www-data www-data 1279 Jul  3 21:32 /var/www/html/config/config.php

5. With **nextcloud container running**, install preview generator:
    ```
    docker exec -it nextcloud php occ app:install previewgenerator
    docker exec -it nextcloud php occ app:enable previewgenerator
    ```

6. Run this to generate previews for all the existing images which don't have a preview yet.
    ```
    docker exec -it nextcloud php occ preview:generate-all
    ``` 
7. Add a cron job on host for background preview generation every 10 minutes. Run:
    ```
    crontab -e
    ```
    Append:
    ```
    */10 * * * * docker exec -u www-data nextcloud php occ preview:pre-generate
    ```