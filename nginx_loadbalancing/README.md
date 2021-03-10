# Nginx Loadbalancing Test

nginx로 로드밸런싱을 테스트하는 예제

### 파일구조
- flask
    - route: 웹서버 호출 시 redirect되는 부분
        - __init__.py: init을 꼭 넣어줘야 함
        - app_route.py
    - app.py: flask 구동 부분
    - Dockerfile
    - requirements.txt
    - uwsgi.ini: nginx와 flask간에 전달을 위해서 핋요함(자세한 거는 찾아볼 것)

- nginx
    - default.conf: ngnix 설정
    - Dockerfile
- docker-compose.yml: nginx, flask를 한꺼번에 실행시키기 위함

### Container 생성 및 실행
```shell script
# flask1,2,3 버전마다 port번호와 This is Server One 구문을 변경함
# 두 개의 flask Image를 만듬

docker build -t flask1 .
docker build -t flask2 .

docker build -t nginx_server .

cd ..
docker-compose up
```
Web Browser에서 localhost:80로 접속을 하며 'LoadBalancing이 되는 것을 확인할 수 있음'

### Nginx Conf
```shell script
upstream flask_uwsgi { 
    # 연결할 서버에 대한 정의
    # 가중치를 줘서 LoadBalancing을 할 수 있음
    server flask1:5001; 
    server flask2:5002;
}

server {
    # Listening할 Port
    listen 80;
    server_name 127.0.0.1;

    location / {
      # uwsgi 명시
      include uwsgi_params;
      uwsgi_pass flask_uwsgi;
    }

}
```
