# Rather secure Wordpress Nginx config

Quickstart
----------
Assumes Debian/Ubuntu /etc/nginx layout, so modify as needed to fit your directory structure.

Clone this repo to /etc/nginx/snippets

**/etc/nginx/snippets/wordpress-nginx/php.conf** just needs the name of the Nginx *upstream* for the desired FPM server

```nginx
# you must define an upstream, probably in /etc/nginx/conf.d/
include snippets/fastcgi-php.conf;
fastcgi_pass php74_backend;
```
for example **/etc/nginx/conf.d/php74_backend.conf**
```nginx
upstream php74_backend {
    server unix:/run/php/php7.4-fpm.sock;
}
```
**/etc/nginx/snippets/wordpress-nginx/admin-whitelist-ips.conf** can optionally list the IPs/subnets allowed Dashboard access
```nginx
allow 127.0.0.1;
# trusted nets
allow 10.1.10.0/24
deny all;
```
or, for no restrictions use
```nginx
allow all;
```

**/etc/nginx/snippets/wordpress-nginx/wordpress.conf**
Permits IP-based restrictions of Wordpress login, theme editor, signup, comments, and cron,
as well as security-minded HTTP headers.
```nginx
### you may want to customize a few common features
# for woo commerce, customers need access to wp-login.php to register/login
location = /wp-login.php {
	include snippets/wordpress-nginx/admin-whitelist-ips.conf;
	include snippets/wordpress-nginx/php.conf;
}
# disable theme editor
location = /wp-admin/network/theme-editor.php {
	return 301 /wp-admin/;
}
location = /wp-signup.php {
	include snippets/wordpress-nginx/admin-whitelist-ips.conf;
	include snippets/wordpress-nginx/php.conf;
}
location = /wp-comments-post.php {
	include snippets/wordpress-nginx/admin-whitelist-ips.conf;
	include snippets/wordpress-nginx/php.conf;
}
# recommended to "define('DISABLE_WP_CRON', 'true');" in wp-config.php and
# run hourly via cron/cli
# "0 * * * *  php  /path/to/wp-cron.php"
location ~ ^/wp-cron.php$ {
        deny all;
        # include snippets/wordpress-nginx/admin-whitelist-ips.conf;
        # include snippets/wordpress-nginx/php.conf;
}
... snip ...
```

**/etc/nginx/snippets/wordpress-nginx/url-exceptions.conf**
Add \.php files you wish to allow access here. 
Unfortunately, some plugins may need exceptions added.

Include wordpress.conf in your Nginx server configs
**/etc/nginx/sites-enabled/foo.example.com**
```nginx
server {
        server_name foo.example.com;
        root /var/www/foo.example.com;
	include snippets/wordpress-nginx/wordpress.conf;
}

Well-behaved plugins and themes will work find. You may find plugins that require you to permit access to their 

```
Benefits
--------
- restrict access to trusted IPs for privileged Wordpress areas
- whitelist of valid Wordpress \*.php URLs eliminates many PHP exploits
- performance and security benefits over attempting such restrictions in a Wordpress plugin

This config patches a potential security hole in the common PHP configuration in both Apache
```apache
<FilesMatch ".+\.ph(ar|p|tml)$">
    SetHandler "proxy:unix:/run/php/php7.3-fpm.sock"
</FilesMatch>
```

and Nginx
```nginx
location ~ \.php$ {
    include fastcgi_params;
    fastcgi_pass php;
}
```
in which web server will execute any .php file it encounters. 
Attackers who gain access will often hide backdoor php files in a site.
