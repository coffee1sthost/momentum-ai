# momentum-ai
A simple project powered by AI
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.getenv('SECRET_KEY', 'your-default-secret-key')
DEBUG = os.getenv('DEBUG', 'False') == 'True'
ALLOWED_HOSTS = ['*']

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'momentum',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME', 'railway'),
        'USER': os.getenv('DB_USER', 'postgres'),
        'PASSWORD': os.getenv('DB_PASSWORD', ''),
        'HOST': os.getenv('DB_HOST', 'localhost'),
        'PORT': os.getenv('DB_PORT', '5432'),
    }
}

STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': os.getenv('REDIS_URL', 'redis://127.0.0.1:6379/1'),
    }
}from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('momentum/', include('momentum.urls')),
]from django.db import models

class Momentum(models.Model):
    user = models.CharField(max_length=255)
    value = models.FloatField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"{self.user} - {self.value}"from django.http import JsonResponse
from django.core.cache import cache
from .models import Momentum

def get_momentum_data(request):
    data = cache.get('momentum_data')
    if not data:
        data = list(Momentum.objects.values('user', 'value', 'created_at'))
        cache.set('momentum_data', data, timeout=3600)
    return JsonResponse({'status': 'success', 'data': data})from django.urls import path
from .views import get_momentum_data

urlpatterns = [
    path('data/', get_momentum_data, name='momentum-data'),
]<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Momentum Dashboard</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">
</head>
<body class="bg-gray-100">
    <div class="container mx-auto">
        <h1 class="text-3xl font-bold text-center my-8">Momentum Dashboard</h1>
        <canvas id="momentumChart" class="w-full"></canvas>
    </div>
    <script>
        async function loadChartData() {
            const response = await fetch('/momentum/data/');
            const result = await response.json();
            const labels = result.data.map(item => new Date(item.created_at).toLocaleString());
            const data = result.data.map(item => item.value);

            new Chart(document.getElementById('momentumChart'), {
                type: 'line',
                data: {
                    labels,
                    datasets: [{
                        label: 'Momentum',
                        data,
                        borderColor: 'rgba(75, 192, 192, 1)',
                        tension: 0.1
                    }]
                },
                options: { responsive: true }
            });
        }
        loadChartData();
    </script>
</body>
</html>FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY backend/ ./backend/

CMD ["python", "backend/manage.py", "runserver", "0.0.0.0:8000"]version: '3.8'
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - SECRET_KEY=${SECRET_KEY}
      - DEBUG=${DEBUG}
      - DB_NAME=${DB_NAME}
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_HOST=${DB_HOST}
      - DB_PORT=${DB_PORT}
      - REDIS_URL=${REDIS_URL}
  redis:
    image: redis:alpine
django
djangorestframework
psycopg2-binary
redis