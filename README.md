# django_clustering
이번 글에서는
apache, django, mariaDB, google cloud storage를 이용하여 WAS(django)를 다중화 해보자.

아래의 프로젝트에서 우리는 아파치-장고-마리아db로 서버를 구축했다.
*https://github.com/hanhunh89/insta_clone

먼저 한개의 서버에 아파치, 장고, 마리아디비를 한번에 구축했다.

서비스가 늘어나면서 web-was-db 를 세개의 서버로 분리했다.

이제 또다시 서비스가 늘고있다. 

django를 다중화해보자

이번 글에서 알수있는것

1. apache로 django 다중화
2. 다중화된 django의 session clustering
3. 다중화된 django 서비스를 위한 google cloud storage 구성.
