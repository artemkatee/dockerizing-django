# Развертывание веб-приложения на сервере

#### Докеризация веб-приложения(nginx + uvicorn + django + postgresql + certbot) производилась на Ubuntu 24.04.4
![1231](https://github.com/artemkatee/dockerizing-django/assets/38987669/4928e695-a025-43bc-8a45-2f4df4689a25)

## 1. Подготовка

#### 1. Установим docker:
```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt-cache policy docker-ce   #Для проверки
sudo apt install docker-ce
sudo systemctl status docker #Для проверки
```
#### к п.1 Если ошибка (Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d):
```bash
cp /etc/apt/trusted.gpg /etc/apt/trusted.gpg.d
```

#### 2. Установим docker-compose:
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.26.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version #Для проверки
```

#### 3. Установим `python3.8-venv`(модуль python для создания виртуального окружения):
```bash
apt-get install vim
apt install python3.10-venv
```


#### 4. Создаем каталог с веб-приложением:
    mkdir /code

#### 5. Создаем каталоги для хранения постоянных данных:
    mkdir /code/persistentdata
    mkdir /code/persistentdata/static
    mkdir /code/persistentdata/media
    mkdir /code/persistentdata/db
    mkdir /code/persistentdata/certbot
    mkdir /code/persistentdata/certbot/www
    mkdir /code/persistentdata/certbot/conf
    mkdir /code/persistentdata/certbot/conf/live/
    mkdir /code/persistentdata/certbot/conf/live/animek.ru/ 

#### 6. Создаем каталоги с фреймворком(django), Базой данных(postgres) и веб-сервером(nginx):
    mkdir /code/django
    mkdir /code/postgresql
    mkdir /code/nginx

#### 7. Создаем Dockerfile в папке django:
    vi /code/django/Dockerfile
 
#### 8. Добавляем запись в `Dockerfile`:
```Dockerfile
FROM python:3.10
WORKDIR /code
RUN apt-get update -y
RUN apt-get upgrade -y
COPY ./requirements.txt ./
RUN pip install --upgrade pip
RUN pip install -r requirements.txt
COPY ./app ./app
CMD ["uvicorn", "--app-dir", "./app", "app.asgi:application", "--lifespan=off", "--host", "0.0.0.0", "--port", "8000"] 
# lifespan - (ASGI Lifespan Support(wontfix) https://code.djangoproject.com/ticket/31508)
```

#### 9. Создаем и используем виртуальное окружение:
    cd /code/django/
    python3 -m venv env
    . ./env/bin/activate

#### 10. Устанавливаем Django, Uvicorn, psycopg2(postgres), записываем зависимости создаем проект:
    pip install -U Django
    pip install -U uvicorn gunicorn
    pip install -U psycopg2-binary
    pip freeze > ./requirements.txt
    django-admin startproject app

#### 11. Вывод команды `cat requirements.txt` будет примерно таким:
```bash
asgiref==3.6.0
backports.zoneinfo==0.2.1
click==8.1.3
Django==4.2.1
gunicorn==20.1.0
h11==0.14.0
psycopg2-binary==2.9.6
sqlparse==0.4.4
uvicorn==0.22.0
```


#### 12. Создаем docker-compose.yaml:
    vi /code/docker-compose.yaml
    
#### 13. Добавляем запись в файл `docker-compose.yaml`:
```yaml
services: 
  django:
    restart: always
    build:
      context: ./django          
    image: djangobackend
```

#### 14. Запускаем `docker-compose.yaml`, чтобы проверить работу сервиса uvicorn:
    cd /code
    docker-compose up

#### 15. Вывод команды:
```bash
code-django-1  | INFO:     Started server process [1]
code-django-1  | INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

## 2. Настройка БД

#### 1. Создаем файл настроек БД `.db_settings`:
    vi /code/postgresql/.db_settings

#### 2. Добавляем запись:
    POSTGRES_USER=deploy-test-user
    POSTGRES_PASSWORD=your-deploy-test-password
    POSTGRES_DB=deploy-db

#### 3. Добавляем сервис `postgresql` в `docker_compose.yaml`:
    vi /code/docker-compose.yaml


```yaml
  postgresql:
    restart: always
    image: postgres
    env_file: ./postgresql/.db_settings
```

#### 4. Создаем файл `local_settings.py`:
    vi /code/django/app/local_settings.py

#### 5. Добавляем запись(SECRET_KEY находится в переменной в файле /code/django/app/app/settings.py):
```Python
SECRET_KEY = 'django-insecure-7k85gz%c*y)xiur)p@2-g=vw$i=ev#l#wk5#34!1s#x+-89823'  
POSTGRES_DB = 'deploy-db'                                                 # как в .db_settings
POSTGRES_USER = 'deploy-test-user'                                        # как в .db_settings
POSTGRES_PASSWORD = 'your-deploy-test-password'                           # как в .db_settings
POSTGRES_HOST = 'postgresql'                             # service name in docker-compose.yaml
```


#### 6. Изменим файл настроек Django `settings.py`:
    vi /code/django/app/app/settings.py

#### 7. Добавим записи в файл `settings.py` и изменим переменнную `SECRET_KEY`:
```Python
import local_settings as settings

SECRET_KEY = settings.SECRET_KEY

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': settings.POSTGRES_DB,
        'USER': settings.POSTGRES_USER,
        'PASSWORD': settings.POSTGRES_PASSWORD,
        'HOST': settings.POSTGRES_HOST,
        'PORT': '', # default
    }
}
```

#### 8. Запускаем `docker-compose.yaml` для проверки БД:
```Bash
cd /code
docker-compose up
```

#### 9. Во втором терминале делаем миграцию для проверки внесения изменений в модели приложения:
```Bash
cd /code
docker-compose exec django bash
```
```Bash
cd app/
python manage.py migrate
```
#### 10. Вывод(должно быть OK):
```
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying sessions.0001_initial... OK
```


## 3. Настройка NGINX

#### 1. Создаем `Dockerfile` и добавляем запись:
    vi /code/nginx/Dockerfile

```Dockerfile
FROM nginx
COPY ./default.conf /etc/nginx/conf.d/default.conf 
```

#### 2. Создаем и добавляем запись в файл настройки nginx `default.conf`:
    vi /code/nginx/default.conf
    
```nginx
upstream innerdjango {
    server django:8000;   
}
server {
    listen 80;
    server_name animek.ru;
    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_pass http://innerdjango;
    }
}
```

#### 3. Добавляем в `docker-compose.yaml` сервис `nginx`:
```bash
vi /code/docker-compose.yaml
```
```yaml
  nginx:
    restart: always
    build:
      context: ./nginx
    ports:
      - "80:80"
```

#### 4. Изменяем переменную в файле настроек Django `settings.py`:
    vi /code/django/app/app/settings.py
```Python
ALLOWED_HOSTS = [ 'animek.ru' ]
 ```
 
#### 5. Запустим `docker-compose.yaml` и проверим nginx зайдя на сайт: 
    docker-compose up --build

![Cap123ture](https://github.com/artemkatee/dockerizing-django/assets/38987669/66f517a7-7a99-4e79-b91e-a3fcb9bb9d19)


## 4. Добавление SSL-шифрования и завершение настройки


#### 1. В файл `docker-compose.yaml` добавим сервис `certbot` и изменим сервис `nginx`:

    vi /code/docker-compose.yaml

```yaml
services:
  nginx:
    restart: always
    build:
      context: ./nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./persistentdata/certbot/conf:/etc/letsencrypt
      - ./persistentdata/certbot/www:/var/www/certbot
  certbot:
    image: certbot/certbot
    volumes:
      - ./persistentdata/certbot/conf:/etc/letsencrypt
      - ./persistentdata/certbot/www:/var/www/certbot
  django:
    restart: always
    build:
      context: ./django
    image: djangobackend
  postgresql:
    restart: always
    image: postgres
    env_file: ./postgresql/.db_settings

```

#### 2. Изменим файл конфигурации сервера `default.conf`:
    vi /code/nginx/default.conf

```nginx
upstream innerdjango {
    server django:8000;
}
server {
    listen 80;
    server_name animek.ru;
    location / {
      return 301 https://$host$request_uri;
    }
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}
```


#### 3. Запустим `docker-compose.yaml`:
    cd /code
```bash
docker-compose up --build
```

#### 4. Используя второй терминал отправляем запрос на создание сертификата(--email почта, -d доменное имя):
    cd /code

```bash
docker-compose run --rm --entrypoint "\
certbot certonly --webroot -w /var/www/certbot \
  --email name@mail.ru \
  -d animek.ru \
  --rsa-key-size 2048 \
  --agree-tos \
  --force-renewal" certbot
```

#### 5. Приводим файл настроек nginx `default.conf` к виду:
    vi /code/nginx/default.conf
```nginx
upstream innerdjango {
    server django:8000;   #service django in docker-compose
}
server {
    listen 80;
    server_name animek.ru;
    location / {
      return 301 https://$host$request_uri;
    }
    location /.well-known/acme-challenge/ { 
      root /var/www/certbot;
    }
}
server {
    listen 443 ssl;
    server_name animek.ru;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
    ssl_certificate /etc/letsencrypt/live/animek.ru/fullchain.pem; 
    ssl_certificate_key /etc/letsencrypt/live/animek.ru/privkey.pem;
    location / {
        proxy_pass http://innerdjango;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }
    location /static/ {
      root /var/www;
    }
    location /media/ {
      root /var/www;
    }
}
```

#### 6. Добавляем рекомендуемую конфигурацию для nginx:
```bash
wget https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf 
mv options-ssl-nginx.conf /code/persistentdata/certbot/conf/
wget https://raw.githubusercontent.com/certbot/certbot/master/certbot/certbot/ssl-dhparams.pem 
mv ssl-dhparams.pem /code/persistentdata/certbot/conf/
```

#### 7. Приводим файл `docker-compose.yaml` к виду:
    vi /code/docker-compose.yaml

```yaml
services:
  nginx:
    restart: always
    build:
        context: ./nginx
    ports:
      - "80:80"
      - "443:443"
    command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"
    volumes:
      - ./persistentdata/certbot/conf:/etc/letsencrypt
      - ./persistentdata/certbot/www:/var/www/certbot
      - ./persistentdata/static:/var/www/static
      - ./persistentdata/media:/var/www/media
  django:
    restart: always
    build:
      context: ./django
    image: djangobackend
    volumes:
      - ./persistentdata/static:/var/www/static
      - ./persistentdata/media:/var/www/media
  postgresql:
    restart: always
    image: postgres
    volumes:
      - ./persistentdata/db:/var/lib/postgresql/data
    env_file:
      - ./postgresql/.db_settings
  certbot:
    image: certbot/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
    volumes:
      - ./persistentdata/certbot/conf:/etc/letsencrypt
      - ./persistentdata/certbot/www:/var/www/certbot
```

#### 8. Добавляем строки в файл настроек Django `settings.py`:
    vi /code/django/app/app/settings.py

#### 9. После BASE_DIR = Path(__file__).resolve().parent.parent добавляем:
```python
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```
#### 10. После STATIC_URL = 'static/' добавляем:
```python
STATIC_ROOT = '/var/www/static'
MEDIA_URL = '/media/'
MEDIA_ROOT = '/var/www/media'
```

#### 11. Затем создаем суперпользователя для приложения:
    cd /code
    docker-compose up --build

#### 12. Во втором терминале по очереди вводим команды:
```bash
cd /code
docker-compose exec django bash
```
```bash
cd app/
python manage.py collectstatic
python manage.py migrate
python manage.py createsuperuser
```

![Captureфыв](https://github.com/artemkatee/dockerizing-django/assets/38987669/45564593-1f0b-4c69-96ad-53bf9d4c4d44)

