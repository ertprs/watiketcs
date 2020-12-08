# Panduan Instalasi Linux

# Basic production deployment (Ubuntu 18.04 VPS)


All instructions below assumes you are NOT running as root, since it will give an error in puppeteer. So let's start:
You'll need two subdomains forwarding to yours VPS ip to follow these instructions. We'll use myapp.mydomain.com to frontend and api.mydomain.com to backend in the following example.
Update all system packages:
sudo apt update && sudo apt upgrade
Install node and confirm node command is available:
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v
npm -v
Install docker and add you user to docker group:
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
sudo apt update
sudo apt install docker-ce
sudo systemctl status docker
sudo usermod -aG docker ${USER}
su - ${USER}
Create Mysql Database using docker: Note: change MYSQL_DATABASE, MYSQL_PASSWORD, MYSQL_USER and MYSQL_ROOT_PASSWORD.
docker run --name watiketdb -e MYSQL_ROOT_PASSWORD=b4l14rtsys -e MYSQL_DATABASE=watiket -e MYSQL_USER=watiket -e MYSQL_PASSWORD=b4l14rtsys --restart always -p 3306:3306 -d mariadb:latest --character-set-server=utf8mb4 --collation-server=utf8mb4_bin
Clone this repository:
cd ~
git clone https://github.com/myfrizqi/watiket watiket
Url github ini bias bikin sendiri dengan tidak menyertakan password supaya lebih mudah dalam instalasi.
Create backend .env file and fill with details:
cp watiket/backend/.env.example watiket/backend/.env
nano watiket/backend/.env
NODE_ENV=
BACKEND_URL=https://backend.mydomain.com      #USE HTTPS HERE, WE WILL ADD SSL LATTER
FRONTEND_URL=https://www.mydomain.com   #USE HTTPS HERE, WE WILL ADD SSL LATTER, CORS RELATED!
PROXY_PORT=443                            #USE NGINX REVERSE PROXY PORT HERE, WE WILL CONFIGURE IT LATTER
PORT=8080

DB_HOST=localhost
DB_DIALECT=
DB_USER=
DB_PASS=
DB_NAME=

JWT_SECRET=3123123213123
JWT_REFRESH_SECRET=75756756756
Install puppeteer dependencies:
sudo apt-get install -y libgbm-dev wget unzip fontconfig locales gconf-service libasound2 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 libgcc1 libgconf-2-4 libgdk-pixbuf2.0-0 libglib2.0-0 libgtk-3-0 libnspr4 libpango-1.0-0 libpangocairo-1.0-0 libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 ca-certificates fonts-liberation libappindicator1 libnss3 lsb-release xdg-utils
Install backend dependencies, build app, run migrations and seeds:
cd watiket/backend
npm install
npm run build
npx sequelize db:migrate
npx sequelize db:seed:all
Install pm2 with sudo, and start backend with it:
sudo npm install -g pm2
pm2 start dist/server.js --name watiket-backend
Make pm2 auto start afeter reboot:
pm2 startup ubuntu -u –-user ‘root’
Go to frontend folder and install dependencies:
cd ../frontend
npm install
Edit .env file and fill it with your backend address, it should look like this:
cp .env.example .env
nano .env
REACT_APP_BACKEND_URL = https://backend.mydomain.com/
Build frontend app:
npm run build
Start frontend with pm2, and save pm2 process list to start automatically after reboot:
pm2 start server.js --name watiket-frontend
pm2 save
To check if it's running, run pm2 list, it should look like:
deploy@ubuntu-whats:~$ pm2 list
┌─────┬─────────────────────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id  │ name                    │ namespace   │ version │ mode    │ pid      │ uptime │ .    │ status    │ cpu      │ mem      │ user     │ watching │
├─────┼─────────────────────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 1   │ watiket-frontend      │ default     │ 0.1.0   │ fork    │ 179249   │ 12D    │ 0    │ online    │ 0.3%     │ 50.2mb   │ deploy   │ disabled │
│ 6   │ watiket-backend       │ default     │ 1.0.0   │ fork    │ 179253   │ 12D    │ 15   │ online    │ 0.3%     │ 118.5mb  │ deploy   │ disabled │
└─────┴─────────────────────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘
Install nginx:
sudo apt install nginx
Remove nginx default site:
sudo rm /etc/nginx/sites-enabled/default
Create a new nginx site to frontend app:
sudo nano /etc/nginx/sites-available/watiket-frontend
Edit and fill it with this information, changing server_name to yours equivalent to myapp.mydomain.com:

server {
  server_name myapp.mydomain.com;

  location / {
    proxy_pass http://127.0.0.1:3333;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_cache_bypass $http_upgrade;
  }
}
Create another one to backend api, changing server_name to yours equivalent to backend.mydomain.com, and proxy_pass to your localhost backend node server URL:
sudo cp /etc/nginx/sites-available/watiket-frontend /etc/nginx/sites-available/watiket-backend
sudo nano /etc/nginx/sites-available/watiket-backend
server {
  server_name backend.mydomain.com;

  location / {
    proxy_pass http://127.0.0.1:8080;
    ......
}
Create a symbolic links to enalbe nginx sites:
sudo ln -s /etc/nginx/sites-available/watiket-frontend /etc/nginx/sites-enabled
sudo ln -s /etc/nginx/sites-available/watiket-backend /etc/nginx/sites-enabled
By default, nginx limit body size to 1MB, what isn't enough to some media uploads. Lets change it to 20MB adding a new line to config file:
sudo nano /etc/nginx/nginx.conf
...
http {
    ...
    client_max_body_size 20M; # HANDLE BIGGER UPLOADS
}
Test nginx configuration and restart server:
sudo nginx -t
sudo service nginx restart
Now, enable SSL (https) on your sites to use all app features like notifications and sending audio messages. A easy way to this is using Certbot:
Install certbor with snapd:
sudo snap install --classic certbot
Enable SSL on nginx (Accept all information asked):
sudo certbot --nginx

Buka website tiketnya:
https://www.namadomain.com/signup
Buat akun administrator kemudian simpan. Setelah itu silahkan login.



Instalasi Phpmyadmin
To download the latest stable version of the image, open a terminal and type the following:
$ docker pull phpmyadmin/phpmyadmin:latest
After downloading the image, we need to run the container making sure that the container connects with the other container running mysql. In order to do so we type the following command:
$ docker run --name watiket-phpmyadmin -d --link watiketdb:db -p 8989:80 phpmyadmin/phpmyadmin
Let’s explain the options for the command docker run.

Yes, that's all…everything is done! Easy right? You only need to open your favourite browser and type the following url: http://www.namadomain.com:8989/ so your instance of phpMyAdmin will show up.
Username: watiket
Password: b4l14rtsys



# Instalasi Phpmyadmin
$ sudo apt update
$ sudo apt install xfce4 xfce4-goodies
$ sudo apt install tightvncserver
$ vncserver
$ vncserver -kill :1
$ mv ~/.vnc/xstartup ~/.vnc/xstartup.bak
$ nano ~/.vnc/xstartup
(paste the follow being in the xstartup file and write out)

#!/bin/bash
xrdb $HOME/.Xresources
startxfce4 &

$ sudo chmod +x ~/.vnc/xstartup
$ vncserver


# watiket
# watiket
# watiket
