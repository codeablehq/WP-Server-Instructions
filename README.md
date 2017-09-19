## 1. SSH onto the droplet
ssh root@[ip of the droplet]

## 2. Upgrade the new instance
Since most new instances are created from an image, the software in the image is likely behind so the first thing to do is to update it all.
`$ apt-get update`

`$ apt-get upgrade` (confirm if asked)

## 3. Create swap
This step is important so that if any of our programs runs out of memory, it can use swap. Not creating it might lead to all kinds of problems with system instability. Although [DigitalOcean tutorials](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-16-04) recommends _not_ using swap on SSD, it is still useful as a protection. It should be always empty, so if it is not, strongly consider increasing the Droplet size.

```
$ fallocate -l 1G /swapfile
$ chmod 0600 /swapfile
$ mkswap /swapfile
$ swapon /swapfile
$ echo "/swapfile none swap sw 0 0" >> /etc/fstab
$ cat /proc/meminfo 
```
(last command optional, to see whether swap is active)

## 4. Install basic components the system will need
`$ apt-get install -y git zsh curl apt-transport-https`

## 5. Install Oh-my-zsh (optional step)
Oh-my-zsh (and z-shell in general) come with a few niceties that Bash doesn't provide.
`$ sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"`

`$ vim ~/.zshrc` and change the `theme` line to what you want, _blinks_ is nice.
Reload the shell:
`$ source ~/.zshrc`

If you want a nicer Vim experience, copy `.vimrc` in this repo to the root's home directory.

## 6. Install Nginx
First, let's check what version is the latest in the _apt_ repository:
`$ apt-cache show nginx` (it is most likely behind)
Visit [http://nginx.org/en/linux_packages.html](Nginx repositories page) to see how to add the repository to our apt.

```
$ cat /etc/*-release 
$ echo "deb https://nginx.org/packages/debian/ stretch nginx" > \
  /etc/apt/sources.list.d/nginx.list
$ curl -vs https://nginx.org/keys/nginx_signing.key | apt-key add -
$ apt-get update && apt-get install -y nginx
$ service nginx start
$ ifconfig 
```
The last command will give you the ip for the interface _eth0_
Open it in browser, and voila!

## 7. Install PHP
Just like with Nginx, we need to create two files: the _apt_ source file, which points to the repository and key file, which is used to verify the integrity of the repository:

```
$ echo "deb https://packages.sury.org/php/ stretch main" > \
  /etc/apt/sources.list.d/php.list
$ curl -o /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
```

```
Now we can install PHP and all the dependencies WordPress needs to run:
$ apt-get update && apt-get install -y \
    imagemagick \
    php7.1-fpm php7.1-mysqli php7.1-curl php7.1-gd php7.1-geoip php7.1-xml \
    php7.1-xmlrpc php7.1-imagick php7.1-mbstring php7.1-ssh2 php7.1-redis
```

## 8. Update Nginx to work with PHP
First, edit the default configuration:
`$ vim /etc/nginx/nginx.conf`

Here, change `user www-data;` and `worker_processes auto;`

`$ vim /etc/nginx/conf.d/default.conf`

Here, change `server_name _;`, then in `location /` block, add `index.php` so that the PHP files are read by Nginx. Then uncomment the `location ~ \.php$` block and change `fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;` 

Also, in this block, change the `root /usr/share/nginx/html`.

First check whether Nginx configuration is in order, then reload it:
```
$ service nginx configtest
$ service nginx reload
```

To test whether PHP works, make an `index.php` in `/usr/share/nginx/html`

And run the following command to put `phpinfo()` into it:
`$ echo "<?php phpinfo();" >> /usr/share/nginx/html/index.php`

Visit the IP in the browser, only to encounter an error.

`$ cat /var/log/nginx/error.log` to see what the error is. Expectedly, it can't connect to PHP-FPM, so we need to configure it properly.

```
$ cd /etc/php/7.1/fpm
$ grep -r "listen" . 
$ vim ./pool.d/www.conf
```

Around line 36, there should be a `listen` directive, which we need to replace with `listen = [::]:9000`.

Now reload PHP-FPM: `$ service php7.1-fpm reload`

Visit your browser again, and the `phpinfo();` should be revealed.
Afterwards, **remove it!!!**

## 9. Set up outgoing emails
Make sure exim4 is installed: 
`$ apt-get install -y exim4`

Now, you need to set up an account with one of the mail providers, such as SendGrid, Mailgun or Mandrill. What you need is an API key.

`$ vim /etc/exim4/passwd.client` and put it in in the format shown in the file

`$ vim /etc/exim4/update-exim4.conf.conf` and put the following snippet in (your configuration for _dc_smarthost_ might be different, depending on the provider):

```
dc_eximconfig_configtype='smarthost'
dc_other_hostnames=''
dc_local_interfaces='127.0.0.1'
dc_readhost='example.com'
dc_relay_domains=''
dc_minimaldns='false'
dc_relay_nets=''
dc_smarthost='smtp.sendgrid.net::2525'
CFILEMODE='644'
dc_use_split_config='false'
dc_hide_mailname='true'
dc_mailname_in_oh='true'
```

`$ /usr/sbin/update-exim4.conf` to update EXIM4 configuration

To test whether it's working correctly, run
`$ echo "This is message body" | mail -s "This is Subject" your-email@example.com`

## 10. Set up Redis
`$ apt-get install -y redis-server`
This will install the default Redis from the Debian repository, which usually doesn't have the latest version, so alternatively, you might want to visit [DigitalOcean's Redis tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-redis-on-ubuntu-16-04) and follow it.

## 11. Set up MySQL
Install Percona MySQL as per instructions on [their page](https://www.percona.com/doc/percona-server/LATEST/installation/apt_repo.html)

```
$ wget https://repo.percona.com/apt/percona-release_0.1-4.$(lsb_release -sc)_all.deb
$ dpkg -i percona-release_0.1-4.$(lsb_release -sc)_all.deb
$ cat /etc/apt/sources.list.d/percona-release.list
$ apt-get update
$ apt-get install -y percona-server-server-5.7
Choose a secure password!
$ service mysql status to verify our MySQL server is up and running
$ mysql -u root mysql -p
Inside MySQL command line, create the database and a corresponding user:
> CREATE DATABASE wordpress CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
> GRANT ALL ON wordpress.* TO wordpress_user@'%' IDENTIFIED BY 'wordpress';
> exit;
```

With all the dependencies installed and configured, it's time to...

## 12. Install WordPress
First create a directory that will hold the WordPress core for your domain:
`$ mkdir -p /var/www/example.com` (change with your domain)

Then, navigate to the [Release archive](https://wordpress.org/download/release-archive/) and right click latest _tar.gz_ and choose _copy link address_. 

```
$ cd /var/www/example.com
$ wget https://wordpress.org/wordpress-4.8.1.tar.gz
$ tar -zxf wordpress-4.8.1.tar.gz
$ mv wordpress/* .
$ rm -rf wordpress wordpress-4.8.1.tar.gz
```

Add a user which will be the owner of WordPress files
`$ adduser --ingroup www-data webmaster`

Visit the [WordPress Codex](https://codex.wordpress.org/Hardening_WordPress) to check recommended permissions on directories and files
```
$ find . -type d -exec chmod 755 {} \;
$ find . -type f -exec chmod 644 {} \;
$ cp wp-config-sample.php wp-config.php
$ chmod 660 wp-config.php
$ chown -R webmaster:www-data .
```

Next, disable cron:
```
$ su webmaster
$ crontab -e 
```

and put the following code at the end of the file:

`*/15 * * * * cd /var/www/example.com; php wp-cron.php > /dev/null 2>&1`

Save and close the file, then:

`$ exit` to return back to _root_
`$ vim wp-config.php` to add `define('DISABLE_WP_CRON', true);` and configure database credentials and [salts](https://api.wordpress.org/secret-key/1.1/salt/)

Next, configure, Nginx to serve WordPress correctly, on our chosen domain
```
$ cd /etc/nginx/conf.d
$ vim example.com.conf 
```
(virtual hosts need `.conf` extension to be loaded)

Paste the the contents of the [basic Nginx config](nginx/basic.conf) into this file, changes lines [2](nginx/basic.conf#L2) and [7](nginx/basic.conf#L7) to reflect your own domain and reload Nginx: 
`$ service nginx reload`

Visit the domain in your browser to start the WordPress wizard.
Create a new account and log into WordPress to start installing.

Visit the plugins page and search for "redis cache" and try to install "Redis Object Cache". It will fail, so we need to configure access.

In console/terminal, do the following:

```
$ su webmaster
$ cd
$ ssh-keygen -t rsa -b 4096 (name it wp_rsa)
$ chmod 0640 wp_rsa*
$ mkdir .ssh
$ cp wp_rsa.pub .ssh/authorized_keys
$ chmod 0644 .ssh/authorized_keys
```

`$ vim .ssh/authorized_keys` and prepend `from="127.0.0.1"`

Finally:
```
$ chmod 0700 .ssh
$ exit
```

Next, add the following lines to `wp-config.php`:

```
define('FTP_PUBKEY','/home/webmaster/wp_rsa.pub');
define('FTP_PRIKEY','/home/webmaster/wp_rsa');
define('FTP_USER','webmaster');
define('FTP_PASS','');
define('FTP_HOST','127.0.0.1:22');
define('FS_METHOD', 'ssh2');
```

Now go back to WordPress and install "Redis Object Cache", and activate it.

---

Before continuing, create a `dhparam.pem` file, which will be needed at a later stage, but it takes a while to complete, so we might as well do it now:
First, create a new _screen_:

```
$ screen -R dhparam
$ cd /etc/nginx
$ openssl dhparam -out dhparam.pem 4096 
```

then and press CTRL+A+D to move this screen into background, while we continue with optimizing WordPress.
To check screen:

```
$ screen -ls
$ screen -r dhparam // to go back to the screen
$ exit
```
---

Next, create the uploads directory:

```
$ cd /var/www/example.com/wp-content
$ mkdir uploads
$ chown webmaster:www-data uploads
$ chmod -R 775 ./uploads
```

Now it's time to install WP-Rocket, one of the best caching plugins on the market. Navigate to [WP-Rocket](https://wp-rocket.me/), make an account, buy the plugin and download it. When you try to install it by uploading the file to WordPress, you will most likely get an error `413 Request Entity Too Large`, so fix that next. Go back to the terminal and run the following commands:

`$ vim /etc/php/7.1/fpm/php.ini` and locate the following three directives and set them to whatever you want your max upload size to be:
```
upload_max_filesize = 20M
post_max_size = 20M
memory_limit = 256M 
```
(the last is not for uploads, but let's increase it while we're at it)

Reload PHP-FPM with `$ service php7.1-fpm reload` and return to WordPress to try installing WP-Rocket again. Upon activation it will complain about a couple of files not being in place and it's unable to write to them, due to our limited permissions. That's why we need to manually create those files.

Run these:

```
$ cd /var/www/example.com/wp-content
$ vim advanced-cache.php // paste the WP-Rocket config in
$ mkdir -p wp-rocket-config cache/wp-rocket cache/min cache/busting
$ chown -R webmaster:www-data wp-rocket-config cache/wp-rocket cache/min cache/busting
$ chmod -R g+w wp-rocket-config cache/wp-rocket cache/min cache/busting
```

Reload WP-Rocket config page, and the errors should go away. Open a new incognito browser with your website and inspect the source of the page to verify WP-Rocket it installed (there should be a comment at the very bottom).

## 13. Advanced Nginx + WP-Rocket configuration

We could have stopped here, and would still have a pretty fast WordPress website, but let's go the extra mile.

We will use [this repository](https://github.com/maximejobin/rocket-nginx) to help us configure our Nginx. Let's follow its instructions:

```
$ cd /etc/nginx
$ git clone https://github.com/maximejobin/rocket-nginx.git
$ cd rocket-nginx
$ vim rocket-nginx.ini
$ vim ../conf.d/example.com.conf // (replace with your domain)
Add include rocket-nginx/default.conf; in the server block
$ service nginx configtest
$ service nginx reload
```

If you want to test that it's indeed working, edit `rocket-nginx.ini` and turn debug on (on line 13).

## 14. Set up HTTPS

We will use Letsencrypt, so we need an _acme_ client, for which we will use [Dehydrated](https://github.com/lukas2511/dehydrated).

```
$ mkdir -p /var/www/acme-challenge/ /etc/dehydrated 
$ cd /usr/src
$ git clone https://github.com/lukas2511/dehydrated
$ cd dehydrated
```

Now we need to check what our $PATH directories are (so we don't need to call the full path every time)

```
echo $PATH
$ ln -s /usr/src/dehydrated/dehydrated /usr/local/bin/dehydrated
```

Copy all the needed files into the proper location (`/etc/`):
```
$ cd ./docs/examples
$ cp * /etc/dehydrated
$ cd /etc/dehydrated

$ vim hook.sh // line 55: service nginx reload
$ vim domains.txt // line 1: your-domain.com
$ vim config
```

Visit the [staging documentation](https://letsencrypt.org/docs/staging-environment/) to first try out our certificate registration process there.

Fill out the relevant lines like this:

```
20: CA="https://acme-staging.api.letsencrypt.org/directory"
23: CA_TERMS="https://acme-staging.api.letsencrypt.org/terms"
50: WELLKNOWN="/var/www/acme-challenge"
92: CONTACT_EMAIL="your-email@example.com"
```

Now try the registration:

`$ dehydrated` and then `$ dehydrated -c`, which will return a warning to accept the terms:

`$ dehydrated --register --accept-terms`

`$ dehydrated -c` which will now succeed, so it's time to switch to production values. Open `/etc/dehydrated/config` and replace the staging URLs with production ones. Then run:

`$ dehydrated --register --accept-terms`

`$ dehydrated -c -x` (`-x` means _force_ because we have our staging certificates in place, and they need to be overriden. You only have to do this once).

Finally, configure cron to run the certificate renewal script once a week:
`$ crontab -e` (as root!)

Put the following at the end of the file:
`0 0 * * 0 /usr/local/bin/dehydrated -c`

Now with our certificates in place, replace the `example.com.conf` with `with-ssl.conf` and modify the values to reflect your domain, enable `rocket-nginx` and restart Nginx:
`$ service nginx reload`

(If you want different SSL/TLS configuration, Mozilla has [a wizard](https://mozilla.github.io/server-side-tls/ssl-config-generator/) to generate it for you)

Finally, go to WordPress dashboard, general settings, and fix the domains to use HTTPS, rather than just HTTP.

All done!