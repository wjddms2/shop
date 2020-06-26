# 6. 실전 프로젝트 - 쇼핑몰

- 웹 프로그래밍에서 가장 중요한 실습 과제
  - 게시판
  - 쇼핑몰
### 6.1 완성품 살펴보기
<그림 1> 상품 목록을 카테고리 별로 보여주는 메인 화면
![ch06_96_loggedin](https://user-images.githubusercontent.com/10287629/82289813-8c12fc80-99e0-11ea-82a7-d7b8b8256620.png)

<그림 2> 카테고리를 선택하여 상품 목록을 보여주는 화면
![ch06_02_category](https://user-images.githubusercontent.com/10287629/82334814-7246d900-9a23-11ea-9915-c985c8a55a7e.png)

<그림 3> 상품 상세보기 화면
![ch06_03_detail](https://user-images.githubusercontent.com/10287629/82335121-da95ba80-9a23-11ea-8509-65fa9325e6a7.png)

<그림 4> 장바구니
![ch06_04_cartl](https://user-images.githubusercontent.com/10287629/82335478-5abc2000-9a24-11ea-867f-538e7cd15f0d.png)

### 6.2 프로젝트 생성
- 아나콘다 파워쉘 프롬프트에서 가상환경 준비

```SHELL {.line-numbers}
# dstagram 가상환경을 복제한 shop 가상환경으로 시작
$ conda create --clone dstagram --name shop
# 가상환경 활성화
$ conda activate shop
# 설치된 패키지 목록 확인
$ conda list
```
- 파이참 프로젝트 생성
파이참 New Project
	- Pure Python
		- Location: c:\work\ch06_shop
		- Existing Interpreter: '...' 단추 클릭하여, 'Add Python Interpreter' 창
			- Conda Environment
				- Interpreter: c:\anaconda3\envs\shop\python.exe
				- OK
		- Create 단추 클릭
- 장고 프로젝트 생성
  파이참 내부에 ch06_shop 프로젝트 생성된 상태에서, 파이참 Terminal 작업

```SHELL {.line-numbers}
# 장고 관련 패키지 3종(django, django-disqus, django-extensions) 설치된 상황 확인
$ conda list django
# 장고 프로젝트 생성(ch06_shop 폴더 하위에 config 폴더와 manage.py 파일 생성됨)
$ django-admin startproject config .
```
### 6.3 DB 설정
- 교재에선 아마존 RDS를 통해 MySQL을 사용하지만, 로컬 서버의 MySQL을 사용하여 실습 진행
- MySQL 쉘에서 DB 생성

```SQL {.line-numbers}
mysql> CREATE DATABASE onlineshop default CHARACTER SET UTF8;
```
- 파이참 Terminal에서 pymysql 패키지 설치

```SHELL {.line-numbers}
# pymysql 설치
$ conda install pymysql --channel conda-forge
```
- config/settings.py 파일에서 MySQL 사용을 위한 준비

```PYTHON {.line-numbers}
import os
import pymysql				        # !!!
pymysql.version_info = (1, 3, 13, "final", 0)   # !!!
pymysql.install_as_MySQLdb()                    # !!!

# ...

# DATABASES = {
#     'default': {
#         'ENGINE': 'django.db.backends.sqlite3',
#         'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
#     }
# }
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'onlineshop',
        'USER': 'root',
        'PASSWORD': '???',  # !!! 자신의 비밀번호로 변경
        'HOST': 'localhost', # 'onlineshop.csfyvlroglcs.ap-northeast-2.rds.amazonaws.com',
        'PORT': '3306',
        'OPTIONS': {
            'init_command': "SET sql_mode='STRICT_TRANS_TABLES,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO'",
        },
    }
}
```
- DB 초기화 및 관리자 계정 생성

```SHELL {.line-numbers}
$ python manage.py migrate
$ python manage.py createsuperuser
```

### 6.4 정적 파일 및 미디어 파일 설정
- 교재에서는 아마존 S3 서버를 활용하지만, 우리 실습에서는 로컬 서버를 활용하여 진행
- config/settings.py

```PYTHON {.line-numbers}
LANGUAGE_CODE = 'ko'		# 관리자 페이지의 언어를 한글로 설정
TIME_ZONE = 'Asia/Seoul'	# 시간 대역을 서울로 설정
# ...
USE_TZ = False				# 이걸 False로 설정해야 Model 시간 대역도 서울로 적용됨

STATIC_URL = '/static/'
STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static'), ]
STATIC_ROOT = os.path.join(BASE_DIR, 'static_files')

MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```
- config/urls.py 미디어 파일 서빙을 위한 코드 추가는 나중에
`urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)`
### 6.5 Shop 앱 생성
#### 6.5.1 앱 생성

```SHELL {.line-numbers}
$ python manage.py startapp shop
```
- `settings.INSTALLED_APPS += 'shop',`
#### 6.5.2 모델 정의

```PYTHON {.line-numbers}
from django.db import models
from django.urls import reverse


class Category(models.Model):
    name = models.CharField(max_length=200, db_index=True)  # 카테고리 이름, DB 색인 열로 지정
    meta_description = models.TextField(blank=True)         # SEO(검색 엔진 최적화)를 위한 열 https://support.google.com/webmasters/answer/7451184?hl=ko

    slug = models.SlugField(max_length=200, db_index=True, unique=True,
		allow_unicode=True)  # 카테고리 및 상품 이름으로 URL 만들기, 유니코드 허용

    class Meta:
        ordering = ['name']
        verbose_name = 'category'           # 관리자 페이지 용
        verbose_name_plural = 'categories'  # 관리자 페이지 용

    def __str__(self):
        return self.name

    def get_absolute_url(self):
        return reverse('shop:product_in_category', args=[self.slug])


class Product(models.Model):
    category = models.ForeignKey(Category, on_delete=models.SET_NULL,
		null=True, related_name='products')  # 외래키, 부모 삭제 시 널 지정
    name = models.CharField(max_length=200, db_index=True)
    slug = models.SlugField(max_length=200, db_index=True, unique=True, allow_unicode=True)
    image = models.ImageField(upload_to='products/%Y/%m/%d', blank=True)
    description = models.TextField(blank=True)
    meta_description = models.TextField(blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)  # 가격, 소수점
    stock = models.PositiveIntegerField()                         # 재고, 음수 불허
    available_display = models.BooleanField('Display', default=True)    # 상품 노출 여부
    available_order = models.BooleanField('Order', default=True)        # 상품 주문 가능 여부
    created = models.DateTimeField(auto_now_add=True)       # settings.USE_TZ = False
    updated = models.DateTimeField(auto_now=True)           # settings.USE_TZ = False

    class Meta:
        ordering = ['-created']
        index_together = [['id','slug']]    # 다중 열 색인

    def __str__(self):
        return self.name

    def get_absolute_url(self):
        return reverse('shop:product_detail', args=[self.id, self.slug])
```

```SHELL {.line-numbers}
$ python manage.py makemigrations shop
$ python manage.py migrate shop
```
- 파이참에서 생성된 DB 확인
  - Databases에서 '+' 클릭, 'Data Sources and Drivers' 창에서 'onlineshop' 접속 테스트
  - Databases에서 onlineshop@localhost 내부의 schemas에 대해서 'Database Tools' > 'Force Refresh' 클릭
  - onlineshop 내부에 생성된 테이블 확인
#### 6.5.3 뷰 정의
- shop/views.py

```PYTHON {.line-numbers}
from django.shortcuts import render, get_object_or_404
from .models import *


def product_in_category(request, category_slug=None):
    # 맥락변수 3종을 목록 화면으로 전달
    current_category = None
    categories = Category.objects.all()
    products = Product.objects.filter(available_display=True)
    if category_slug:
        current_category = get_object_or_404(Category, slug=category_slug)
        products = products.filter(category=current_category)

    return render(request, 'shop/list.html',
                  {'current_category': current_category, # 지정된 카테고리 객체
                   'categories': categories,             # 모든 카테고리 객체
                   'products': products})                # 모든 또는 지정된 카테고리의 상품


def product_detail(request, id, product_slug=None):
    # 지정된 상품 객체를 상세 화면으로 전달
    product = get_object_or_404(Product,
                                id=id, slug=product_slug)
    return render(request, 'shop/detail.html', {'product': product})
```
- 지연평가(lazy evalution)
  - product_in_category() 뷰에서 filter() 메소드가 여러 번 호출되는 것처럼 코딩되었음
  - 실제로 데이터 질의는 단 한 번 호출됨
- get_object_or_404() 함수는 객체 검색이 실패할 경우 자동적으로 404 오류 페이지를 출력함
#### 6.5.4 접속경로 정의
- shop/urls.py

```PYTHON {.line-numbers}
from django.urls import path
from .views import *

app_name = 'shop'

urlpatterns = [
    path('', product_in_category,
         name='product_all'),  		# 카테고리 지정 없이, 모든 상품을 조회
    # path('<slug:category_slug>/', product_in_category, # !!! 한글 슬러그 문제
    #      name='product_in_category'), # 카테고리 지정하여, 해당 상품을 조회
    path('<category_slug>/', product_in_category,
         name='product_in_category'),   # 카테고리 지정하여, 해당 상품을 조회
    path('<int:id>/<product_slug>/', product_detail,
         name='product_detail'),        # 상품 지정하여, 해당 상품을 조회
]
```
- config/urls.py

```PYTHON {.line-numbers}
from django.contrib import admin
from django.urls import path, include
from django.conf.urls.static import static
from django.conf import settings

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('shop.urls'))
]
urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```
#### 6.5.5 템플릿 정의
- templates/base.html

```HTML {.line-numbers}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}{% endblock %}</title>
    <link rel="stylesheet"
          href="https://stackpath.bootstrapcdn.com/bootstrap/4.1.3/css/bootstrap.min.css"
          integrity="sha384-MCw98/SFnGE8fJT3GXwEOngsV7Zt27NXFoaoApmYm81iuXoPkFOJwJ8ERdknLPMO"
          crossorigin="anonymous">
    <!-- jquery slim 지우고 minified 추가 -->
    <script src="https://code.jquery.com/jquery-3.3.1.min.js"
            crossorigin="anonymous"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.3/umd/popper.min.js"
            integrity="sha384-ZMP7rVo3mIykV+2+9J3UJ46jBk0WLaUAdn689aCwoqbBJiSnjAK/l8WvCWPIPm49"
            crossorigin="anonymous"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.1.3/js/bootstrap.min.js"
            integrity="sha384-ChfqqxuZUCnJSK3+MXmPNIyE6ZbWh2IMqE241rYiqJxyMiZ6OW/JmZQ5stwEULTy"
            crossorigin="anonymous"></script>
    {% block script %}{% endblock %}
    {% block style %}{% endblock %}
</head>
<body>
{% load humanize %}
<nav class="navbar navbar-expand-lg navbar-light bg-light">
    <!--  브랜드 로고 및 햄버거 메뉴 아이콘  -->
    <a class="navbar-brand" href="/">Django Shop</a>
    <button class="navbar-toggler" type="button" data-toggle="collapse"
            data-target="#navbarSupportedContent"
            aria-controls="navbarSupportedContent" aria-expanded="false"
            aria-label="Toggle navigation">
        <span class="navbar-toggler-icon"></span>
    </button>
    <div class="collapse navbar-collapse justify-content-end" id="navbarSupportedContent">
        <ul class="navbar-nav justify-content-end">
	        <!-- 카트 요약 정보 -->
            <li class="nav-item active">
                <a class="nav-link btn btn-outline-success" href="#">Cart
                    {% if cart|length > 0 %}
                        &#8361;{{ cart.get_product_total | floatformat:'0' | intcomma }} with {{cart|length}} items
                    {% else %}
                        : Empty
                    {% endif %}
                </a>
            </li>
        </ul>
    </div>
</nav>
<div class="container">
    {% block content %}{% endblock %}
</div>
</body>
</html>
```
- config/settings.py 템플릿 설정
  `settings.TEMPLATES[0]['DIRS'][0] = os.path.join(BASE_DIR, 'templates')`
- shop/templates/shop/list.html

```HTML {.line-numbers}
{% extends 'base.html' %}
{% block title %}Category Page{% endblock %}
{% block content %}
    {% load humanize %}
    <div class="row">
        <div class="col-2">
            <div class="list-group">
                <a href="/" class="list-group-item {% if not current_category %}active{% endif %}">
                    All
                </a>
                {% for c in categories %}
                <a href="{{c.get_absolute_url}}"
                   class="list-group-item {% if current_category.slug == c.slug %}active{% endif %}">
                    {{c.name}}
                </a>
                {% endfor %}
            </div>
        </div>
        <div class="col">
            <div class="alert alert-info" role="alert">
                {% if current_category %}
                    {{current_category.name}}
                {% else %}
                    All Products
                {% endif %}
            </div>
            <div class="row">
            {% for product in products %}
                <div class="col-4">
                    <div class="card">
                        <img class="card-img-top" src="{{product.image.url}}" alt="Product Image">
                        <div class="card-body">
                            <h5 class="card-title">{{product.name}}</h5>
                            <p class="card-text">
                                {{product.description}}
                                <span class="badge badge-secondary">
									&#8361;{{product.price | floatformat:'0' | intcomma}}
                                </span>
                            </p>
                            <a href="{{product.get_absolute_url}}" class="btn btn-primary">View Detail</a>
                        </div>
                    </div>
                </div>
            {% endfor %}
            </div>
        </div>
    </div>
{% endblock %}
```
- shop/templates/shop/detail.html

```HTML {.line-numbers}
{% extends 'base.html' %}
{% block title %}Product Detail{% endblock %}
{% block content %}
    {% load humanize %}
    <div class="container">
        <div class="row">
            <div class="col-4">
                <img src="{{product.image.url}}" width="100%">
            </div>
            <div class="col">
                <h3 class="display-6">{{product.name}}</h3><br/>
                <p>
                    <span class="badge badge-secondary">Price</span>
                    &#8361;{{product.price | floatformat:'0' | intcomma}}
                </p>
                <p>
                    <span class="badge badge-secondary">Description</span>
                    {{product.description|linebreaksbr}}
                </p><br/>
                <form action="" method="post">
                    {% csrf_token %}
                    <input type="submit" class="btn btn-primary btn-sm" value="Add to Cart">
                </form>
            </div>
        </div>
    </div>
{% endblock %}
```
- 원화 표시 기호는 `'&#8361;'`를 사용하여 '&#8361;'로 출력
- 천 단위 구분 쉼표
	- `settings.INSTALLED_APPS += 'django.contrib.humanize'`
	- html 파일의 body 태그 및 콘텐츠 블럭 첫 줄에서 `{% load humanize %}` 형태로 로드
	- `{{ object.price | intcomma }}` 형태로 활용
- 실수형 가격을 정수형처럼 출력
    - `{{ object.price | floatformat:'0' }}` 형태로 활용
    - `{{ object.price | floatformat:'0' | intcomma }}`로 하면,
      천 단위 구분 쉼표를 포함한 정수형으로 출력됨
- 런 서버 확인
<그림 > 메인 화면에 브랜드 로고, 햄버거 메뉴, 카드 요약 정보 및 카테고리 표시한 모습
![ch06_51_list](https://user-images.githubusercontent.com/10287629/82181063-c0bd8000-991c-11ea-8767-4319b522836a.png)

#### 6.5.6 관리자 페이지 등록
- shop/admin.py

```PYTHON {.line-numbers}
from django.contrib import admin
from .models import *


class CategoryAdmin(admin.ModelAdmin):
    list_display = ['name', 'slug']
    prepopulated_fields = {'slug': ('name',)}  # slug 열을 name 열 값으로 자동 설정


admin.site.register(Category, CategoryAdmin)


class ProductAdmin(admin.ModelAdmin):
    list_display = ['name', 'slug', 'category', 'price', 'stock',
                    'available_display', 'available_order',
					'created', 'updated']
    list_filter = ['available_display', 'created', 'updated', 'category']
    prepopulated_fields = {'slug': ('name',)}  # slug 열을 name 열 값으로 자동 설정
    list_editable = ['price', 'stock', 'available_display', 'available_order']  # 목록에서도 주요 값 수정 허용


admin.site.register(Product, ProductAdmin)
```
- 관리자 화면 확인
<그림 > 관리자 화면에서 제품 목록 보는 상태에서 데이터 수정이 가능한 모습
![ch06_52_admin](https://user-images.githubusercontent.com/10287629/82183782-85718000-9921-11ea-92e7-3e06f48951dd.png)
