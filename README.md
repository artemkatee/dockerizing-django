# Развертывание веб-приложения используя докер
## Веб приложение

тут картинка
(nginx + uvicorn + django + postgresql + certbot)

## Создаем каталоги

#### Создаем каталог с веб-приложением:
    mkdir /code

#### Создаем каталоги для хранения постоянных данных:
    mkdir /code/persistentdata
    mkdir /code/persistentdata/static
    mkdir /code/persistentdata/media
    mkdir /code/persistentdata/db
    mkdir /code/persistentdata/certbot
    mkdir /code/persistentdata/certbot/www
    mkdir /code/persistentdata/certbot/conf
    mkdir /code/persistentdata/certbot/conf/live/
    mkdir /code/persistentdata/certbot/conf/live/animek.ru/ 

#### Создаем каталоги с фреймворком(django), Базой данных(postgres) и веб-сервером(nginx):
    mkdir /code/django
    mkdir /code/postgresql
    mkdir /code/nginx

#### Переходим в каталог и создаем локальный репозиторий в папке с проектом:
    cd /code
    git init . 

#### Создадем файл `.gitignore`:
    vi /code/.gitignore

#### Добавляем в `.gitignore` записи с этого сайта https://raw.githubusercontent.com/github/gitignore/master/Python.gitignore, а также(обязательно):
    # Django stuff:
    local_settings.py
    persistentdata/*
    .db_settings


#### Создаем Dockerfile в папке django:
    vi /code/django/Dockerfile

#### Добавляем запись в `Dockerfile`:
```Dockerfile
FROM python:3.9
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

#### Создаем и используем виртуальное окружение:
    cd /code/django/
    python3 -m venv env
    . ./env/bin/activate

#### Устанавливаем Django, Uvicorn, psycopg2(postgres), записываем зависимости создаем проект:
    pip install -U Django
    pip install -U uvicorn gunicorn
    pip install -U psycopg2-binary
    pip freeze > ./requirements.txt
    django-admin startproject app

#### Создаем docker-compose.yaml:
    vi /code/docker-compose.yaml
    
#### Добавляем запись в файл `docker-compose.yaml`:
```yaml
version: "3.9"
services: 
  django-backend:
    restart: always
    build:
      context: ./django          
    image: djangobackend
```

#### Запускаем `docker-compose.yaml`, чтобы проверить работу сервиса uvicorn:
    docker-compose up
![image](https://user-images.githubusercontent.com/38987669/236900483-bc042abc-47b8-4e65-ac12-c414a4751bc6.png)

## Настройка БД

#### Создаем файл настроек БД `.db_settings`:
    vi /code/postgresql/.db_settings

#### Добавляем запись:
    POSTGRES_USER=deploy-test-user
    POSTGRES_PASSWORD=your-deploy-test-password
    POSTGRES_DB=deploy-db

#### Добавляем сервис `postgresql` в `docker_compose.yaml`:
```Dockerfile
    postgresql:
        restart: always
        image: postgres
        env_file: ./postgresql/.db_settings
```

#### Создаем файл `local_settings.py`:
    vi /code/django/app/local_settings.py

#### Добавляем запись(SECRET_KEY находится в переменной в файле /code/django/app/app/settings.py):
```Python
SECRET_KEY = 'django-insecure-7k85gz%c*y)xiur)p@2-g=vw$i=ev#l#wk5#34!1s#x+-89823'  
POSTGRES_DB = 'deploy-db'                                                 # как в .db_settings
POSTGRES_USER = 'deploy-test-user'                                        # как в .db_settings
POSTGRES_PASSWORD = 'your-deploy-test-password'                           # как в .db_settings
POSTGRES_HOST = 'postgresql'                             # service name in docker-compose.yaml
```


#### Изменим файл настроек Django `settings.py`:
    vi /code/django/app/app/settings.py

#### Добавим записи в файл `settings.py` и изменим переменнную `SECRET_KEY`:
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

#### Запускаем `docker-compose.yaml` для проверки БД:
```Bash
cd /code
docker-compose up --build
```
##Тут картинка?:
    oisdfgiofd


#### Во втором терминале делаем миграцию для проверки внесения изменений в модели приложения:
```Bash
cd /code
docker-compose exec django bash
```
```Bash
cd app/
python manage.py migrate
```
#### (должно быть OK) картинка

## Настройка NGINX

#### Создаем `Dockerfile` и добавляем запись:
    vi /code/nginx/Dockerfile

```Dockerfile
FROM nginx
COPY ./default.conf /etc/nginx/conf.d/default.conf 
```

#### Создаем и добавляем запись в файл настройки nginx `default.conf`:
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

#### Добавляем в `docker-compose.yaml` сервис `nginx`:
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

#### Изменяем переменную в файле настроек Django `settings.py`:
    vi /code/django/app/app/settings.py
    
```Python
ALLOWED_HOSTS = [ 'animek.ru' ]
 ```
 
#### Запустим `docker-compose.yaml` и проверим nginx зайдя на сайт: 
```bash
docker-compose up --build
```



## Добавляем SSL-шифрование


Добавим сервис certbot

vi /code/docker-compose.yaml
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

изменим конфигурацию сервера

vi /code/nginx/default.conf
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

Генерируем сертификат

Запускаем 
docker-compose up --build


Во втором терминале отправляем запрос на создание сертификата

cd /code

docker-compose run --rm --entrypoint "\
certbot certonly --webroot -w /var/www/certbot \
  --email commandandconquer5@mail.ru \
  -d animek.ru \
  --rsa-key-size 2048 \
  --agree-tos \
  --force-renewal" certbot

Заканчиваем настройку

vi /code/nginx/default.conf
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

Добавляем рекомендуемый конфиг для nginx сервера
wget https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf 
mv options-ssl-nginx.conf /code/persistentdata/certbot/conf/ # move it
wget https://raw.githubusercontent.com/certbot/certbot/master/certbot/certbot/ssl-dhparams.pem 
mv ssl-dhparams.pem /code/persistentdata/certbot/conf/

vi /code/docker-compose.yaml
version: "3.9"

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
      - ./persistentdata/certbot/w

Добавляем строки в settings.py:
vi /code/django/app/app/settings.py

после BASE_DIR = Path(__file__).resolve().parent.parent добавляем:
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

После STATIC_URL = 'static/' добавляем:
STATIC_ROOT = '/var/www/static'
MEDIA_URL = '/media/'
MEDIA_ROOT = '/var/www/media'

Затем создаем суперпользователя для приложения:
docker-compose up --build

Во втором терминале по очереди вводим команды:

cd /code
docker-compose exec django bash
cd app/
python manage.py collectstatic
python manage.py migrate
python manage.py createsuperuser

