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


### 6.6 소셜 로그인 기능 추가
- 가상환경에 django-allauth 설치
```SHELL {.line-numbers}
$ conda install django-allauth
```
- config/settings.py 파일에 앱 등록
```PYTHON {.line-numbers}
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django.contrib.humanize',
    'shop',
    'django.contrib.sites',						# allauth 사이트 활용을 위해서
    'allauth',									# allauth 앱
    'allauth.account',							# 계정관리
    'allauth.socialaccount',					# 소셜 계정 관리
    'allauth.socialaccount.providers.naver',	# 네이버 계정 연동
]
# ...
# 파일 맨 끝에 추가
# 장고 인증은 사용자 이름을 쓰지만, allauth 인증은 이메일을 사용
AUTHENTICATION_BACKENDS = (
    'django.contrib.auth.backends.ModelBackend',			# 장고 방식 인증
    'allauth.account.auth_backends.AuthenticationBackend',	# allauth 인증
)
SITE_ID = 1
LOGIN_REDIRECT_URL = '/'
```
- config/urls.py 파일에 로그인 접속 경로 추가
```PYTHON {.line-numbers}
urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('allauth.urls')),  # !!!
    path('', include('shop.urls'))
]
```
- 추가한 앱을 위하여 DB 현행화 하면,
  - onlineshop DB에 'account_~' 및 'socialaccount_~' 테이블 다수 추가됨
  - 현재 상태에서도 기본 로그인/로그아웃은 작동함
```SHELL {.line-numbers}
$ python manage.py migrate
```
- templates/base.html 파일에서 '카트 요약 정보' 직전에 '로그인/로그아웃 메뉴' 추가
```HTML {.line-numbers}
<!-- 로그인/로그아웃 -->
<li class="nav-item">
{% if user.is_authenticated %}
	<a class="nav-link" href="{% url 'account_logout' %}">Logout</a> <!-- allauth 앱의 경로 이름 -->
{% else %}
	<a class="nav-link" href="{% url 'account_login' %}">Login</a>   <!-- allauth 앱의 경로 이름 -->
{% endif %}
</li>
<!-- 카트 요약 정보 -->
```
<그림> http://localhost:8000/accounts/logout/ 접속 화면
![ch06_53_logout](https://user-images.githubusercontent.com/10287629/82188142-60ccd680-9928-11ea-8a50-81b7982a99ee.png)
<그림> http://localhost:8000/accounts/login/ 접속 화면
![ch06_54_login](https://user-images.githubusercontent.com/10287629/82188318-ab4e5300-9928-11ea-9e49-c131d997f2a1.png)
<그림> http://localhost:8000/accounts/signup/ 접속 화면
![ch06_55_signup](https://user-images.githubusercontent.com/10287629/82188514-f4060c00-9928-11ea-91f5-2b24212b0816.png)
- 네이버 아이디로 로그인 기능 추가
  - [네이버 개발자 사이트](https://developers.naver.com/products/login/api) 접속

<그림 > 네이버 개발자 사이트에서 '오픈 API 이용 신청' 클릭
![ch06_90_naver](https://user-images.githubusercontent.com/10287629/82216289-b8346c00-9953-11ea-878f-9281a3c93e47.png)

<그림 > 애플리케이션 등록 (API 이용 신청)
![ch06_91_request_api](https://user-images.githubusercontent.com/10287629/82216883-8d96e300-9954-11ea-9280-8572c32b5141.png)
- 애플리케이션 이름: '장고 온라인 쇼핑몰'
- 사용 API에서 '선택하세요' 부분은 무시하고, '네아로 (네이버 아이디로 로그인)' 부분에서
  회원이름 및 이메일 필수 부분에 체크
- '로그인 오픈 API 서비스 환경' 부분에서 '환경추가' 선택 박스에서 'PC 웹' 선택하고, 다음 URL 입력
	- 서비스 URL: http://127.0.0.1:8000
	- Callback URL: http://127.0.0.1:8000/accounts/naver/login/callback/
	- Callback URL 끝에 '/' 포함됨에 유의
- '등록하기' 버튼 클릭하면 나타나는 페이지에서 다음 두 항목 값을 메모장 등에 복사해 두기
	- Client ID
	- Client Secret

<그림> 관리자 화면에서 '소셜 어플리케이션' 항목의 '+ 추가' 버튼 클릭
![ch06_93_s_app](https://user-images.githubusercontent.com/10287629/82218741-4f4ef300-9957-11ea-8ee7-72cf2de49bc2.png)

<그림> 나타나는 '소셜 어플리케이션 추가' 화면에서 다음과 같이 입력
![ch06_94_add_s_app](https://user-images.githubusercontent.com/10287629/82219964-fd0ed180-9958-11ea-9cec-6d23db21a51f.png)
- 제공자: Naver
- 이름: 네이버 로그인
- 클라이언트 아이디: <네이버 API에서 발급받은 Client ID>
- 비밀 키: <네이버 API에서 발급받은 Client Secret>
- Sites 부분에서 '이용 가능한 sites' 부분에 보이는 'example.com' 항목을 '선택된 sites' 부분으로 옮기고,
- '저장' 단추 클릭

<그림> 관리자 화면에서 '소셜 어플리케이션' 클릭하여 등록된 '네이버 로그인' 확인
![ch06_94_added_s_app](https://user-images.githubusercontent.com/10287629/82289258-5a4d6600-99df-11ea-85ec-13c3701016a3.png)

<그림> 우리 앱의 로그인 화면에서 'Naver' 링크 클릭하여 이용 동의하는 화면
![ch06_95_login](https://user-images.githubusercontent.com/10287629/82224565-f420fe80-995e-11ea-94da-d769e785f329.png)
- 일단 [로그아웃](http://127.0.0.1:8000/accounts/logout/)하고,
- 다시 [로그인](http://127.0.0.1:8000/accounts/login/) 화면에 접속하면 보이는 'Naver' 링크를 클릭하면
- '장고 온라인 쇼핑몰'에서 네이버 로그인 사용에 '동의하기' 단추 클릭 (최초 한 번만)

<그림> 로그인 직후, 메인 화면의 모습
![ch06_96_loggedin](https://user-images.githubusercontent.com/10287629/82289813-8c12fc80-99e0-11ea-82a7-d7b8b8256620.png)

<그림> 로그인 직후, 관리자 화면에서 사용자 조회하여 소셜 로그인으로 등록한 계정을 확인하는 화면
![ch06_97_loggedin_user](https://user-images.githubusercontent.com/10287629/82290002-deecb400-99e0-11ea-9e35-d82ee30e8187.png)
- logis: 장고 로그인 사용자
- logistex: 소셜 로그인 사용자

### 6.7 cart 앱 생성
#### 6.7.1 앱 생성
```SHELL {.line-numbers}
$ python manage.py startapp cart
```
- `settings.INSTALLED_APPS += 'cart',`
#### 6.7.2 카트 클래스 정의
- 상품 주문을 위해 일시적으로 저장하는 역할
  - DB에 저장하는 방식
  - 세션 기능을 활용하여 저장하는 방식
- cart/cart.py 작성
```PYTHON {.line-numbers}
from decimal import Decimal
from django.conf import settings
from shop.models import Product

class Cart(object):
    def __init__(self, request):
        # 장바구니 초기화 메소드 (request.session 변수에 저장)
        self.session = request.session
        cart = self.session.get(settings.CART_ID)  # CART_ID 변수는 settings.py 파일에서 정의
        if not cart:
            cart = self.session[settings.CART_ID] = {}
        self.cart = cart

    def __len__(self):
        # 장바구니의 상품별 수량에 대한 총계
        return sum(
            item['quantity'] for item in self.cart.values()
        )

    def __iter__(self):
        # 장바구니에 담긴 상품의 가격과 금액을 iterable 상태로 제공하는 메소드
        product_ids = self.cart.keys()
        products = Product.objects.filter(id__in=product_ids)
        for product in products:
            self.cart[str(product.id)]['product'] = product
        for item in self.cart.values():
            item['price'] = Decimal(item['price'])
            item['total_price'] = item['price'] * item['quantity']
            yield item  # https://tech.ssut.me/what-does-the-yield-keyword-do-in-python/

    def add(self, product, quantity=1, is_update=False):
        # 장바구니에 상품 추가
        product_id = str(product.id)
        if product_id not in self.cart:
            # 일단 수량을 0으로 초기화
            self.cart[product_id] = {'quantity': 0, 'price': str(product.price)}
        if is_update:  # 지정한 수량으로 수정
            self.cart[product_id]['quantity'] = quantity
        else:          # 지정한 만큼 수량 증가
            self.cart[product_id]['quantity'] += quantity
        self.save()

    def save(self):
        # 장바구니에 담긴 상품을 세션 변수로 저장
        self.session[settings.CART_ID] = self.cart
        self.session.modified = True

    def remove(self, product):
        # 장바구니에서 특정 상품을 삭제
        product_id = str(product.id)
        if product_id in self.cart:
            del(self.cart[product_id])
            self.save()

    def clear(self):
        # 장바구니를 비우는 기능, 주문 완료 경우에도 사용
        self.session[settings.CART_ID] = {}
        # self.session['coupon_id'] = None
        self.session.modified = True

    def get_product_total(self):
        # 장바구니에 담긴 상품의 총 가격을 계산
        return sum(
            Decimal(item['price']) * item['quantity'] for item in self.cart.values()
        )
```
- config/settings.py 파일 끝에 변수 추가
```PYTHON {.line-numbers}
CART_ID = 'cart_in_session'
```
- cart/forms.py 카트에 상품을 추가하기 위한 폼 작성
  - 상품 상세 페이지에서는 지정한 수량만큼 증가되도록 is_update 값을 False로 설정
  - 장바구니 페이지에서는 지정한 수량으로 설정되도록 is_update 값을 True로 설정
```PYTHON {.line-numbers}
from django import forms

class AddProductForm(forms.Form):
    quantity = forms.IntegerField()
    is_update = forms.BooleanField(required=False, initial=False,
                                   widget=forms.HiddenInput)
```
- cart/views.py 파일
```PYTHON {.line-numbers}
from django.shortcuts import render, redirect, get_object_or_404
from django.views.decorators.http import require_POST

from shop.models import Product
from .forms import AddProductForm
from .cart import Cart


@require_POST
def add(request, product_id):  # 장바구니에 상품을 추가하는 뷰
    cart = Cart(request)
    product = get_object_or_404(Product, id=product_id)
    form = AddProductForm(request.POST)
    if form.is_valid():
        cd = form.cleaned_data
        cart.add(product=product, quantity=cd['quantity'], is_update=cd['is_update'])
    return redirect('cart:detail')


def remove(request, product_id):  # 장바구니에서 지정한 상품을 삭제하는 뷰
    cart = Cart(request)
    product = get_object_or_404(Product, id=product_id)
    cart.remove(product)
    return redirect('cart:detail')


def detail(request):  # 장바구니 페이지 뷰
    cart = Cart(request)
    for product in cart:  # 장바구니에 담긴 상품마다 (수량 수정 가능하도록) AddProductForm 생성
        product['quantity_form'] = AddProductForm(
            initial={'quantity': product['quantity'], 'is_update': True}
        )
    return render(request, 'cart/detail.html', {'cart': cart})
```
#### 6.7.3 접속경로 정의
- cart/urls.py 생성
```PYTHON {.line-numbers}
from django.urls import path
from .views import *

app_name = 'cart'

urlpatterns = [
    path('', detail, name='detail'),
    path('add/<int:product_id>', add, name='product_add'),
    path('remove/<int:product_id>', remove, name='product_remove'),
]
```
- config/urls.py 한 줄 추가
```PYTHON {.line-numbers}
urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('allauth.urls')),
    path('cart/', include('cart.urls')),  		# !!!
    path('', include('shop.urls')),
]
```
#### 6.7.4 템플릿 정의
- cart/templates/cart/detail.html
```HTML {.line-numbers}
{% extends "base.html" %}
{% load static %}   <!--load 명령은 extends 보다는 뒤에!!!-->
{% load humanize %} <!--load 명령은 extends 보다는 뒤에!!!-->

{% block title %}
    Shopping cart
{% endblock %}

{% block content %}
    <h3>Shopping cart</h3>
    <table class="table table-striped">
        <thead>
            <tr>
                <th scope="col" style="width: 25%;">Image</th>
                <th scope="col">Product</th>
                <th scope="col">Quantity</th>
                <th scope="col">Remove</th>
                <th scope="col">Unit price</th>
                <th scope="col">Price</th>
            </tr>
        </thead>
        <tbody>
        {% for item in cart %}
            {% with product=item.product %}<!--with 명령-->
            <tr>
                <th scope="row">
                    <a href="{{ product.get_absolute_url }}">
                        <img src="{{ product.image.url }}" class="img-thumbnail" style="max-height: 90%; max-width: 90%">
                    </a>
                </th>
                <td>{{ product.name }}</td>
                <td>
                    <form action="{% url 'cart:product_add' product.id %}" method="post">
                        {% csrf_token %}
                        {{ item.quantity_form.quantity }}
                        {{ item.quantity_form.is_update }}
                        <div class="row-fluid">
                            <input type="submit" class="btn btn-primary pull-right form-control" value="Update">
                        </div>
                    </form>
                </td>
                <td><a href="{% url 'cart:product_remove' product.id %}">Remove</a></td>
                <td class="num">&#8361;{{item.price | floatformat:'0' | intcomma}}</td>
                <td class="num">&#8361;{{item.total_price | floatformat:'0' | intcomma}}</td>
            </tr>
            {% endwith %}
        {% endfor %}

            <tr class="total">
                <td>Total</td>
                <td colspan="4"></td>
                <td class="num">&#8361;{{cart.get_product_total | floatformat:'0' | intcomma}}</td>
            </tr>
        </tbody>
    </table>

    <p class="text-right">
        <a href='{% url "shop:product_all" %}' class="btn btn-secondary">Continue shopping</a>
    </p>
{% endblock %}
```
#### 6.7.5 상품 상세 페이지 수정
- shop.views.product_detail 함수 뷰에 장바구니에 담기 기능을 추가
```PYTHON {.line-numbers}
from cart.forms import AddProductForm                                       # !!!

def product_detail(request, id, product_slug=None):
    # 지정된 상품 객체를 상세 화면으로 전달
    product = get_object_or_404(Product, id=id, slug=product_slug)
    add_to_cart = AddProductForm(initial={'quantity': 1})                   # !!!
    return render(request, 'shop/detail.html', {'product': product,
                                                'add_to_cart': add_to_cart})# !!!
```
- shop/templates/shop/detail.html 템플릿에 장바구니에 담기 기능을 추가
  - add_to_cart 내용을 출력
  - form의 action 속성을 'cart:product_add' 접속 경로로 지정
```HTML {.line-numbers}
<form action="{% url 'cart:product_add' product.id %}" method="post">  <!-- !!! -->
	{{add_to_cart}}                                                    <!-- !!! -->
	{% csrf_token %}
	<input type="submit" class="btn btn-primary btn-sm" value="Add to Cart">
</form>
```
- templates/base.html 템플릿의 메인 메뉴에 있는 카트 링크를 'cart:detail' 접속 경로로 연결
```HTML {.line-numbers}
<!-- 카트 요약 정보 -->
<li class="nav-item active">
	<a class="nav-link btn btn-outline-success" href="{% url 'cart:detail' %}">Cart:
		{% if cart|length > 0 %}
			&#8361;{{ cart.get_product_total | floatformat:'0' | intcomma }} ({{cart|length}} items)
		{% else %}
			Empty
		{% endif %}
	</a>
</li>
```
<그림> 장바구니 페이지의 모습
![ch06_98_cart](https://user-images.githubusercontent.com/10287629/82321199-710bb100-9a0f-11ea-9454-6f8ddbce39f7.png)

- 카트 요약 정보 개선
  - (cart.views.detail() 뷰가 cart 맥락변수를 전달한) cart/detail.html에서는
  	카트 요약 정보가 정확하게 표시됨
  - (cart 맥락변수를 전달받지 못한) shop/list.html 및 shop/detail.html에서는
    카트 요약 정보가 정확하지 않게 표시됨
  - shop/views.py 파일의 두 뷰에서 각 템플릿으로 cart 맥락 정보를 전달하도록 수정
```PYTHON {.line-numbers}
# ...
from cart.cart import Cart  # ***

def product_in_category(request, category_slug=None):
	# ...
    cart = Cart(request)
#	return render(request, 'shop/list.html', {...,
                                               'cart': cart})

def product_detail(request, id, product_slug=None):
	# ...
	cart = Cart(request)
#	return render(request, 'shop/detail.html', {...,
											'cart': cart})
```
### 6.8 쿠폰 앱 생성
#### 6.8.1 앱 생성
```SHELL {.line-numbers}
$ python manage.py startapp coupon
```
- `settings.INSTALLED_APPS += 'coupon',`
#### 6.8.2 모델 정의
- coupon/models.py
```PYTHON {.line-numbers}
from django.db import models
from django.core.validators import MinValueValidator, MaxValueValidator


class Coupon(models.Model):
    code = models.CharField(max_length=50, unique=True)  # 쿠폰 사용할 때 입력할 코드
    use_from = models.DateTimeField()                    # 쿠폰 사용 시작 시점
    use_to = models.DateTimeField()                      # 쿠폰 사용 종료 시점
    amount = models.IntegerField(                        # 할인 금액 (최대/최소 제약)
        validators=[MinValueValidator(0), MaxValueValidator(100000)])
    active = models.BooleanField()						 # True인 쿠폰만 유효함

    def __str__(self):
        return self.code
```
- 모델 DB 현행화
```SHELL {.line-numbers}
$ python manage.py makemigrations coupon
$ python manage.py migrate coupon
```
#### 6.8.3 뷰 정의
- coupon/forms.py
```PYTHON {.line-numbers}
from django import forms


class AddCouponForm(forms.Form):  # 쿠폰 번호 입력하는 폼
    code = forms.CharField(label='Your Coupon Code')
```
- coupon/views.py
```PYTHON {.line-numbers}
from django.shortcuts import redirect
from django.utils import timezone
from django.views.decorators.http import require_POST

from .models import Coupon
from .forms import AddCouponForm


@require_POST
def add_coupon(request):  # 입력받은 쿠폰 코드로 쿠폰을 조회
    now = timezone.now()
    form = AddCouponForm(request.POST)
    if form.is_valid():
        code = form.cleaned_data['code']
        try:
            coupon = Coupon.objects.get(  # iexact로 대소문자 구분 없이
                code__iexact=code,        # use_from <= now <= use_to
                use_from__lte=now, use_to__gte=now, active=True)  # active=True
            request.session['coupon_id'] = coupon.id  # 쿠폰 id를 세션 변수로 저장
        except Coupon.DoesNotExist:
            request.session['coupon_id'] = None
    return redirect('cart:detail')  # 장바구니 보기로 리다이렉트
```
#### 6.8.4 접속 경로 정의
- coupon/urls.py
```PYTHON {.line-numbers}
from django.urls import path
from .views import add_coupon

app_name='coupon'

urlpatterns = [
    path('add/', add_coupon, name='add'),  # 쿠폰 추가
]
```
- config/urls.py 코드에 한 줄 추가
```PYTHON {.line-numbers}
urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('allauth.urls')),
    path('cart/', include('cart.urls')),
    path('coupon/', include('coupon.urls')),  # !!!
    path('', include('shop.urls')),
]
```
#### 6.8.5 카트 코드 수정
- cart/cart.py
```PYTHON {.line-numbers}
# ...
from coupon.models import Coupon


class Cart(object):
    def __init__(self, request):
        # 장바구니 초기화 메소드 (request.session 변수에 저장)
        self.session = request.session
        cart = self.session.get(settings.CART_ID)
        if not cart:
            cart = self.session[settings.CART_ID] = {}
        self.cart = cart
        self.coupon_id = self.session.get('coupon_id')  # 카트에 쿠폰 id 변수 추가


# ...
def clear(self):
	# 장바구니를 비우는 기능, 주문 완료 경우에도 사용
	self.session[settings.CART_ID] = {}
	self.session['coupon_id'] = None  # 장바구니 비울 때, 쿠폰 정보도 삭제
	self.session.modified = True

# ...
@property  # 클래스 메소드를 클래스 속성 변수로 만들기 위하여
def coupon(self):
	if self.coupon_id:
		return Coupon.objects.get(id=self.coupon_id)
	return None

def get_discount_total(self):
	if self.coupon:
		if self.get_product_total() >= self.coupon.amount:
			return self.coupon.amount
	return Decimal(0)  # (구매총액 < 쿠폰금액)이면 할인금액은 0

def get_total_price(self):  # 할인 적용된 촘 금액 반환
	return self.get_product_total() - self.get_discount_total()
```
- cart/views.py
```PYTHON {.line-numbers}
# ...
from coupon.forms import AddCouponForm  # !!!
# ...

def detail(request):  # 장바구니 페이지 뷰
    cart = Cart(request)
    add_coupon = AddCouponForm()  # !!! AddCouponForm
    for product in cart:
        product['quantity_form'] = AddProductForm(
            initial={'quantity': product['quantity'], 'is_update': True}
        )
    return render(request, 'cart/detail.html', 				 # 폼을 템플릿에 전달하여 출력
                  {'cart': cart, 'add_coupon': add_coupon})  # !!! AddCouponForm
```
- cart/templates/cart/detail.html 코드에 2 블럭 추가
```HTML {.line-numbers}
<!-- ... -->
<!-- tbody 내부에서 for 반복 끝나는 지점 밑에, 할인 전 총액과 쿠폰 관련 서브 토탈 행을 추가-->
	{% if cart.coupon %}  <!-- 카트에 쿠폰이 있을 경우에만 쿠폰 관련 할인 정보를 출력 -->
		<tr class="subtotal">
			<td>Subtotal</td>
			<td colspan="4"></td>
			<td >&#8361;{{cart.get_product_total | floatformat:'0' | intcomma}}</td>
		</tr>
		<tr>
			<td colspan="5">"{{ cart.coupon.code }}" coupon (&#8361;{{cart.coupon.amount | floatformat:'0' | intcomma}}) discounted</td>
			<td>&#8361;{{cart.get_discount_total | floatformat:'0' | intcomma}}</td>
    	</tr>
	{% endif %}
		<tr class="total">
			<td>Total</td>
			<td colspan="4"></td>
			<td class="num">&#8361;{{cart.get_total_price | floatformat:'0' | intcomma}}</td>
		</tr>
	</tbody>
</table>
<!-- table 닫는 태그 직후, 쿠폰 입력 폼을 추가-->
<p>
	Add Coupon:
</p>
<form action='{% url "coupon:add" %}' method="post">
	{{ add_coupon }}
	<input type="submit" value="Add">
	{% csrf_token %}
</form>
<!-- ... -->
```
#### 6.8.6 관리자 페이지에 모델 등록
- coupon/admin.py
```PYTHON {.line-numbers}
from django.contrib import admin
from .models import Coupon


class CouponAdmin(admin.ModelAdmin):
    list_display = ['code','use_from','use_to','amount', 'active']
    list_filter = ['active','use_from','use_to']
    search_fields = ['code']


admin.site.register(Coupon,CouponAdmin)
```
- 서버 실행 후, 관리자 페이지에서 쿠폰 등록하고, 장바구니 페이지에서 적용

<그림 > 장바구니 페이지에서 쿠폰을 활용하여 할인이 적용된 모습
![ch06_99_coupon](https://user-images.githubusercontent.com/10287629/82332572-876e3880-9a20-11ea-90e3-7ca8aba4ea6a.png)
- 쿠폰 할인의 효과가 화면 상단의 쿠폰 요약 정보에도 반영되도록, base.html에서
  `{{cart.get_product_total}}`을 `{{cart.get_total_price}}`로 수정

### 6.9 order 앱 작성
#### 6.9.1 앱 생성
```SHELL {.line-numbers}
$ python manage.py startapp order
```
- `settings.INSTALLED_APPS += 'order',`
#### 6.9.2 결제 API 준비
- iamport 서비스로 결제 연동하기 위해서,
  - iamport 서비스 API를 사용하기 위한 함수를 미리 작성
  - 이를 위해 iamport 서비스에 가입
    - https://www.iamport.kr/ 접속하여, 우측 상단의 '대시보드' 버튼 클릭
    - 로그인 화면이 나오면, 우측 하단의 '회원 가입' 버튼 클릭
    - 회원 가입 창에서 관련 정보 입력하고 '등록하기' 버튼 클릭
  - 등록하면 관리자 페이지가 뜨는데, 여기서 '시스템 설정' 탭을 클릭
    - 'PG설정(일반결제 및 정기결제)' 탭을 클릭
    - '기본 PG사'의 'PG사'에 'KT이니시스(웹표준결제창)'을 선택하고,
      '테스트모드(Sandbox)'가 'ON'으로 된 상태에서,
      '전체 저장' 버튼을 클릭
  - '재 정보' 탭을 클릭하여 보이는 다음 정보를 메모장 등에 보관하고, settings.py에 입력
    - REST API 키
    - REST API secret:
- config/settings.py 파일에 import 정보 입력
```PYTHON {.line-numbers}
IAMPORT_KEY = '당신의-API-key'
IAMPORT_SECRET = '당신의-secret-key'
```
- order/iamport.py 파일에 아임포트 API 통신용 메소드 작성
```PYTHON {.line-numbers}
import requests
from django.conf import settings

def get_token():
    # iamport 서버와 통신하기 위한 토큰을 받아오는 함수
    access_data = {
        'imp_key': settings.IAMPORT_KEY,
        'imp_secret': settings.IAMPORT_SECRET,
    }
    url = "https://api.iamport.kr/users/getToken"
    # requests : 특정 서버와 http 통신을 하게 해주는 모듈
    req = requests.post(url, data=access_data)
    access_res = req.json()
    if access_res['code'] is 0:
        return access_res['response']['access_token']
    else:
        return None

def payments_prepare(order_id, amount, *args, **kwargs):
    # 결제를 준비하는 함수 - iamport에 주문번호와 금액을 미리 전송
    access_token = get_token()
    if access_token:
        access_data = {
            'merchant_uid': order_id,
            'amount': amount,
        }
        url = "https://api.iamport.kr/payments/prepare"
        headers = {'Authorization': access_token, }
        req = requests.post(url, data=access_data, headers=headers)
        res = req.json()
        if res['code'] is not 0:
            raise ValueError("API 통신 오류")
    else:
        raise ValueError("토큰 오류")

def find_transaction(order_id,*args,**kwargs):
    # 결제 완료를 확인해주는 함수 - 실 결제 정보를 iamport에서 가져옴
    access_token = get_token()
    if access_token:
        url = "https://api.iamport.kr/payments/find/"+order_id
        headers = {'Authorization':access_token, }
        req = requests.post(url, headers=headers)
        res = req.json()
        if res['code'] is 0:
            context = {
                'imp_id': res['response']['imp_uid'],
                'merchant_order_id': res['response']['merchant_uid'],
                'amount': res['response']['amount'],
                'status': res['response']['status'],
                'type': res['response']['pay_method'],
                'receipt_url': res['response']['receipt_url'],
            }
            return context
        else:
            return None
    else:
        raise ValueError("토큰 오류")
```
- API 통신에서 사용하는 requests 패키지를 현재의 shop 가상환경에 설치
```SHELL {.line-numbers}
$ conda install requests
```
#### 6.9.3 모델 정의
- order/models.py
```PYTHON {.line-numbers}
from django.db import models
from django.core.validators import MinValueValidator, MaxValueValidator
import hashlib
from .iamport import payments_prepare, find_transaction
from django.db.models.signals import post_save

from coupon.models import Coupon
from shop.models import Product


class Order(models.Model):  # Order:OrderItem = 일:다
    # 주문 정보를 저장하는 모델 (회원 정보를 외래키로 연결하지 않고, 저장하는 방식)
    first_name = models.CharField(max_length=50)        # 주문자 정보
    last_name = models.CharField(max_length=50)
    email = models.EmailField()
    address = models.CharField(max_length=250)          # 주소
    postal_code = models.CharField(max_length=20)
    city = models.CharField(max_length=100)
    created = models.DateTimeField(auto_now_add=True)   # 일시
    updated = models.DateTimeField(auto_now=True)
    paid = models.BooleanField(default=False)           # 결제 여부
    coupon = models.ForeignKey(Coupon,                  # 쿠폰 정보
                               on_delete=models.PROTECT,
                               related_name='order_coupon', null=True, blank=True)
    discount = models.IntegerField(default=0,           # 할인 정보
                                   validators=[MinValueValidator(0),MaxValueValidator(100000)])

    class Meta:
        ordering = ['-created']

    def __str__(self):
        return 'Order {}'.format(self.id)

    def get_total_product(self):  # (할인 전) 주문 총액(=단가*수량)
        return sum(
            item.get_item_price() for item in self.items.all())  # 자식 테이블에서 지정한 related_name

    def get_total_price(self):  # (할인 후) 주문 총액
        total_product = self.get_total_product()
        return total_product - self.discount


class OrderItem(models.Model):  # Order:OrderItem = 일:다, OrderItem:Product = 다:일
    # 주문 내역 정보
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name='items')
    product = models.ForeignKey(Product, on_delete=models.PROTECT, related_name='order_products')
    price = models.DecimalField(max_digits=10, decimal_places=2)  # 상품 테이블 단가와 별도로 저장
    quantity = models.PositiveIntegerField(default=1)

    def __str__(self):
        return '{}'.format(self.id)

    def get_item_price(self):
        return self.price * self.quantity


class OrderTransactionManager(models.Manager):
    # OrderTransaction 모델의 관리자 클래스, 기본 관리자 클래스 objects 대신 사용
    def create_new(self, order, amount, success=None, transaction_status=None):
        if not order:
            raise ValueError("주문 오류")
        order_hash = hashlib.sha1(str(order.id).encode('utf-8')).hexdigest()
        email_hash = str(order.email).split("@")[0]
        final_hash = hashlib.sha1((order_hash  + email_hash).encode('utf-8')).hexdigest()[:10]
        merchant_order_id = "%s"%(final_hash)  # 아임포트에 결제 요청할 때 고유한 주문번호가 요구됨
        payments_prepare(merchant_order_id,amount)
        tranasction = self.model(order=order, merchant_order_id=merchant_order_id, amount=amount)
        if success is not None:
            tranasction.success = success
            tranasction.transaction_status = transaction_status
        try:
            tranasction.save()
        except Exception as e:
            print("save error",e)
        return tranasction.merchant_order_id

    def get_transaction(self, merchant_order_id):
        result = find_transaction(merchant_order_id)
        if result['status'] == 'paid':
            return result
        else:
            return None


class OrderTransaction(models.Model):
    # 결제 정보를 저장하는 모델
    order = models.ForeignKey(Order, on_delete=models.CASCADE)
    merchant_order_id = models.CharField(max_length=120, null=True, blank=True)
    transaction_id = models.CharField(max_length=120, null=True, blank=True)  # 정산 문제 확인 및 환불 처리 용도
    amount = models.PositiveIntegerField(default=0)
    transaction_status = models.CharField(max_length=220, null=True,blank=True)
    type = models.CharField(max_length=120, blank=True)
    created = models.DateTimeField(auto_now_add=True, auto_now=False)
    objects = OrderTransactionManager()  # 기본 관리자 클래스를 자작 관리자 클래스로 지정

    def __str__(self):
        return str(self.order.id)

    class Meta:
        ordering = ['-created']


def order_payment_validation(sender, instance, created, *args, **kwargs):
    # (특정 기능의 수행을 장고 앱 전체에 알리는) 시그널을 활용한 결제 검증 함수
    if instance.transaction_id:
        import_transaction = OrderTransaction.objects.get_transaction(merchant_order_id=instance.merchant_order_id)
        merchant_order_id = import_transaction['merchant_order_id']
        imp_id = import_transaction['imp_id']
        amount = import_transaction['amount']
        local_transaction = OrderTransaction.objects.filter(
            merchant_order_id=merchant_order_id, transaction_id= imp_id, amount=amount).exists()
        if not import_transaction or not local_transaction:
            raise ValueError("비정상 거래입니다.")

# 결제 정보가 생성된 후에 호출할 함수를 연결해준다.
post_save.connect(order_payment_validation, sender=OrderTransaction)
```
- DB 현행화
```SHELL {.line-numbers}
$ python manage.py makemigrations order
$ python manage.py migrate order
```
#### 6.9.4 뷰 정의
- order/forms.py 파일에 주문 정보 입력 폼 작성
```PYTHON {.line-numbers}
from django import forms

from .models import Order


class OrderCreateForm(forms.ModelForm):
    class Meta:
        model = Order
        fields = ['first_name', 'last_name', 'email', 'address', 'postal_code', 'city']
```
- order/views.py 파일에 주문 생성 뷰 작성
```PYTHON {.line-numbers}
from django.shortcuts import render, get_object_or_404
from django.views.generic.base import View  # 결제를 위한 임포트
from django.http import JsonResponse        # 결제를 위한 임포트

from .models import *
from cart.cart import Cart
from .forms import *


def order_create(request):
    # 주문 입력 뷰, 결제 진행 후 주문 정보를 저장,
    # ajax 기능으로 주문서 처리하므로 주문서 입력용 폼 출력하는 경우를 제외하면,
    # 자바스크립트가 동작하지 않는 환경에서만 입력 정보를 처리하는 뷰
    cart = Cart(request)
    if request.method == 'POST':  # 서버로 정보가 전달되면
        form = OrderCreateForm(request.POST)  # 주문서 입력 폼 저장
        if form.is_valid():
            order = form.save()  # 폼을 저장
            if cart.coupon:  # 카트에 쿠폰이 있으면, 주문에 적용
                order.coupon = cart.coupon
                order.discount = cart.coupon.amount
                order.save()
            for item in cart:  # 카트의 모든 상품을 주문내역으로 생성
                OrderItem.objects.create(
                    order=order, product=item['product'], price=item['price'], quantity=item['quantity'])
            cart.clear()  # 카트 비우기
            return render(request, 'order/created.html', {'order': order})  # 저장된 주문 정보를 맥락 정보로 전달
    else:  # 사용자가 정보를 서버로 전달하지 않은 상태라면
        form = OrderCreateForm()  # 주문서 입력 폼 생성하고, 카트 및 폼을 템플릿으로 전달
    return render(request, 'order/create.html', {'cart': cart, 'form': form})


# from django.contrib.admin.views.decorators import staff_member_required
# @staff_member_required
# def admin_order_detail(request, order_id):
#     order = get_object_or_404(Order, id=order_id)
#     return render(request, 'order/admin/detail.html', {'order':order})
#
# # pdf를 위한 임포트
# from django.conf import settings
# from django.http import HttpResponse
# from django.template.loader import render_to_string
# import weasyprint
#
# @staff_member_required
# def admin_order_pdf(request, order_id):
#     order = get_object_or_404(Order, id=order_id)
#     html = render_to_string('order/admin/pdf.html', {'order':order})
#     response = HttpResponse(content_type='application/pdf')
#     response['Content-Disposition'] = 'filename=order_{}.pdf'.format(order.id)
#     weasyprint.HTML(string=html).write_pdf(response, stylesheets=[weasyprint.CSS(settings.STATICFILES_DIRS[0]+'/css/pdf.css')])
#     return response


def order_complete(request):
    # ajax로 결제 후에 보여줄 결제 완료 화면
    order_id = request.GET.get('order_id')  # 서버로부터 주문번호를 얻어와서
    order = Order.objects.get(id=order_id)  # 이 주문번호를 이용해서 주문 완료 화면을 출력
    return render(request, 'order/created.html', {'order': order})


class OrderCreateAjaxView(View):  # Ajax 뷰
    # 입력된 주문 정보를 서버에 저장하고, 카트 상품을 OrderItem 객체에 저장하고, 카트 비우기
    def post(self, request, *args, **kwargs):
        if not request.user.is_authenticated:  # 인증되지 않은 사용자라면 403 상태 반환
            return JsonResponse({"authenticated": False}, status=403)
        cart = Cart(request)                    # 카트 정보 획득
        form = OrderCreateForm(request.POST)    # 입력된 주문 정보 획득
        if form.is_valid():  # 폼이 정당하면
            order = form.save(commit=False)  # 폼을 메모리에서 저장
            if cart.coupon:                  # 쿠폰 정보 처리
                order.coupon = cart.coupon
                order.discount = cart.coupon.amount
            order = form.save()             # 폼 저장
            for item in cart:               # 카트를 OrderItem으로 저장
                OrderItem.objects.create(order=order, product=item['product'],
                                         price=item['price'], quantity=item['quantity'])
            cart.clear()                    # 카트 비우기
            data = {"order_id": order.id}
            return JsonResponse(data)       # 주문번호를 반환
        else:   # 폼이 정당하지 않으면 401 상태 반환
            return JsonResponse({}, status=401)


class OrderCheckoutAjaxView(View):  # 결제 정보 OrderTransaction 객체 생성
    def post(self, request, *args, **kwargs):
        if not request.user.is_authenticated:  # 인증되지 않은 사용자이면 403 상태 반환
            return JsonResponse({"authenticated":False}, status=403)
        order_id = request.POST.get('order_id')  # 입력된 주문번호 획득하여 주문 검색
        order = Order.objects.get(id=order_id)
        amount = request.POST.get('amount')      # 입력된 금액 획득
        try:  # 결제용 id 생성을 시도
            merchant_order_id = OrderTransaction.objects.create_new(
                order=order, amount=amount)
        except:  # 결제용 id 생성 실패한 경우
            merchant_order_id = None
        if merchant_order_id is not None:  # 결제용 id 생성이 성공한 경우
            data = {"works": True, "merchant_id": merchant_order_id}
            return JsonResponse(data)
        else:  # 결제용 id 생성이 실패한 경우 401 상태 반환
            return JsonResponse({}, status=401)


class OrderImpAjaxView(View):  # 실제 결제 여부 확인
    def post(self, request, *args, **kwargs):
        if not request.user.is_authenticated:  # 사용자 인증 실패이면 403 반환
            return JsonResponse({"authenticated": False}, status=403)
        order_id = request.POST.get('order_id')  # 주문번호 획득
        order = Order.objects.get(id=order_id)
        merchant_id = request.POST.get('merchant_id')  # 결제번호 획득
        imp_id = request.POST.get('imp_id')
        amount = request.POST.get('amount')
        try:  # 결제 객체 검색 시도
            trans = OrderTransaction.objects.get(
                order=order, merchant_order_id=merchant_id, amount=amount)
        except:
            trans = None
        if trans is not None:  # 결제 객체 검색 성공이면
            trans.transaction_id = imp_id
            trans.success = True
            trans.save()
            order.paid = True
            order.save()
            data = {"works": True}
            return JsonResponse(data)
        else:  # 결제 객체 검색 실패하면 401 반환
            return JsonResponse({}, status=401)
```
#### 6.9.5 접속 경로 정의
- order/urls.py 작성
```PYTHON {.line-numbers}
from django.urls import path
from .views import *

app_name = 'orders'

urlpatterns = [
    path('create/', order_create, name='order_create'),
    path('create_ajax/', OrderCreateAjaxView.as_view(),name='order_create_ajax'),
    path('checkout/', OrderCheckoutAjaxView.as_view(),name='order_checkout'),
    path('validation/', OrderImpAjaxView.as_view(),name='order_validation'),
    path('complete/', order_complete,name='order_complete'),
]
```
- config/urls.py 수정
```PYTHON {.line-numbers}
urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('allauth.urls')),
    path('cart/', include('cart.urls')),
    path('coupon/', include('coupon.urls')),
    path('order/', include('order.urls')),  # !!!
    path('', include('shop.urls')),
]
```

#### 6.9.6 템플릿 정의
- order/tempates/order/create.html (주문 정보 입력 페이지)
```HTML {.line-numbers}
{% extends 'base.html' %}

{% block title %}Checkout{% endblock %}

{% block script %}
    <script type="text/javascript">  // 결제를 위한 ajax 통신 URL 변수를 정의하는 JS
        csrf_token = '{{ csrf_token }}';
        order_create_url = '{% url "orders:order_create_ajax" %}';
        order_checkout_url = '{% url "orders:order_checkout" %}';
        order_validation_url = '{% url "orders:order_validation" %}';
        order_complete_url = '{% url "orders:order_complete" %}';
    </script>

    <script src="https://cdn.iamport.kr/js/iamport.payment-1.1.5.js" type="text/javascript"></script>

    {% load static %}
    <script src="{% static 'js/checkout.js' %}" type="text/javascript"></script>
{% endblock %}

{% block content %}
<div class="row">
    <div class="col">
        <div class="alert alert-info" role="alert">
          Your Order
        </div>
        <ul class="list-group">
            {% for item in cart %}
                <li class="list-group-item">
                    {{item.quantity}}X{{item.product.name}}
                    <span>{{item.total_price}}</span>
                </li>
            {% endfor %}
            {% if cart.coupon %}
                <li class="list-group-item">
                    {% load humanize %}
                    "{{ cart.coupon.code }}" ({{ cart.coupon.amount }}% off)
                    <span>- ${{ cart.get_total_discount | floatformat:'0' | intcomma }}</span>
                </li>
            {% endif %}
        </ul>
        <div class="alert alert-success" role="alert">Total : {{cart.get_total_price|floatformat:"2"}}</div>
        <form action="" method="post" class="order-form">  <!-- form에 class 추가 -->
            {{form.as_p}}
            {% csrf_token %}
            <!-- hidden field 추가-->
            <input type="hidden" name="pre_order_id" value="0">
            <input type="hidden" name="amount" value="{{ cart.get_total_price|floatformat:'2' }}">
            <input type="submit" class="btn btn-primary float-right" value="Place Order">
        </form>
    </div>
</div>
{% endblock %}
```
- static/js/checkout.js (실 결제를 위한 JS 코드)
```javascript {.line-numbers}
$(function () {
    var IMP = window.IMP;
    IMP.init('가맹점식별코드');  // 아임포트 결제 준비 (가맹점식별코드는 아임포트 관리자 페이지 시스템 설정 창에서 확인)
    $('.order-form').on('submit', function (e) {  // 주문 폼 입력 후 'Place Order' 버튼 클릭하면
        var amount = parseFloat($('.order-form input[name="amount"]').val().replace(',', ''));
        var type = $('.order-form input[name="type"]:checked').val();
        // 폼 데이터를 기준으로 주문 생성
        var order_id = AjaxCreateOrder(e);
        if (order_id == false) {
            alert('주문 생성 실패\n다시 시도해주세요.');
            return false;
        }
        // 결제 정보 생성
        var merchant_id = AjaxStoreTransaction(e, order_id, amount, type);
        // 결제 정보가 만들어졌으면 iamport로 실제 결제 시도
        if (merchant_id !== '') {
            IMP.request_pay({
                merchant_uid: merchant_id,
                name: 'E-Shop product',
                buyer_name:$('input[name="first_name"]').val()+" "+$('input[name="last_name"]').val(),
                buyer_email:$('input[name="email"]').val(),
                amount: amount
            }, function (rsp) {
                if (rsp.success) {
                    var msg = '결제가 완료되었습니다.';
                    msg += '고유ID : ' + rsp.imp_uid;
                    msg += '상점 거래ID : ' + rsp.merchant_uid;
                    msg += '결제 금액 : ' + rsp.paid_amount;
                    msg += '카드 승인번호 : ' + rsp.apply_num;
                    // 결제가 완료되었으면 비교해서 디비에 반영
                    ImpTransaction(e, order_id, rsp.merchant_uid, rsp.imp_uid, rsp.paid_amount);
                } else {
                    var msg = '결제에 실패하였습니다.';
                    msg += '에러내용 : ' + rsp.error_msg;
                    console.log(msg);
                }
            });
        }
        return false;
    });
});

// 폼 데이터로 주문 생성 (주문정보를 서버로 전달해 Order 객체 생성)
function AjaxCreateOrder(e) {
    e.preventDefault();
    var order_id = '';
    var request = $.ajax({
        method: "POST",
        url: order_create_url,
        async: false,
        data: $('.order-form').serialize()
    });
    request.done(function (data) {
        if (data.order_id) {
            order_id = data.order_id;
        }
    });
    request.fail(function (jqXHR, textStatus) {
        if (jqXHR.status == 404) {
            alert("페이지가 존재하지 않습니다.");
        } else if (jqXHR.status == 403) {
            alert("로그인 해주세요.");
        } else {
            alert("문제가 발생했습니다. 다시 시도해주세요.");
        }
    });
    return order_id;
}

// 결제 정보 저장(OrderTransaction 객체를 생성하는 뷰 호출하고 아임포트에 전달할 merchant_id 반환)
function AjaxStoreTransaction(e, order_id, amount, type) {
    e.preventDefault();
    var merchant_id = '';
    var request = $.ajax({
        method: "POST",
        url: order_checkout_url,
        async: false,
        data: {
            order_id : order_id,
            amount: amount,
            type: type,
            csrfmiddlewaretoken: csrf_token,
        }
    });
    request.done(function (data) {
        if (data.works) {
            merchant_id = data.merchant_id;
        }
    });
    request.fail(function (jqXHR, textStatus) {
        if (jqXHR.status == 404) {
            alert("페이지가 존재하지 않습니다.");
        } else if (jqXHR.status == 403) {
            alert("로그인 해주세요.");
        } else {
            alert("문제가 발생했습니다. 다시 시도해주세요.");
        }
    });
    return merchant_id;
}

// 결제 완료 후 결제 검증
// iamport에 결제 정보가 있는지 확인 후 결제 완료 페이지로 이동
function ImpTransaction(e, order_id,merchant_id, imp_id, amount) {
    e.preventDefault();
    var request = $.ajax({
        method: "POST",
        url: order_validation_url,
        async: false,
        data: {
            order_id:order_id,
            merchant_id: merchant_id,
            imp_id: imp_id,
            amount: amount,
            csrfmiddlewaretoken: csrf_token
        }
    });
    request.done(function (data) {
        if (data.works) {
            $(location).attr('href', location.origin+order_complete_url+'?order_id='+order_id)
        }
    });
    request.fail(function (jqXHR, textStatus) {
        if (jqXHR.status == 404) {
            alert("페이지가 존재하지 않습니다.");
        } else if (jqXHR.status == 403) {
            alert("로그인 해주세요.");
        } else {
            alert("문제가 발생했습니다. 다시 시도해주세요.");
        }
    });
}
```
- order/templates/order/created.html (주문 완료 화면 작성)
```HTML {.line-numbers}
{% extends 'base.html' %}

{% block title %}Order Complete{% endblock %}

{% block content %}
    <div class="row">
        <div class="col">
            <div class="alert alert-success" role="alert">Thank you!<br/>
                Your order has been successfully completed.<br/>
                Your order number is <strong>{{order.id}}</strong></div>
        </div>
    </div>
{% endblock %}
```
- config/settings.py 파일에 정적 폴더 설정 
```PYTHON {.line-numbers}
STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static'), ]
```
- collectstatic 작업 수행하여, 정적 파일을 static_files 폴더로 수집
```shell {.line-numbers}
$ python manage.py collectstatic
```
- cart/templates/cart/detail.html 파일 수정(장바구니 페이지에 주문 버튼 추가)
```HTML {.line-numbers}
<p class="text-right">
    <a href='{% url "shop:product_all" %}' class="btn btn-secondary">Continue shopping</a>
    <a href='{% url "orders:order_create" %}' class="btn btn-primary">Checkout</a> <!-- !!! -->
</p>
```
#### 6.9.7 결제 시험
<그림> 장바구니에서 결제하기 누르기 직전의 모습
![ch06_110_cart](https://user-images.githubusercontent.com/10287629/83474279-934d0680-a4c6-11ea-99f0-2f28f5b805ca.png)

<그림> 주문 확정하는 모습 
![ch06_120_checkout](https://user-images.githubusercontent.com/10287629/83474322-afe93e80-a4c6-11ea-91f0-f8e2c18d1a7e.png)

<그림> 결제 대행사에서 결제 수단 선택하는 모습
![ch06_130_checkout](https://user-images.githubusercontent.com/10287629/83474345-b8da1000-a4c6-11ea-9780-642f0562ee7f.png)

<그림> 카카오페이로 결제하는 모습
![ch06_135_checkout](https://user-images.githubusercontent.com/10287629/83474337-b7104c80-a4c6-11ea-906d-7eda25c1b035.png)

<그림> 결제 완료 화면
![ch06_140_checkedout](https://user-images.githubusercontent.com/10287629/83474339-b7a8e300-a4c6-11ea-9c56-5d317703b408.png)

<그림> 카톡으로 전송된 결제 요청 메시지

![ch06_149_KakaoTalk_20200602_113037687](https://user-images.githubusercontent.com/10287629/83474342-b8417980-a4c6-11ea-8887-ecf8dba63abf.png)

<그림> 카톡으로 전송된 결제 확인 메시지

![ch06_150_checkedout](https://user-images.githubusercontent.com/10287629/83474343-b8da1000-a4c6-11ea-833f-33b554c0e86c.png)

<그림> 결제 알림 메일 
![ch06_151_notice](https://user-images.githubusercontent.com/10287629/83510316-bf3cac00-a507-11ea-9e21-565dd6911f3d.png)


#### 6.9.8 관리자 페이지 커스터마이징
- order/admin.py 수정
```PYTHON {.line-numbers}
import csv
import datetime
from django.http import HttpResponse
from django.urls import reverse
from django.utils.safestring import mark_safe
from django.contrib import admin

from .models import OrderItem, Order


def export_to_csv(modeladmin, request, queryset):   # 주문 목록을 csv로 저장하는 함수
    opts = modeladmin.model._meta
    response = HttpResponse(content_type='text/csv')
    # HttpResponse 객체로 응답을 만들 때, 'Content-Disposition' 값을 attachment 형식으로 설정하면
    # 브라우저가 이 응답을 파일로 다운로드해 줌
    response['Content-Disposition'] = 'attachment;filename={}.csv'.format(opts.verbose_name)
    writer = csv.writer(response)
    fields = [field for field in opts.get_fields() if not field.many_to_many and not field.one_to_many]
    # csv 파일 컬럼 타이틀 줄
    writer.writerow([field.verbose_name for field in fields])
    # 실제 데이터 출력
    for obj in queryset:
        data_row = []
        for field in fields:
            value = getattr(obj, field.name)
            if isinstance(value, datetime.datetime):
                value = value.strftime("%Y-%m-%d")
            data_row.append(value)
        writer.writerow(data_row)
    return response


def order_detail(obj):  # 주문 목록에 열 데이터로 출력되는 값을 뷰를 호출하여 HTML 태그로 생성
    return mark_safe('<a href="{}">Detail</a>'.format(reverse('orders:admin_order_detail', args=[obj.id])))


def order_pdf(obj):  # 주문 목록에 열 데이터로 출력되는 값을 뷰를 호출하여 HTML 태그로 생성
    return mark_safe('<a href="{}">PDF</a>'.format(reverse('orders:admin_order_pdf', args=[obj.id])))


class OrderItemInline(admin.TabularInline):  # TabularInline 상속, 주문 정보 아래에 주문 내역 출력
    model = OrderItem
    raw_id_fields = ['product']


class OrderAdmin(admin.ModelAdmin):
    list_display = ['id','first_name','last_name','email','address','postal_code','city','paid',order_detail, order_pdf,'created','updated']
    list_filter = ['paid','created','updated']
    inlines = [OrderItemInline]  # 다른 모델과 연결되어있는 경우 한페이지 표시하고 싶을 때
    actions = [export_to_csv]


# 해당 함수를 관리자 페이지에 명령으로 추가할 때 사용할 이름 지정
export_to_csv.short_description = 'Export to CSV'
order_detail.short_description = 'Detail'
order_pdf.short_description = 'PDF'

admin.site.register(Order, OrderAdmin)
```
- order/views.py (admin_order_detail 및 admin_order_pdf 뷰 작성)
```PYTHON {.line-numbers}
from django.contrib.admin.views.decorators import staff_member_required
@staff_member_required  # 관리자만 접근 허가
def admin_order_detail(request, order_id):
    order = get_object_or_404(Order, id=order_id)
    return render(request, 'order/admin/detail.html', {'order':order})
    
# pdf를 위한 임포트
from django.conf import settings
from django.http import HttpResponse
from django.template.loader import render_to_string
import weasyprint

@staff_member_required  # 관리자만 접근 허가
def admin_order_pdf(request, order_id):
    order = get_object_or_404(Order, id=order_id)
    html = render_to_string('order/admin/pdf.html', {'order':order})
    response = HttpResponse(content_type='application/pdf')
    response['Content-Disposition'] = 'filename=order_{}.pdf'.format(order.id)
    weasyprint.HTML(string=html).write_pdf(response, stylesheets=[weasyprint.CSS(settings.STATICFILES_DIRS[0]+'/css/pdf.css')])
    return response    
```
- order/urls.py 수정
```PYTHON {.line-numbers}
urlpatterns = [
    path('create/', order_create, name='order_create'),
    path('create_ajax/', OrderCreateAjaxView.as_view(),name='order_create_ajax'),
    path('checkout/', OrderCheckoutAjaxView.as_view(),name='order_checkout'),
    path('validation/', OrderImpAjaxView.as_view(),name='order_validation'),
    path('complete/', order_complete,name='order_complete'),
    path('admin/order/<int:order_id>/', admin_order_detail, name='admin_order_detail'),  # !!!
    path('admin/order/<int:order_id>/pdf/', admin_order_pdf, name='admin_order_pdf'),  # !!!
]
```
- order/templates/order/admin/detail.html (관리자 페이지를 위한 주문 상세 페이지)
```HTML {.line-numbers}
{# 부모 템플릿 상속 C:\anaconda3\envs\shop\lib\site-packages\django\contrib\admin\templates\admin\base.html#}
{% extends 'admin/base_site.html' %} 

{% block title %}Order {{order.id}}{% endblock %}

{% block breadcrumbs %}
    <div class="breadcrumbs">
        <a href="{% url 'admin:index' %}">Home</a> &rsaquo;
        <a href="{% url 'admin:order_order_changelist' %}">Orders</a> &rsaquo;
        <a href="{% url 'admin:order_order_change' order.id %}">Order {{order.id}}</a> &rsaquo;
        Detail
    </div>
{% endblock %}


{% block content %}
    <h1>Order {{order.id}}</h1>
    <ul class="object-tools">
        <li>
            <a href="#" onclick="window.print();">Print order</a>
        </li>
    </ul>
    <table>
        <tr>
            <th>Created</th>
            <td>{{order.created}}</td>
        </tr>
        <tr>
            <th>Customer</th>
            <td>{{order.first_name}} {{order.last_name}}</td>
        </tr>
        <tr>
            <th>E-mail</th>
            <td>{{order.email}}</td>
        </tr>
        <tr>
            <th>Address</th>
            <td>{{order.address}} {{order.postal_code}} {{order.city}}</td>
        </tr>
        <tr>
            <th>Total amount</th>
            <td>{{order.get_total_price}}</td>
        </tr>
    </table>
{% endblock %}
```
- order/templates/order/admin/pdf.html
```HTML {.line-numbers}
<html>
<body>
    <h1>Django onlineshop</h1>
    <p>
        Invoice no. {{ order.id }}</br>
        <span class="secondary">{{ order.created|date:"M d, Y" }}</span>
    </p>

    <h3>{% if order.paid %}Payment Accepted{% else %}Pending payment{% endif %}</h3>
    <p>
        {{ order.first_name }} {{ order.last_name }}<br>
        {{ order.email }}<br>
        {{ order.address }}<br>
        {{ order.postal_code }}, {{ order.city }}
    </p>

    <h3>Product List</h3>
    <table>
        <thead>
            <tr>
                <th>Product</th>
                <th>Price</th>
                <th>Quantity</th>
                <th>Cost</th>
            </tr>
        </thead>
        <tbody>
        {% for item in order.items.all %}
            <tr class="row{% cycle "1" "2" %}">
                <td>{{ item.product.name }}</td>
                <td class="num">${{ item.price }}</td>
                <td class="num">{{ item.quantity }}</td>
                <td class="num">${{ item.get_item_price }}</td>
            </tr>
        {% endfor %}
            {% if order.coupon %}
            <tr class="discount">
                <td colspan="3">Discount</td>
                <td class="num">${{ order.discount }}</td>
            </tr>
            {% endif %}
            <tr class="total">
                <td colspan="3">Total</td>
                <td class="num">${{ order.get_total_price }}</td>
            </tr>
        </tbody>
    </table>
</body>
</html>
```
- static/css/style.css
```css {.line-numbers}
body {
    backgroud:black;
}
```

- static/css/pdf.css
```css {.line-numbers}
body {
    font-family:Helvetica, sans-serif;
    color:#222;
    line-height:1.5;
}

table {
    width:100%;
    border-spacing:0;
    border-collapse: collapse;
    margin:20px 0;
}

table th, table td {
    text-align:left;
    font-size:14px;
    padding:10px;
    margin:0;
}

tbody tr:nth-child(odd) {
    background:#efefef;
}

thead th, tbody tr.discount {
    background:#46a500;
    color:#fff;
    font-weight:bold;
}

thead th, tbody tr.total {
    background:#5993bb;
    color:#fff;
    font-weight:bold;
}

h1 {
    margin:0;
}

.secondary {
    color:#bbb;
    margin-bottom:20px;
}

.num {
    text-align:right;
}
```
#### 6.9.9 weasyprint 설치
- 맥(Mac OS)에 설치하는 방법은 교재 참고
- 윈도우 기준 설치 방법
- 파이참 터미널 창에서 pip 최신화 및 weasyprint 설치
```HTML {.line-numbers}
# 'conda install weasyprint' 명령으로는 설치 실패하므로, pip 설치 시도 
$ conda update pip        # 콘다 환경에서 pip 최신화
$ pip install weasyprint  # pip 명령으로 설치
$ conda list weasy        # 콘다에서 설치 확인하면 51 버전
```
- GTK+ 라이브러리 설치  
    - [MSYS2 사이트](https://www.msys2.org)에서 MSYS2 다운로드하고, 안내에 따라서 설치
    - 윈도 시작메뉴에서 'MSYS2 64bit' 폴더 내부의 'MSYS2 MinGW 64-bit' 실행하고 다음과 같이 GTK+ 설치
    `$ pacman -S mingw-w64-x86_64-gtk3`
    - 윈도 시스템 변수 path의 최상위 항목으로 'C:\Program Files\GTK3-Runtime Win64\bin' 등록
![ch06_200 pacman](https://user-images.githubusercontent.com/10287629/83487808-af60a000-a4e6-11ea-9490-90fedb4ec6a9.png)

#### 6.9.10 관리자 페이지 테스트 런
<그림> 관리자 페이지에서 주문 목록 확인
![ch06_250admin](https://user-images.githubusercontent.com/10287629/83487804-ae2f7300-a4e6-11ea-9c6d-5a23b23959b7.png)
<그림> 관리자 페이지에서 주문 상세 확인
![ch06_251admin_detail](https://user-images.githubusercontent.com/10287629/83487805-aec80980-a4e6-11ea-9f6d-b549d31b75ac.png)
<그림> 관리자 페이지에서 주문 내역을 PDF 파일로 확인 및 내려받기
![ch06_252admin_detail](https://user-images.githubusercontent.com/10287629/83487807-af60a000-a4e6-11ea-867b-0084655a2e5f.png)

#### 6.9.11 컨텍스트 프로세서 작성
- 상단 메뉴의 장바구니 정보를 어느 페이지에서도 확인할 수 있도록 조치
- cart/context_processors.py 
```PYTHON {.line-numbers}
from .cart import Cart

def cart(request):
    cart = Cart(request)  
    return {'cart':cart}  # 장바구니 정보를 사전형으로 반환 
```
- config/settings.py 파일에 컨텍스트 프로세서 추가 등록 
```PYTHON {.line-numbers}
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
                'cart.context_processors.cart',                         # !!!
            ],
        },
    },
]
```
### 6.10 파이썬애니웨어 서버 배포
- 파이썬애니웨어 서버에서 가상환경 설치를 위한 BASE_DIR\req.txt 파일 
``` {.line-numbers}
django==3.0.5
django-allauth==0.41.0
django-disqus==0.5
django-extensions==1.9.7
pillow==7.1.1
pymysql==0.9.3
weasyprint==51
```
<그림> 깃허브 저장소에 push된 모습
![ch06_280_git](https://user-images.githubusercontent.com/10287629/83502462-94008f80-a4fc-11ea-8867-93edbc402d70.png)

<그림> 파이썬애니웨어 서버에 pull된 모습
![ch06_281_pa](https://user-images.githubusercontent.com/10287629/83502894-39b3fe80-a4fd-11ea-8d6d-9e59a25c871a.png)

- 파이썬애니웨어 서버에서 가상환경 생성
- 파이썬애니웨어 서버에서 가상환경에 패키지 일괄 설치
```bash
~/shop $ virtualenv venv --python=python3.8  # shop 폴더 내부에서 생성
~/shop $ source venv/bin/activate            # 가상환경 활성화
~/shop $ pip -r req.txt
```
- 파이썬애니웨어 서버 Databases 메뉴에서 MySQL 설정
![ch06_300_padb_init](https://user-images.githubusercontent.com/10287629/83510495-104ca000-a508-11ea-8847-35283dfc5196.png)

- config/settings.py 코드에서 DB 설정 수정
아래 그림에서 'OPTIONs' 부분은 주석 처리하지 않기를 권장함
![ch06_310_padb_settings](https://user-images.githubusercontent.com/10287629/83510722-6de0ec80-a508-11ea-970f-93e85868a3e2.png)

- 파이썬애니웨어 Web 탭에서 설정
![ch06_320_pa_web](https://user-images.githubusercontent.com/10287629/83510955-d4fea100-a508-11ea-8a0c-c62da74f12d3.png)

- 파이썬애니웨어 Web 탭의 WSGI configuration file 편집
(13번 행의 빈 줄이 있도록!)
![ch06_320_pa_wsgi](https://user-images.githubusercontent.com/10287629/83511178-39216500-a509-11ea-8f0c-8b8f68e74549.png)
<그림> 상품 진열된 모습
![ch06_400_cart](https://user-images.githubusercontent.com/10287629/83512169-cfa25600-a50a-11ea-9bee-ef5d1918b92d.png)
<그림> 카트에서 쿠폰 할인 적용된 모습
![ch06_410_cart](https://user-images.githubusercontent.com/10287629/83512178-d16c1980-a50a-11ea-9910-dec71aaee94b.png)
<그림> 결제 확정하는 모습
![ch06_420_checkoutt](https://user-images.githubusercontent.com/10287629/83512194-d4ffa080-a50a-11ea-8aab-c3f16aa127a6.png)

![djGirls](https://user-images.githubusercontent.com/10287629/83512531-4c353480-a50b-11ea-848c-495356104ff2.png)