# Python-Assignment-
The assignment goal is to basically make a script to scrape some data from a website exposed as a django rest framework API.

##Main structure of my project:

crypto_scraper/
├── crypto_scraper/
│   ├── __init__.py
│   ├── asgi.py
│   ├── celery.py
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
├── api/
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations/
│   │   ├── __init__.py
│   ├── models.py
│   ├── tasks.py
│   ├── tests.py
│   ├── urls.py
│   ├── views.py
├── manage.py
└── requirements.txt




django-admin startproject crypto_scraper
cd crypto_scraper
django-admin startapp api (bash)              
                                         
                                        
                                                                                                    
pip install djangorestframework celery requests selenium                                            
(bash)

(python)

INSTALLED_APPS = [
    ...
    'rest_framework',
    'api',
]                                                                                                   
Configure Celery in settings.py:
                                                                                                    

 (python)                                                                                                  
# Celery configuration
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'


                                                                                                    
from __future__ import absolute_import, unicode_literals                                            
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'crypto_scraper.settings')

app = Celery('crypto_scraper')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
                                                                                        

                                                                                      
from .celery import app as celery_app                                                                
__all__ = ('celery_app',)                                                                            



(pyhton)
from celery import shared_task
import requests
from selenium import webdriver                                                                       
from selenium.webdriver.chrome.service import Service                                                
from selenium.webdriver.chrome.options import Options
from webdriver_manager.chrome import ChromeDriverManager

@shared_task
def fetch_crypto_data(coin_acronyms):
    data = {}
    options = Options()
    options.headless = True
    service = Service(ChromeDriverManager().install())
    driver = webdriver.Chrome(service=service, options=options)
    
    for acronym in coin_acronyms:
        url = f'https://coinmarketcap.com/currencies/{acronym}/'
        driver.get(url)
        # You will need to modify the following lines to correctly scrape the data you need
        try:
            price = driver.find_element_by_css_selector('.priceValue').text
            data[acronym] = price
        except Exception as e:
            data[acronym] = str(e)

    driver.quit()
    return data


from rest_framework.views import APIView
from rest_framework.response import Response                                                       

from rest_framework import status
from .tasks import fetch_crypto_data

class CryptoDataView(APIView):
    def post(self, request):
        coin_acronyms = request.data.get('coins', [])
        if not coin_acronyms:
            return Response({"error": "No coin acronyms provided"}, status=status.HTTP_400_BAD_REQUEST)
        
        task = fetch_crypto_data.delay(coin_acronyms)
        return Response({"task_id": task.id}, status=status.HTTP_202_ACCEPTED)

    def get(self, request):
        task_id = request.query_params.get('task_id')
        if not task_id:
            return Response({"error": "No task_id provided"}, status=status.HTTP_400_BAD_REQUEST)
        
        from celery.result import AsyncResult
        result = AsyncResult(task_id)
        if result.state == 'SUCCESS':
            return Response(result.result, status=status.HTTP_200_OK)
        else:
            return Response({"status": result.state}, status=status.HTTP_202_ACCEPTED)
            (python)



from django.urls import path                                                                 
from .views import CryptoDataView                                                            

urlpatterns = [
    path('crypto/', CryptoDataView.as_view(), name='crypto-data')
]

from django.contrib import admin                                                        

from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
] (python)


python manage.py runserver                                                                                                                                                                                                   
celery -A crypto_scraper worker --loglevel=infoRun  (bash)                                

                                                           -----END------
