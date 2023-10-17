# django_clustering
이번 글에서는
apache, django, mariaDB, google cloud storage를 이용하여 WAS(django)를 다중화 해보자.

이전의 프로젝트에서 우리는 아파치-장고-마리아db로 서버를 구축했다.

*https://github.com/hanhunh89/insta_clone

먼저 한개의 서버에 아파치, 장고, 마리아디비를 한번에 구축했다.

그 다음 사용자가 늘어나면서 web-was-db 를 세개의 서버로 분리했다.

이제 또다시 서비스가 늘고있다. 

django를 다중화해보자

이번 글을 통해서 다음 세가지를 알 수 있다. 
1. apache로 django 다중화
2. 다중화된 django의 session clustering
3. 다중화된 django 서비스를 위한 google cloud storage 구성.

------------------
시작해보자
## WAS서버 구성
### 1. 새로운 django 서버 구성 및 django settings.py 파일에서 mysql 접속 설정

   *https://github.com/hanhunh89/insta_clone 을 참고하여 django를 하나 더 구성한다.

  setting.py파일을 아래와 같이 수정한다. 
  SQLlite 내용을 주석처리하고 mysql 내용을 적용한다. 
  ```
  '''
  #setting for SQLlite
  DATABASES = {
    'default': {
      'ENGINE': 'django.db.backends.sqlite3',
      'NAME': BASE_DIR / 'db.sqlite3',
    }
  }
  #setting for mysql
  '''
  DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'mydatabase',
        'USER': 'myuser',
        'PASSWORD': '12341234',
        'HOST': '34.64.146.23',
        'PORT': '3306',
     }
  }
  ```

### 2. gunicorn 구동
  *https://github.com/hanhunh89/insta_clone 을 참고하여 gunicorn 구동
  ```
  gunicorn --bind 0:8000 myproject.wsgi:application 
  ```

## DB서버 구성
### 1. DB에 새로운 WAS가 접속할 수 있도록 구성한다. 
  db에 접속 후 다음의 쿼리로 db에 접속 가능한 django 서버 ip를 확인한다.
  ```
  mariaDB [myDB]> SELECT user, host FROM mysql.user WHERE user = 'myuser'
  ```
  이전에 insta_clone에서 구성한 한개의 서버만 보일것이다.
  다음의 명령어로 새로 만든 django 서버도 추가해준다.
  ```
  GRANT ALL PRIVILEGES ON myDB.* TO 'myuser'@'34.125.160.132' IDENTIFIED BY 'password123123' WITH GRANT OPTION;
  FLUSH PRIVILEGES;
  ```
  이제 다시 db에 접속가능한 django 서버 ip를 확인하면 두개의 ip가 뜰 것이다.
  ```
  mariaDB [myDB]> SELECT user, host FROM mysql.user WHERE user = 'myuser'
  ```


## WEB 서버 구성 
### 1. 아파치 서버 설정

   virtualhost를 수정합니다.

   *아파치 공식문서를 참고했습니다.
   
   *https://httpd.apache.org/docs/2.4/mod/mod_proxy_balancer.html
   ```
   cd /etc/apache2/conf-available
   ```
   cluster.conf 파일을 만들고 아래와 같이 입력합니다.
   ```
   <VirtualHost *:80>
     ServerName 34.64.106.7
     #ProxyPass / http://34.64.44.173:8000/
     #ProxyPassReverse / http://34.64.44.173:8000/

     ProxyPass / balancer://mycluster/
     ProxyPassReverse / balancer://mycluster/
     Header add Set-Cookie "ROUTEID=.%{BALANCER_WORKER_ROUTE}e; path=/" env=BALANCER_ROUTE_CHANGED
     <Proxy balancer://mycluster>
       BalancerMember http://34.64.44.173:8000 route=1
       #34.64.44.173 -> django 1번서버 ip
       BalancerMember http://34.125.160.132:8000 route=2
       #34.125.160.132 -> django 2번서버 ip
       ProxySet stickysession=ROUTEID
     </Proxy>  
        
     ErrorLog ${APACHE_LOG_DIR}/error.log
     CustomLog ${APACHE_LOG_DIR}/access.log combined
   </VirtualHost>
   ```

### 2. 이제 virtualhost가 충돌하지 않게 정리합니다.
   ```
   cd /etc/apache2/sites-enabled
   ls
   ```
   ls의 결과를 보면 아래의 두개 파일이 보입니다.
   ```
   aaa@apache:/etc/apache2/sites-enabled$ ls
   000-default.conf  my.conf
   ```
   my.conf는 충돌할 우려가 있으므로 해제합니다.
   ```
   sudo a2dissite my.conf
   ```
   다시 ls를 쳐보면 my.conf가 사라졌습니다.

   이제 cluster.conf를 적용합니다.
   ```
   sudo a2ensite cluster.conf
   ```

### 3. 이제 필요한 mod를 enable 해줍니다. 
   ```
   sudo a2enmod proxy
   sudo a2enmod proxy_balancer
   sudo a2enmod lbmethod_byrequests
   sudo a2enmod headers
   sudo a2enmod proxy_http
   ```

### 4. 아파치 서버 재시작
   ```
   sudo service apache2 restart
   ```

## FAIL OVER TEST
   이제 두개의 장고 서버가 클러스터링 되었습니다. 
   하나의 장고서버가 fail되면 자연스럽게 나머지 하나의 서버로 접속합니다.

------------------------------------------------

하지만 이 구성에는 두가지의 문제가 있습니다.

1. 이미지를 was 서버에 저장해서, 이중화된 다른 서버에서 참조가 불가능하다.

2. 세션이 하나의 서버에 sticky하게 묶여있다.

먼저 1번부터 해결합시다. 

브라우저 두개로 각각 게시물을 업로드해 봅시다.
1번 서버와 2번 서버의 화면이 다릅니다. 
같은 사이트인데도 보여주는 이미지가 다르다니... 망했습니다. 


![1번서버 이미지](two_server_has_different_file.jpg)


이것은 각각의 서버가 이미지를 각자의 서버에 저장하기 때문입니다.

1번 서버에 dog.jpg를 올린다고 합시다.

1번 서버에 dog.jpg가 저장되고 dbserver 에 반영됩니다.

2번 서버로 접속하면 db에서는 "dog.jpg가 있어!" 라고 하지만,

2번 서버에는 dog.jpg파일이 없습니다. 

그래서... 파일을 찾을 수 없는 상태가 됩니다.

우린 망했습니다.

이 프로젝트를 다시 살리기 위해

이미지나 동영상 파일 같은 첨부파일은 별도의 스토리지에 저장합니다.

gcp의 클라우드 스토리를 사용해 봅시다.
