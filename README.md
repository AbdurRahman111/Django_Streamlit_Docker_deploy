# Django_Streamlit_Docker_deploy

Here’s how to deploy a Django app and a Streamlit app on separate subdomains using Docker Compose on an AWS Lightsail Linux instance with SSL certificates via Let's Encrypt.

---

### *Directory Structure*
Create the following directory structure:



![image](https://github.com/user-attachments/assets/b4a7eb84-93d2-47b9-8ea9-4364617ae5c3)
plaintext

project/

├── docker-compose.yml

├── django/

│   ├── Dockerfile

│   ├── app/  # Your Django app files

│   ├── requirements.txt

│   └── ...

├── streamlit/

│   ├── Dockerfile

│   ├── app/  # Your Streamlit app files

│   └── requirements.txt

├── nginx/

│   ├── conf.d/

│   │   ├── django.conf

│   │   ├── streamlit.conf

│   └── Dockerfile


---

### *Step 1: Django Dockerfile*
Create `django/Dockerfile`:

dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["gunicorn", "app.wsgi:application", "--bind", "0.0.0.0:8000"]


---

### *Step 2: Streamlit Dockerfile*
Create `streamlit/Dockerfile`:

dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8501

CMD ["streamlit", "run", "app/main.py", "--server.port=8501", "--server.enableCORS=false"]


---

### *Step 3: Nginx Configuration*
Create `nginx/Dockerfile`:

dockerfile
FROM nginx:alpine

COPY conf.d /etc/nginx/conf.d


#### *Django Nginx Config*
Create `nginx/conf.d/django.conf`:

nginx
server {
    server_name django.example.com;

    location / {
        proxy_pass http://django:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    listen 80;
}


#### *Streamlit Nginx Config*
Create `nginx/conf.d/streamlit.conf`:

nginx
server {
    server_name streamlit.example.com;

    location / {
        proxy_pass http://streamlit:8501;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    listen 80;
}


---

### *Step 4: Docker Compose*
Create `docker-compose.yml`:

yaml
version: "3.9"

services:
  django:
    build:
      context: ./django
    container_name: django
    expose:
      - "8000"

  streamlit:
    build:
      context: ./streamlit
    container_name: streamlit
    expose:
      - "8501"

  nginx:
    build:
      context: ./nginx
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - certbot_certs:/etc/letsencrypt

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot_certs:/etc/letsencrypt
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do sleep 2073600; done'"

volumes:
  certbot_certs:


---

### *Step 5: Obtain SSL Certificates*
Run Certbot to obtain SSL certificates for both subdomains:

bash
docker run --rm -v $(pwd)/nginx/certs:/etc/letsencrypt certbot/certbot certonly \
    --standalone -d django.example.com -d streamlit.example.com \
    --non-interactive --agree-tos --email your-email@example.com


---

### *Step 6: Update Nginx Config for SSL*
Update the Nginx configurations (`nginx/conf.d/django.conf` and `nginx/conf.d/streamlit.conf`) to include SSL:

#### *Django SSL Config*
nginx
server {
    server_name django.example.com;

    location / {
        proxy_pass http://django:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/django.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/django.example.com/privkey.pem;
}

server {
    listen 80;
    server_name django.example.com;
    return 301 https://$host$request_uri;
}


#### *Streamlit SSL Config*
nginx
server {
    server_name streamlit.example.com;

    location / {
        proxy_pass http://streamlit:8501;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/streamlit.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/streamlit.example.com/privkey.pem;
}

server {
    listen 80;
    server_name streamlit.example.com;
    return 301 https://$host$request_uri;
}


---

### *Step 7: Start the Applications*
Run the Docker Compose setup:

bash
docker-compose up --build -d


---

### *Step 8: Automate Certificate Renewal*
Add a cron job to automatically renew certificates:

1. Open the crontab editor:
   bash
   crontab -e
   

2. Add the renewal command:
   bash
   0 0 * * * docker run --rm -v $(pwd)/nginx/certs:/etc/letsencrypt certbot/certbot renew --quiet
   

---

### *Access Your Apps*
- Django: `https://django.example.com`
- Streamlit: `https://streamlit.example.com`

=================================================================================================================================









Here’s the updated code to include *Celery* as one of the containers, along with *Redis* as its message broker. Celery will work with the Django app to handle background tasks.

---

### Updated Directory Structure
plaintext
project/
├── docker-compose.yml
├── django/
│   ├── Dockerfile
│   ├── app/  # Your Django app files
│   ├── celery.py
│   ├── requirements.txt
│   └── ...
├── streamlit/
│   ├── Dockerfile
│   ├── app/  # Your Streamlit app files
│   └── requirements.txt
├── nginx/
│   ├── conf.d/
│   │   ├── django.conf
│   │   ├── streamlit.conf
│   └── Dockerfile

![image](https://github.com/user-attachments/assets/6f06ef65-06aa-41e7-80f6-0446535ed89d)

---

### *Step 1: Django Dockerfile*
Add *Celery dependencies* to Django's `requirements.txt`:
plaintext
celery[redis]==5.2.7


Create `django/Dockerfile`:
dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["gunicorn", "app.wsgi:application", "--bind", "0.0.0.0:8000"]


---

### *Step 2: Celery Configuration in Django*
Add `celery.py` to the `django/app/` directory:
python
from celery import Celery

# Initialize Celery
app = Celery('app')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()


Update `settings.py`:
python
# Celery Configuration
CELERY_BROKER_URL = 'redis://redis:6379/0'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'


---

### *Step 3: Redis Container*
Add Redis to `docker-compose.yml` (below).

---

### *Step 4: Updated `docker-compose.yml`*
Add Celery and Redis containers to the stack:
yaml
version: "3.9"

services:
  django:
    build:
      context: ./django
    container_name: django
    expose:
      - "8000"
    depends_on:
      - redis
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0

  celery:
    build:
      context: ./django
    container_name: celery
    command: celery -A app worker --loglevel=info
    depends_on:
      - django
      - redis
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0

  redis:
    image: redis:alpine
    container_name: redis
    expose:
      - "6379"

  streamlit:
    build:
      context: ./streamlit
    container_name: streamlit
    expose:
      - "8501"

  nginx:
    build:
      context: ./nginx
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - certbot_certs:/etc/letsencrypt

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot_certs:/etc/letsencrypt
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do sleep 2073600; done'"

volumes:
  certbot_certs:


---

### *Step 5: Celery Integration*
Add tasks in your Django app (e.g., `tasks.py`):
python
from celery import shared_task

@shared_task
def sample_task():
    return "Task completed!"


Call tasks in your Django views:
python
from .tasks import sample_task

def trigger_task(request):
    sample_task.delay()
    return HttpResponse("Task triggered!")


---

### *Step 6: Nginx Configuration*
No changes needed; the current configuration works for both Django and Streamlit.

---

### *Step 7: Run the Stack*
Build and start all containers:
bash
docker-compose up --build -d


---

### *Step 8: Test Celery*
Access your Django app and trigger a task. Verify Celery logs by running:
bash
docker logs celery


---

### *Result*
- Django and Streamlit apps run on separate subdomains with SSL.
- Celery handles background tasks with Redis as the broker.
