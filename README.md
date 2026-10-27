# Развёртывание виртуального сервера

## Содержание

1. [Установка Linux-сервера](#1-установка-linux-сервера)
2. [Запуск приложения](#2-запуск-приложения)
3. [Контейнер сервлетов Tomcat](#3-контейнер-сервлетов-tomcat)
4. [Проксирование через nginx](#4-проксирование-через-nginx)

---

## 1. Установка Linux-сервера

### 1.1. VirtualBox

1. Скачайте и установите VirtualBox: <https://www.virtualbox.org/wiki/Downloads> (Platform Packages для Windows).
2. Там же скачайте **Extension Pack**, затем в VirtualBox Manager: **Файл → Плагины → «+»** → выберите файл `.vbox-extpack`.

### 1.2. Создание виртуальной машины

1. Скачайте ISO Ubuntu Server: <https://ubuntu.com/download/server>
2. В VirtualBox Manager нажмите **«Создать»**:
   - Имя: `Ubuntu Server`, версия `24.04`
   - ISO: выберите скачанный файл, снимите галочку **«Осуществить установку автоматически»**
   - ОЗУ: 2048 МБ, процессоры: 1
   - Диск: 25 ГБ, **без** галочки «Выделить место в полном размере»
3. Нажмите **«Готово»**.

### 1.3. Установка Ubuntu

Запустите машину и пройдите установщик (только клавиатура):

- Язык и раскладка — на ваш выбор
- Тип установки — **по умолчанию**, сторонние драйверы — **отключить**
- Сеть, прокси, зеркало — **по умолчанию**
- Диск — **«Использовать весь диск»** + **LVM**
- Профиль — введите имя, имя сервера, пользователя и пароль
- Ubuntu Pro — **пропустить**
- SSH — **обязательно установить OpenSSH server**
- Snaps — ничего не выбирать

После установки выберите **«Перезагрузить сейчас»** и извлеките ISO (или нажмите Enter, если опция неактивна).

### 1.4. Первичная настройка

```bash
sudo apt update
sudo apt upgrade
```

Проверьте SSH:

```bash
ssh
```

Если не найдена:

```bash
sudo apt-get install openssh-server
```

### 1.5. Сетевой мост

1. Выключите машину: `sudo poweroff`
2. VirtualBox Manager → **Настроить → Сеть → Адаптер 1**: смените **NAT** на **Сетевой мост**
3. Запустите машину и узнайте IP:

```bash
ip address
```

Запишите адрес (например, `192.168.1.102`).

### 1.6. Управление из консоли

```bash
# Запуск в фоне
VBoxManage startvm "Ubuntu Server 24.04" --type headless

# Выключение
VBoxManage controlvm "Ubuntu Server 24.04" poweroff
```

### 1.7. Подключение по SSH

**Linux/macOS:**

```bash
ssh wil@192.168.1.102
```

**Windows** — через PuTTY: введите IP в поле «Имя хоста» и нажмите «Открыть».

### 1.8. SSH-ключи (вход без пароля)

В PowerShell:

```powershell
ssh-keygen -t ed25519
```

На все вопросы — Enter. Затем скопируйте ключ на сервер:

```powershell
type %USERPROFILE%\.ssh\id_ed25519.pub | ssh vadim@192.168.1.102 "cat >> ~/.ssh/authorized_keys"
```

Проверка (пароль не должен запрашиваться):

```bash
ssh wil@192.168.1.102
```

### 1.9. Проверка интернета

```bash
wget https://github.com/Cheshyuka/3-java-file-manager/blob/main/src/main/webapp/fileList.js
ls -la
rm fileList.jsp
```

---

## 2. Запуск приложения

### 2.1. Установка JDK

```bash
sudo apt-get install default-jdk
java -version
```

### 2.2. Запуск JAR

```bash
java -jar demo-0.0.1-SNAPSHOT.jar
```

Проверка:

```bash
curl http://localhost:8000
```

### 2.3. Копирование на сервер

```bash
scp demo-0.0.1-SNAPSHOT.jar vadim@192.168.1.102:~/projects/demo.jar
```

### 2.4. Запуск в фоне

```bash
nohup java -jar filename.jar >> app1.log &
```

### 2.5. Диагностика

```bash
less app1.log      # лог
ps af              # процессы
netstat -tnpl      # порты
```

Проверка в браузере: `http://192.168.1.102:8000/`

---

## 3. Контейнер сервлетов Tomcat

### 3.1. Подготовка

Соберите приложение из модуля 3 в **WAR-файл**. Создайте пользователя:

```bash
sudo useradd -r -m -U -d /opt/tomcat -s /bin/false tomcat
```

### 3.2. Установка Tomcat 9

```bash
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.117/bin/apache-tomcat-9.0.117.tar.gz -P /tmp
sudo tar xf /tmp/apache-tomcat-9*.tar.gz -C /opt/tomcat
sudo ln -s /opt/tomcat/apache-tomcat-9.0.117 /opt/tomcat/latest
sudo chown -RH tomcat: /opt/tomcat/latest
sudo sh -c 'chmod +x /opt/tomcat/latest/bin/*.sh'
```

### 3.3. systemd-сервис

Создайте `/etc/systemd/system/tomcat.service`:

```ini
[Unit]
Description=Tomcat 9 servlet container
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64"
Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom -Djava.awt.headless=true"
Environment="CATALINA_BASE=/opt/tomcat/latest"
Environment="CATALINA_HOME=/opt/tomcat/latest"
Environment="CATALINA_PID=/opt/tomcat/latest/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/opt/tomcat/latest/bin/startup.sh
ExecStop=/opt/tomcat/latest/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```

Путь к Java узнайте командой `readlink -f $(which java)`.

### 3.4. Запуск

```bash
sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl status tomcat
sudo systemctl enable tomcat
```

### 3.5. Firewall

```bash
sudo ufw allow 8080/tcp
```

### 3.6. Пользователи веб-интерфейса

В `/opt/tomcat/latest/conf/tomcat-users.xml` добавьте:

```xml
<tomcat-users>
    <role rolename="admin-gui"/>
    <role rolename="manager-gui"/>
    <user username="admin" password="admin_password" roles="admin-gui,manager-gui"/>
</tomcat-users>
```

### 3.7. Доступ к Manager/Host Manager

По умолчанию доступ только с localhost. Чтобы открыть с любого IP, в файлах:

```
/opt/tomcat/latest/webapps/manager/META-INF/context.xml
/opt/tomcat/latest/webapps/host-manager/META-INF/context.xml
```

закомментируйте блок `<Valve ... RemoteAddrValve ... />`:

```xml
<Context antiResourceLocking="false" privileged="true">
    <!--
    <Valve className="org.apache.catalina.valves.RemoteAddrValve"
           allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
    -->
</Context>
```

Либо добавьте свой IP в `allow` через `|`.

### 3.8. Перезапуск и деплой

```bash
sudo systemctl restart tomcat
```

Откройте `http://<IP>:8080/manager/html` → секция **«WAR file to deploy»** → загрузите WAR-файл.

---

## 4. Проксирование через nginx

### 4.1. hosts на локальной машине

Откройте от администратора `c:\windows\system32\drivers\etc\hosts` и добавьте:

```
192.168.1.102 example.com
```

### 4.2. Установка nginx

```bash
sudo apt install nginx -y
```

### 4.3. Конфигурация

Создайте `/etc/nginx/sites-available/example.com`:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8080/;
    }

    location /tomcat {
        proxy_pass http://127.0.0.1:8080/;
    }

    location /spring/ {
        proxy_pass http://127.0.0.1:8000/;
    }
}
```

### 4.4. Активация

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl start nginx
sudo systemctl enable nginx
```

### 4.5. Проверка

| Адрес | Куда |
|---|---|
| `http://example.com` | Tomcat (8080) |
| `http://example.com/tomcat` | Tomcat (8080) |
| `http://example.com/spring/` | Приложение (8000) |
