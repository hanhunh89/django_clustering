# django_clustering
이번 글에서는
apache, django, mariaDB, google cloud storage를 이용하여 WAS(django)를 다중화 해보자.

아래의 프로젝트에서 우리는 아파치-장고-마리아db로 서버를 구축했다.
*https://github.com/hanhunh89/insta_clone

먼저 한개의 서버에 아파치, 장고, 마리아디비를 한번에 구축했다.

서비스가 늘어나면서 web-was-db 를 세개의 서버로 분리했다.

이제 또다시 서비스가 늘고있다. 

django를 다중화해보자

이번 글을 통해서 다음 세가지를 알 수 있다. 
1. apache로 django 다중화
2. 다중화된 django의 session clustering
3. 다중화된 django 서비스를 위한 google cloud storage 구성.

------------------
시작해보자

1. 새로운 was(django) 서버 구성 및 gunicorn 구동
   *https://github.com/hanhunh89/insta_clone 을 참고하여 django를 하나 더 구성한다.

2. 아파치 서버 설정
   ```
   sudo nano /etc/apache2/apache2.conf
   ```
   apache2.conf 파일 맨 밑에 다음 세줄을 첨부합니다.
   ```
   #apache-django clustering by daramzi
   LoadModule proxy_module modules/mod_proxy.so
   LoadModule proxy_http_module modules/mod_proxy_http.so
   LoadModule proxy_balancer_module /usr/lib/apache2/modules/mod_proxy_balancer.so

   <Proxy balancer://my_django_cluster>
     BalancerMember http://34.64.44.173:8000
     BalancerMember http://34.125.160.132:8000
     # 추가적인 django server의 ip를 필요한 만큼 추가합니다.
   </Proxy>
   ```
   mod_proxy_balancer.so 는 위의 두개 모듈과는 다르게 절대경로를 입력했습니다.
   절대경로를 입력하지 않고 moddules~ 만 입력했을 때 모듈을 찾지 못했습니다.
   에러가 나지 않는다면 modules/mod_proxy_balancer.so 만 입력해도 문제 없습니다.

   다음으로 virtualhost를 수정합니다. 
   ```
   cd /etc/apache2/conf-available
   ```
   cluster.conf 파일을 만들고 아래와 같이 입력합니다.
   ```
   <VirtualHost *:80>
     ServerName 34.64.106.7
     ProxyPass / balancer://my_django_cluster/
     ProxyPassReverse / balancer://my_django_cluster/
   </VirtualHost>
   ```
   위의 ServerName은 apache의 도메인(또는 ip)를 입력합니다. 
   
   아파치 서버 재시작
   ```
   sudo service apache2 restart
   ```

   
