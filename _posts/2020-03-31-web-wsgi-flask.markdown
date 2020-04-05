---
layout: post
title: "WSGI를 통한 Apache와 Flask 연동"
date: 2020-03-31 00:54:00
author: Juhyeok Bae
categories: WEB
---
# 소개
WSGI는 Web Server Gateway Interface의 약자로 과거 CGI나 fpm과 비슷한 역할을 하는 gateway interface 이다.  
Apache 같은 웹서버와 flask, django로 만든 WAS 사이에 위치하여 서로 통신을 가능하게 해주는 인터페이스 역할을 한다.  
WSGI 또한 프로토콜이기 때문에 웹서버와 WAS 간 규격을 맞춰야 하는데, django와 같은 프레임워크는 WSGI 표준에 맞춰 구현되어 있어 개발자가 WSGI를 처음 부터 구현할 일은 없다.  
보통 웹페이지를 보여주는 앱을 구성할 시 정적인 페이지를 보여주는 웹서버와 동적인 처리를 하는 WAS로 나누게 된다. 서버는 나뉘지만 보통 하나의 URL을 제공하기 때문에 웹서버로 들어온 request가 동적 처리를 위한 것이면 WAS로 전달 해야 한다. 이때 WSGI를 통해 request를 전달할 수 있다.  


# WSGI 종류
- **mod_wsgi**  
  해당 문서에서 다룰 wsgi이다. Graham Dumpletion이 개발한 Apache에서 사용 하는 모듈이며, Apache로 들어온 request를 WSGI를 통해 python app으로 전달 하는 역할을 한다.
- **uWSGI**  
  WSGI 컨테이너 이다. 다른 언어/플랫폼 과도 호환이 잘 되도록 pluggable architecture를 지원 한다. 다만 무겁다는 단점이 있다. 사용시에는 uWSGI 컨테이너 단독으로도 실행 가능하고, nginx와 uWSGI를 연동 하여서 사용할 수 도 있다.
- **Gunicorn**  
  uWSGI 처럼 WSGI 컨테이너 이다. uWSGI는 다소 무거운데 반해 Gunicorn은 좀 더 가볍다는 장점이 있다. 마찬가지 단독으로도 실행 가능하고, nginx와 연동해 사용도 가능하다.

# 설정
1) 패키지 설치  
  apache에서 쓰는 라이브러리 설치 및 모듈 enable이 필요 합니다.
    ```
    # apt-get install libapache2-mod-wsgi-py3
    # a2enmod wsgi
    ```

2) Apache 설정  
  Apache는 정적 처리를 위한 웹서버 이기 때문에 python을 직접 처리 하지 않습니다. 따라서 WSGI를 실행 하기 위해서 스크립트 경로 지정 및 몇몇 설정이 필요 합니다.
    ```
    <VirtualHost *:80>
        ServerName flask.juk.internal
        DocumentRoot /var/www/flask

        WSGIScriptAlias / /var/www/flask/flask.wsgi
        WSGIDaemonProcess myapp user=www-data group=www-data threads=5
        WSGIProcessGroup myapp

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>

    <Directory /var/www/flask/myapp/>
        Order deny,allow
        Allow from all
    </Directory>
    ```

3) WSGI 스크립트 설정  
  아파치에서 WSGIScriptAlias에 적어준 스크립트를 작성 해야 합니다. 이 스크립트는 실제 로직 수행을 위한 기능을 가지고 있는 스크립트가 아닙니다. 로직 수행하는 스크립트의 경로를 지정하여 트리거 하는 역할을 합니다.  
  아래는 /var/www/flask/myapp 에 있는 스크립트를 실행 하기 위한 WSGI 스크립트 입니다.
    ```
    import sys

    sys.path.insert(0, "/var/www/flask/")
    from myapp import app as application
    ```

4) Flask app 구현  
  flask 앱의 위치는 wsgi 스크립트 작성시 적어준 위치에 있어야 수행 됩니다.  
  Python에서 단일 스크립트가 아니라 패키지 형태 여러 파일로 구성된 경우 `__init__.py` 가 제일 먼저 수행 됩니다. 현 예제는 로직을 구현한 파일이 하나 밖에 없긴 하지만 패키지 처럼 사용 하기 위해 `__init__.py`로 파일을 생성 합니다.  
  아래는 간단히 / path로 들어올시 json 결과를 출력하도록 구현한 flask app 입니다.
    ```
    from flask import Flask

    app = Flask(__name__)

    @app.route("/")
    def root_app():
      return "{\"authentication\": \"passed\"}"

    if __name__ == '__main__':
      app.run()
    ```

5) 서비스 재시작 및 확인  
  프로세스 재시작 및 curl로 확인 합니다.  
    ```
    # service apache2 restart
    # curl http://flask.juk.internal
    {"authentication": "passed"}
    ```
