## ownCloud私有云存储

### MySQL
    
    yum install -y gcc gcc-c++ make automake zlib-devel ncurses ncurses-devel bison
    
    tar -zxf cmake-3.11.4.tar.gz
    cd cmake-3.11.4/
    ./bootstrap --prefix=/usr/local/cmake
    gmake && gmake install
    
    mkdir /usr/local/mysql
    mkdir /data/mysql/mysql -p
    
    groupadd mysql
    useradd -r -g mysql -s /sbin/nologin mysql -M
    chown -R mysql:mysql /data/mysql

    tar -zxf mysql-boost-5.7.22.tar.gz
    cd mysql-5.7.22    
    cmake . \
    -DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
    -DMYSQL_DATADIR=/data/mysql/mysql \
    -DSYSCONFDIR=/etc \
    -DMYSQL_TCP_PORT=3306 \
    -DWITH_INNOBASE_STORAGE_ENGINE=1 \
    -DWITH_MYISAM_STORAGE_ENGINE=1 \
    -DWITH_PARTITION_STORAGE_ENGINE=1 \
    -DWITH_FEDERATED_STORAGE_ENGINE=1 \
    -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
    -DWITH_ARCHIVE_STORAGE_ENGINE=1 \
    -DENABLED_LOCAL_INFILE=1 \
    -DWITH_PARTITION_STORAGE_ENGINE=1 \
    -DDEFAULT_CHARSET=utf8 \
    -DEXTRA_CHARSETS=all \
    -DDEFAULT_COLLATION=utf8_general_ci \
    -DWITH_EMBEDDED_SERVER=1 \
    -DWITH_DEBUG=0 \
    -DWITH_BOOST=boost
    
    gmake -j4 && gmake install

    cd /usr/local/mysql/
    cp support-files/mysql.server /etc/init.d/mysqld
    chmod 755 /etc/init.d/mysqld
    chkconfig mysqld on
    systemctl daemon-reload
    
    cat >> /etc/profile.d/mysql.sh << EOF
    export PATH=$PATH:/usr/local/mysql/bin
    export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/mysql/lib
    EOF
    
    mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysql/mysql
    
    mysql_ssl_rsa_setup --datadir=/data/mysql/mysql
    chown -R mysql:mysql /data/mysql
    mkdir /var/lib/mysql
    chown -R mysql:mysql /var/lib/mysql
    mkdir /data/mysql/binlog
    chown -R mysql:mysql /data/mysql/binlog
    mkdir /usr/local/mysql/logs
    chown -R mysql:mysql /usr/local/mysql/logs
    
    mysql_secure_installation

    mysql -uroot -p
    sql> create database cloud;
    sql> grant all privileges on cloud.* to cloud@'localhost' identified by 'xxxxx';
    sql> flush privileges;
    
## Nginx
    
    groupadd wwwroot
    useradd -g wwwroot -s /sbin/nologin wwwroot -M
    yum install -y gcc gcc-c++ make zlib zlib-devel pcre pcre-devel openssl-devel gd gd-devel perl-devel perl-ExtUtils-Embed
    
    ./configure --prefix=/opt/nginx \
    --user=wwwroot \
    --group=wwwroot \
    --prefix=/opt/nginx \
    --with-http_stub_status_module \
    --with-pcre \
    --with-http_ssl_module \
    --with-mail_ssl_module \
    --with-http_gzip_static_module \
    --with-http_degradation_module \
    --with-http_realip_module \
    --with-http_addition_module \
    --with-http_sub_module \
    --with-http_perl_module \
    --with-stream \
    --with-threads \
    --with-http_image_filter_module \
    --with-http_flv_module \
    --add-module=/usr/local/ngx_cache_purge-2.3 \
    --add-module=/usr/local/ngx_http_consistent_hash-master
    make && make install
    
## PHP
    
    yum install gcc gcc-c++ curl curl-devel zlib-devel zlib net-snmp net-snmp-devel libtool libxslt libxslt-devel \
    libpng libpng-devel bzip2 bzip2-devel libxml2-devel libXpm-devel libcurl-devel openssl-devel libmcrypt expat libxslt \
    freetype freetype-devel libmcrypt-devel autoconf libjpeg libjpeg-devel gd-devel bzip2-devel
    
    yum install php-intl php-pecl-json php-pdo libtiff-devel libwebp-devel unixODBC unixODBC-devel icu libicu libicu-devel
    
    tar -zxf libiconv-1.15.tar.gz
    ./configure --prefix=/usr/local/libiconv
    
    tar -zxf jpegsrc.v9c.tar.gz
    mkdir /usr/local/jpeg9 && mkdir /usr/local/jpeg9/{bin,lib,include}
    mkdir /usr/local/jpeg9/man/man1 -p
    cp -rf /usr/share/libtool/config/config.sub . && cp -rf /usr/share/libtool/config/config.guess .
    ./configure --prefix=/usr/local/jpeg9 --enable-shared --enable-static
    make && make install
    
    tar -zxf libzip-1.5.1.tar.gz
    mkdir build
    cd build
    cmake ..
    make
    make test
    make install
    
    export LD_LIBRARY_PATH=/opt/icu/lib
    
    tar -zxf libgd-2.2.5.tar.gz
    ./configure --prefix=/usr/local/libgd2 --with-zlib --with-jpeg=/usr/local/jpeg9 --with-png --with-freetype --with-fontconfig --with-xpm --with-webp --with-tiff
    
    groupadd php-fpm
    useradd -g php-fpm -s /sbin/nologin php-fpm -M
    
    tar -zxf php-7.2.7.tar.gz && cd php-7.2.7
    ./configure \
    --prefix=/opt/php --with-config-file-path=/opt/php/etc \
    --enable-fpm --with-fpm-user=php-fpm --with-fpm-group=php-fpm \
    --with-mysql-sock=/var/lib/mysql/mysql.sock --with-mysqli=/usr/local/mysql/bin/mysql_config \
    --with-jpeg-dir=/usr/local/jpeg9 --with-gd=/usr/local/libgd2 --with-iconv-dir=/usr/local/libiconv \
    --with-openssl-dir --with-freetype-dir --with-libxml-dir --with-png-dir --with-zlib --with-curl \
    --with-mhash --with-pear --with-pcre-dir --with-gettext --enable-soap --enable-mbstring --enable-exif \
    --enable-sockets --enable-ftp --enable-bcmath --enable-shmop --with-snmp --enable-sysvsem --enable-zip \
    --with-bz2 --with-libxml-dir --enable-intl --with-icu-dir=/usr --enable-mbstring --with-libmbfl --with-xmlrpc \
    --with-libzip --enable-mysqlnd --with-zlib-dir --with-pdo-mysql --enable-pcntl --with-unixODBC=/usr \
    --with-dbmaker --with-webp-dir --with-png-dir --with-xpm-dir --with-zlib-dir --with-freetype-dir \
    --enable-gd-jis-conv --with-gettext
    
    cp sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm
    
## ownCloud
    
    tar -jxf owncloud-10.0.8.tar.bz2
    mv owncloud /data/wwwroot/html/
