Developer Setup (WIP)
=====================

The setup is currently more complex that it needs to be, this will be improved later.

You need:

* Docker-Compose
* an nginx installation (that will proxy the docker nginx that proxies another docker nginx...)

# Modify nextcloud-docker/docker-compose.yml

First, check what IP address docker chose for its internal network.

$ ip addr

And look for an entry for "br-xxxxxxxxx" where xxx is a random string, or "docker0"
it will look something like this:

```
1067: br-cf89f830787c: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:50:77:1e:52 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global br-cf89f830787c
```

The 172.17.0.1 is the important part (it can also be 172.18, 19, or 20)
(You might see a "docker0" link with an IP of 192.168.x.x, you can ignore that one)

Now, edit the line "wechange-dev:172.17.0.1" in the extra_hosts section to the 172.x.0.1 IP your Docker has.

# Edit /etc/hosts

add an entry

```
127.0.0.1 wechange-dev
```

# create a nginx proxy site

install nginx on your OS, this guide assumes you use a debia based distro.

in /etc/nginx/sites-available, add a new file "wechange" (name is not important) with the following contents:

```
server {
    listen 80;
    server_name wechange-dev;
    access_log /tmp/nginx.log;

    location /nextcloud/ {
        proxy_pass http://127.0.0.1:8080/;
        proxy_set_header Host $host;
        #proxy_set_header X-Real-IP $remote_addr;
        #proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        proxy_pass http://127.0.0.1:8000/;
        proxy_set_header Host $host;
        proxy_redirect off;
    }
}
```

then go to /etc/nginx/sites-enabled and create a symlink

ln -s ../sites-available/wechange

then restart nginx.

# Start nextcloud

cd to nextcloud-docker and execute

$ docker-compose up -d

you can also omit the `-d` if you want to see the log output without having to execute docker-compose logs

collabora will spam a lot of text about creating symlinks at startup, that is normal

# Update the config.php file in the container

$ docker-compose logs -f app

Now wait until you see "Nextcloud was sucessfully installed"

```
Attaching to nextcloud-docker_app_1
app_1        | Initializing nextcloud 17.0.1.1 ...
app_1        | Initializing finished
app_1        | New nextcloud instance
app_1        | Installing with PostgreSQL database
app_1        | starting nextcloud installation
app_1        | Nextcloud was successfully installed
app_1        | setting trusted domains…
app_1        | System config value trusted_domains => 1 set to string wechange-dev
app_1        | [18-Dec-2019 16:26:45] NOTICE: fpm is running, pid 1
app_1        | [18-Dec-2019 16:26:45] NOTICE: ready to handle connections
```

press Ctrl+C

do 

$ docker-compose exec app su www-data -s /bin/sh -c './occ config:system:set overwritewebroot --value /nextcloud'

(make sure that the container is running)

You should now be able to visit http://wechange-dev/nextcloud and see the login screen

# Configure oauth2 application in cosinnus

At the time of writing, the oauth2_provider application config interface is broken, so we have to add the data manually. Connect to your WECHANGE Postgres database (not the postgres used in docker-compose; that one is used by Nextcloud), and execute:

```
insert into oauth2_provider_application (client_id, client_secret, client_type, redirect_uris, authorization_grant_type, name, skip_authorization, created, updated) values ('foobar', 'barfoo', 'public', 'http://wechange-dev/nextcloud/apps/sociallogin/custom_oauth2/wechange', 'authorization-code', 'nextcloud', true, now(), now());
```

Obviously, "foobar"/"barfoo" should not be used in production.

# Add necessary modules

Go to http://wechange-dev/nextcloud and log in as "admin", password "admin"

Then visit http://wechange-dev/nextcloud/settings/apps/integration and install the "Social Login" app

Then go to http://wechange-dev/nextcloud/settings/admin/sociallogin

Activate "update user profile at login", and add a new Custom OAuth Server (not Custom OpenID Connect)

* Internal name: wechange
* Name: wechange
* API-Base: http://wechange-dev/o
* Authorize-URL: http://wechange-dev/o/authorize/  (contrary to what the UI claims, the url cannot be relative)
* Token-URL: http://wechange-dev/o/token
* Profile-URL: http://wechange-dev/group/forum/cloud/oauth2/
* Logout-URL: (empty)
* Client ID: foobar
* Client Secret: barfoo
* Scope: read

Then save

Now go to http://wechange-dev/nextcloud/settings/users and add a new group named "wechange-Forum"

Go to http://wechange-dev/nextcloud/apps/files/ and create a new folder in the root directory and name it "Groupfolders"


# Add settings to settings.py

* Add "wechange-dev" to ALLOWED_HOSTS

Add
```
COSINNUS_CLOUD_NEXTCLOUD_URL = "http://wechange-dev/nextcloud"
COSINNUS_CLOUD_NEXTCLOUD_ADMIN_USERNAME = "admin"
COSINNUS_CLOUD_NEXTCLOUD_GROUPFOLDER_BASE = "Groupfolders"
COSINNUS_CLOUD_NEXTCLOUD_AUTH = ("admin", "admin")
```


Change domainname in http://localhost/admin/sites/site to wechange-dev

# You're set

You should be able to visit http://wechange-dev/group/forum/cloud and log in by clicking on "login with wechange". If you get an
internal error that wechange-dev cannot be reached, the docker ip might have changed. Do `ip addr`  again and edit the "wechange-dev:172.x.x.x" entry in the docker-compose.yml file, then do docker-compose restart
