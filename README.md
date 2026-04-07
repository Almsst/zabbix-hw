# Домашнее задание: Установка Zabbix Server
# Студент: Тугушев Данила Федорович

## Задание 1

Установлен Zabbix Server с веб-интерфейсом на Ubuntu 24.04.

### Скриншот авторизации в админке

<img width="1548" height="758" alt="image" src="https://github.com/user-attachments/assets/096a8982-d3a4-4dba-8e84-f9c1c5029105" />


### Использованные команды

```bash
# 1. Обновление системы и установка PostgreSQL
sudo apt update && sudo apt upgrade -y
sudo apt install postgresql postgresql-contrib -y

# 2. Добавление репозитория Zabbix 7.2
wget https://repo.zabbix.com/zabbix/7.2/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.2+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.2+ubuntu24.04_all.deb
sudo apt update

# 3. Установка Zabbix Server
sudo apt install zabbix-server-pgsql zabbix-frontend-php php8.3-pgsql zabbix-nginx-conf zabbix-sql-scripts zabbix-agent -y

# 4. Создание базы данных
sudo -u postgres createuser --pwprompt zabbix
sudo -u postgres createdb -O zabbix zabbix

# 5. Импорт схемы
sudo zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix

# 6. Настройка пароля в конфиге
sudo sed -i 's/# DBPassword=/DBPassword=ВАШ_ПАРОЛЬ/' /etc/zabbix/zabbix_server.conf

# 7. Настройка Nginx (порт 8080)
sudo tee /etc/nginx/conf.d/zabbix.conf << 'EOF'
server {
    listen 8080;
    listen [::]:8080;
    root /usr/share/zabbix/ui;
    index index.php;
    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
EOF

# 8. Настройка PHP
sudo sed -i 's/post_max_size = 8M/post_max_size = 16M/' /etc/php/8.3/fpm/php.ini
sudo sed -i 's/max_execution_time = 30/max_execution_time = 300/' /etc/php/8.3/fpm/php.ini
sudo sed -i 's/max_input_time = 60/max_input_time = 300/' /etc/php/8.3/fpm/php.ini

# 9. Запуск служб
sudo systemctl restart php8.3-fpm nginx zabbix-server zabbix-agent
sudo systemctl enable zabbix-server zabbix-agent nginx php8.3-fpm

## Задание 2. Установка Zabbix Agent на два хоста

### 1. Configuration > Hosts (Узлы сети)
<img width="1550" height="91" alt="image" src="https://github.com/user-attachments/assets/6f9c4eee-ee3c-4854-8185-2b33faa16d7f" />


### 2. Лог Zabbix Agent
<img width="1297" height="465" alt="image" src="https://github.com/user-attachments/assets/50f5cc2d-7a6f-4953-a306-3a5bc95918f7" />


### 3. Monitoring > Latest data (Последние данные)
<img width="728" height="731" alt="image" src="https://github.com/user-attachments/assets/7902b381-e610-4e2c-99dd-aab46343e212" />


### Использованные команды

bash
# Установка первого агента
sudo apt install zabbix-agent -y

# Настройка первого агента (порт 10050)
sudo tee /etc/zabbix/zabbix_agentd.conf << 'EOF'
Server=127.0.0.1
ServerActive=127.0.0.1
Hostname=Zabbix-Server
ListenPort=10050
LogFile=/var/log/zabbix/zabbix_agentd.log
LogFileSize=0
PidFile=/var/run/zabbix/zabbix_agentd.pid
EOF

# Настройка второго агента (порт 10051)
sudo cp /usr/sbin/zabbix_agentd /usr/sbin/zabbix_agentd2
sudo tee /etc/zabbix/zabbix_agentd2.conf << 'EOF'
Server=127.0.0.1
ServerActive=127.0.0.1
Hostname=Ubuntu-Agent-2
ListenPort=10051
LogFile=/var/log/zabbix/zabbix_agentd2.log
LogFileSize=0
PidFile=/var/run/zabbix/zabbix_agentd2.pid
EOF

# Создание systemd сервиса для второго агента
sudo tee /etc/systemd/system/zabbix-agent2.service << 'EOF'
[Unit]
Description=Zabbix Agent 2
After=network.target
[Service]
Type=forking
PIDFile=/var/run/zabbix/zabbix_agentd2.pid
ExecStart=/usr/sbin/zabbix_agentd2 -c /etc/zabbix/zabbix_agentd2.conf
ExecStop=/bin/kill $MAINPID
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF

# Запуск агентов
sudo systemctl daemon-reload
sudo systemctl restart zabbix-agent
sudo systemctl start zabbix-agent2
sudo systemctl enable zabbix-agent zabbix-agent2
