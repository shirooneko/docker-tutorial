# docker-tutorial  
## instalasi docker di ubuntu server  
Install Dependensi untuk Menggunakan Repository HTTPS :
```
sudo apt install apt-transport-https ca-certificates curl software-properties-common
```
Tambahkan GPG Key Docker Repository :
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
Tambahkan Docker Repository ke daftar repository APT :
```
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
Setelah itu baru Install Docker :
```
sudo apt install docker-ce docker-ce-cli containerd.io
```
Aktfikan docker :
```
sudo systemctl start docker
```
Enable kan docker :
```
sudo systemctl enable docker
```
periksa apakah docker sudah terinstall atau belum :
```
sudo docker --version
```
Tambahkan pengguna ke dalam docker :
```
sudo usermod -aG docker shiroo
```
ganti `shiroo` dengan nama yang sesuai dengan ubuntu server anda, untuk memudahkan  

## Instalasi mysql didalam docker  
Pull Image MySQL dari docker hub :
```
sudo docker pull mysql:latest
```
Buat Container MySQL :
```
sudo docker run --name mysql-container -e MYSQL_ROOT_PASSWORD=root -d -p 3306:3306 mysql:latest
```
--name mysql-container: Memberi nama container.  
-e MYSQL_ROOT_PASSWORD=root: Mengatur kata sandi root MySQL.    
-d: Menjalankan container dalam mode latar belakang (daemon).  
-p 3306:3306: Mengarahkan port 3306 dari server Anda ke port 3306 dalam container MySQL. Ini adalah port default MySQL.  
periksa Instalasi MySQL :
```
sudo docker ps
```
![docker-tutorial](image/1.jpg)
masuk ke mysql yang ada di docker :
```
sudo docker exec -it mysql-container mysql -u root -p
```
masukan password yang tadi sudah di atur. jika sudah masuk maka tampilanya akan menjadi seperti ini :
![docker-tutorial](image/2.jpg)
## instalasi phpmyadmin di docker
pull image phpmyadmin dari docker hub :
```
sudo docker pull phpmyadmin/phpmyadmin
```
buat container phpmyadmin :
```
docker run -d --name phpmyadmin-container -e PMA_HOST=192.168.0.103 -e PMA_PORT=3306 -p 8080:80 phpmyadmin/phpmyadmin
```
`PMA_HOST=192.168.0.103` masukan ip dari ubuntu server anda. setalah itu coba jalankan `192.168.0.103:8080`

## Install nginx dan php fpm di  docker

Sebelum install nginx buatlah directory yang berfungsi untuk menghubungkan directory pada 2 container yaitu nginx dan php fpm nya
Buat directory dengan nama bebas
```
mkdir ci4nginx
```
masuk ke directory
```
cd ci4nginx
```
### Install nginx
pull image nginx 
```
sudo docker pull nginx:latest
```

buat container nginx
```
sudo docker run -d -p 8081:80 --name nginx-container -v $(pwd):/var/www nginx
```
### Install php fpm
buat Dockerfile
```
sudo nano Dockerfile
```
lalu pastekan kode berikut untuk menginstall php fpm
```
FROM php:8.0-fpm

# Ganti server repositori Debian ke server repositori resmi
RUN sed -i 's/http:\/\/deb.debian.org\/debian/http:\/\/deb.debian.org\/debian/' /etc/apt/sources.list

# Tambahkan paket gnupg
RUN apt-get update && apt-get install -y --no-install-recommends gpg

# Install system dependencies
RUN apt-get update && apt-get install -y git libicu-dev

# Install PHP extensions
RUN docker-php-ext-install pdo_mysql intl mysqli

# Get latest Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www

# Customize FPM configuration to use default port 9000
RUN sed -i 's/;listen = 127.0.0.1:9000/listen = 0.0.0.0:9000/' /usr/local/etc/php-fpm.d/www.conf

# Expose the FPM port
EXPOSE 9000

# CMD to start the PHP FPM server
CMD ["php-fpm"]
```
setelah itu ketikan perintah build dengan nama yang diinginkan
```
docker build -t fpm:latest .
```
Setelah itu buat container fpm dengan image yang telah kita buat tadi. Pastikan kita masih ada directory ci4nginx karna itu untuk menghubungkan directory kedua container
```
 docker run -d -p 9000:9000 --name fpm-container -v $(pwd):/var/www fpm:latest
```
Kemudian cek ip fpm-container
```
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' fpm-container
```

### KONFIGURASI NGINX
Copy file default.conf ke dalam directory host
```
docker cp nginx-container:/etc/nginx/conf.d/default.conf /home/shiroo/ci4nginx/default.conf
```
Lalu edit file nya menjadi seperti ini
```
# Tidak ada konfigurasi untuk menolak akses ke file .htaccess
# location ~ /\.ht {
#     deny  all;
# }

server {
    listen 80;
    listen [::]:80;
    server_name localhost;

client_max_body_size 20M;

    # Location for static files
    location / {
        root /var/www;  # Sesuaikan dengan direktori aplikasi Anda
        index index.php index.html index.htm;
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~* \.(js|css|png|jpeg|gif|ico|html)$ {
        root /var/www;  # Sesuaikan dengan direktori aplikasi Anda
        try_files $uri $uri/ =404;
    }

    # pass the PHP scripts to FastCGI server listening on fpm-container:9000
    location ~ \.php$ {
        root /var/www;  # Sesuaikan dengan direktori aplikasi Anda
        fastcgi_pass 172.17.0.5:9000;  # Sesuaikan dengan nama container dan port PHP FPM
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Error pages
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```
Setelah itu copy lagi file nya kedalam container
```
docker cp /home/shiroo/ci4nginx/default.conf nginx-container:/etc/nginx/conf.d/default.conf
```
Setelah itu reload container
```
docker exec -it nginx-container nginx -s reload
```
Buat file php untuk mengetest apakah sudah berjalan atau belum konfigurasi kita
```
sudo nano index.php
```
Isikan test php
```
<?php
phpinfo();
?>
```
Setelah itu buka di web browser 192.168.0.103:8081/




