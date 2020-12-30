Install LEMP stack to run Wordpress


yum install epel-release -y

yum update -y

firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload

sed -i s/^SELINUX=.*$/SELINUX=disabled/ /etc/selinux/config
setenforce 0 && sestatus

yum install nginx -y
systemctl enable --now nginx



yum install mariadb-server mariadb -y

systemctl enable --now mariadb

yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
yum install yum-utils -y

yum --disablerepo="*" --enablerepo="remi-safe" list php[7-9][0-9].x86_64

yum-config-manager --enable remi-php74

yum install -y php php-mysqlnd php-cli php-fpm php-mysql php-json php-opcache php-mbstring php-xml php-gd php-curl

php --version

vi /etc/php-fpm.d/www.conf
…
; RPM: apache user chosen to provide access to the same directories as httpd
user = nginx
; RPM: Keep a group allowed to write in log dir.
group = nginx
…

listen = /var/run/php-fpm/php-fpm.sock;

listen.owner = nginx
listen.group = nginx
listen.mode = 0660

systemctl enable --now php-fpm


yum install wget rsync -y
wget http://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz

sudo rsync -avP ~/wordpress/  /usr/share/nginx/mysite/

chown -R nginx. /usr/share/nginx/mysite/


mysql_secure_installation

mysql -u root
CREATE DATABASE mysite_db;
CREATE USER mysite@localhost IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON mysite_db.* TO mysite@localhost IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
exit



vi /etc/nginx/conf.d/mysite.conf
server {
    listen       80;
    server_name  server_domain_or_IP;

    root   /usr/share/nginx/mysite;
    index index.php index.html index.htm;

    location / {
        #try_files $uri $uri/ =404;
        try_files $uri $uri/ /index.php?q=$uri&$args; 
    }
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;

    location = /50x.html {
        root /usr/share/nginx/mysite;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    # set client body size to 50M
    client_max_body_size 50M;
}


vi /etc/php.ini
upload_max_filesize = 50M
post_max_size = 50M
max_execution_time = 300

vi /usr/share/nginx/mysite/wp-config.php
/* That's all, stop editing! Happy publishing. */
define('WP_MEMORY_LIMIT', '96M');
