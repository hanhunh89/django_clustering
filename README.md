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

3. 아파치 서버 설정

   virtualhost를 수정합니다. 
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
   이제 virtualhost가 충돌하지 않게 정리합니다.
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

   아파치 서버 재시작
   ```
   sudo service apache2 restart
   ```


->>> apache.conf 설정에 문제가 있어 안된다. 내일 다시 봅시당 빠잉 
   
