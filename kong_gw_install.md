# Kong Gateway Install
https://legacy-gateway--kongdocs.netlify.app/enterprise/2.4.x/deployment/installation/ubuntu/    

## Download
```
curl -Lo kong-enterprise-edition-3.4.1.1.amd64.deb "https://packages.konghq.com/public/gateway-34/deb/ubuntu/pool/jammy/main/k/ko/kong-enterprise-edition_3.4.1.1/kong-enterprise-edition_3.4.1.1_amd64.deb"

ubuntu@elk-dev-server:~/kong_gw$ curl -Lo kong-enterprise-edition-3.4.1.1.amd64.deb "https://packages.konghq.com/public/gateway-34/deb/ubuntu/pool/jammy/main/k/ko/kong-enterprise-edition_3.4.1.1/kong-enterprise-edition_3.4.1.1_amd64.deb"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 62.6M  100 62.6M    0     0  2995k      0  0:00:21  0:00:21 --:--:-- 9939k
```

## Install
```
sudo apt install -y ./kong-enterprise-edition-3.4.1.1.amd64.deb

ubuntu@elk-dev-server:~/kong_gw$ sudo apt install -y ./kong-enterprise-edition-3.4.1.1.amd64.deb


ubuntu@elk-dev-server:~/kong_gw$ curl -1sLf \
  'https://packages.konghq.com/public/gateway-34/setup.deb.sh' \
  | sudo -E bash

ubuntu@elk-dev-server:~/kong_gw$ sudo apt-get install kong  
```

## UnInstall
```
sudo apt remove kong-enterprise-edition
```

## PostgreSQl
```
sudo apt-get install postgresql postgresql-contrib

ubuntu@elk-dev-server:~/kong_gw$ sudo apt-get install postgresql postgresql-contrib
```
### PostgreSQL launch and User creation
```
sudo -i -u postgres
psql

psql> CREATE USER kong; CREATE DATABASE kong OWNER kong; ALTER USER kong WITH password 'wjdalstjq1!';

psql> \q
exit


ubuntu@elk-dev-server:~/kong_gw$ sudo -i -u postgres
postgres@elk-dev-server:~$

postgres@elk-dev-server:~$ psql
psql (14.9 (Ubuntu 14.9-0ubuntu0.22.04.1))
Type "help" for help.

postgres=#
postgres=# CREATE USER kong; CREATE DATABASE kong OWNER kong; ALTER USER kong WITH password 'wjdalstjq1!';
CREATE ROLE
CREATE DATABASE
ALTER ROLE
postgres=#

```

## Modify Kong Gateway's configuration file
- 디폴트 구성 파일을 카피한다.
```
sudo cp /etc/kong/kong.conf.default /etc/kong/kong.conf
```
- PostgreSQL 데이터베이스 및 사용자를 설정한다. /etc/kong/kong.conf
```
 pg_user = kong
 pg_password = wjdalstjq1!
 pg_database = kong
```

## Seed the Super Admin's password and bootstrap Kong Gateway
Kong을 시작하게 전에 Super Admin의 패스워드를 설정한다. 
- Super Admin의 패스워드를 환경변수로 생성한다 (해당 패스워드는 기억해야 된다). Kong Database를 위해 migrations을 시작한다.
```
sudo KONG_PASSWORD=<password-only-you-know> /usr/local/bin/kong migrations bootstrap -c /etc/kong/kong.conf

ubuntu@elk-dev-server:~/kong_gw$ sudo KONG_PASSWORD=wjdalstjq1! /usr/local/bin/kong migrations bootstrap -c /etc/kong/kong.conf
```
- Start Kong Gateway
```
sudo /usr/local/bin/kong start -c /etc/kong/kong.conf

ubuntu@elk-dev-server:~/kong_gw$ sudo /usr/local/bin/kong start -c /etc/kong/kong.conf
2023/10/18 04:10:10 [warn] ulimit is currently set to "1024". For better performance set it to at least "4096" using "ulimit -n"
2023/10/18 04:10:10 [warn] ulimit is currently set to "1024". For better performance set it to at least "4096" using "ulimit -n"
Kong started
```
- Verify Kong Gateway
```
curl -i -X GET --url http://localhost:8002/services

ubuntu@elk-dev-server:~/kong_gw$ curl -i -X GET --url http://localhost:8002/services
HTTP/1.1 200 OK
Date: Wed, 18 Oct 2023 04:10:40 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
X-Kong-Admin-Request-ID: 8fNd9SYS1gNKKB6LaG3AGzosurRrWJX2
Content-Length: 23
X-Kong-Admin-Latency: 235
Server: kong/3.4.1.1-enterprise-edition
```

## Enable and configure Kong Manager

- GUI Kong manager 접근을 위해 /etc/kong/kong.conf 파일에 admin_gui_url 속성을 수정한다. 
```
 admin_gui_url = http://<DNSorIP>:8002

 admin_gui_url = http://USER-IP:8002
```
외부에서 접근가능한다 DNS or IP를 설정한다.   

- Admin Listen Port를 설정한다.
```
 admin_listen = 0.0.0.0:8002, 0.0.0.0:8445 ssl
```

- 또한 네트워크 인터페이스를 분리적으로 목록화 할 수도 있다.
```
 admin_listen = 0.0.0.0:8001, 0.0.0.0:8444 ssl, 127.0.0.1:8001, 127.0.0.1:8444 ssl
```

- Restart Kong
```
sudo /usr/local/bin/kong restart

ubuntu@elk-dev-server:~/kong_gw$ sudo /usr/local/bin/kong restart
2023/10/18 04:14:46 [warn] ulimit is currently set to "1024". For better performance set it to at least "4096" using "ulimit -n"
2023/10/18 04:14:46 [warn] ulimit is currently set to "1024". For better performance set it to at least "4096" using "ulimit -n"
2023/10/18 04:14:46 [warn] Found dangling unix sockets in the prefix directory ("/usr/local/kong") while preparing to start Kong. This may be a sign that Kong was previously shut down uncleanly or is in an unknown state and could require further investigation.
2023/10/18 04:14:46 [warn] Attempting to remove dangling sockets before starting Kong...
2023/10/18 04:14:46 [warn] removing unix socket: /usr/local/kong/worker_events.sock
Kong started

```

# Kong Test
- Service 생성
```
ubuntu@elk-dev-server:~/kong_gw$ curl -i -X POST http://localhost:8001/services --data name=fapi-dev --data url='https://api-address-url'
HTTP/1.1 201 Created
Date: Wed, 18 Oct 2023 06:32:20 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: http://146.56.151.13:8002
Access-Control-Allow-Credentials: true
Content-Length: 381
X-Kong-Admin-Latency: 9
Server: kong/3.4.2

{"host":"fapi-dev.fantoo.co.kr","enabled":true,"name":"fapi-dev","tags":null,"ca_certificates":null,"write_timeout":60000,"created_at":1697610740,"updated_at":1697610740,"retries":5,"path":null,"protocol":"https","tls_verify":null,"tls_verify_depth":null,"id":"23daaff7-2f0a-429b-9c8b-dcad287518e1","connect_timeout":60000,"read_timeout":60000,"client_certificate":null,"port":443
```
- Route 등록
```
ubuntu@elk-dev-server:~/kong_gw$ curl -i -X POST http://localhost:8001/services/fapi-dev/routes --data 'paths[]=/country' --data name=country
HTTP/1.1 201 Created
Date: Wed, 18 Oct 2023 06:35:07 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: http://USER-IP:8002
Access-Control-Allow-Credentials: true
Content-Length: 482
X-Kong-Admin-Latency: 35
Server: kong/3.4.2

{"snis":null,"request_buffering":true,"response_buffering":true,"created_at":1697610907,"updated_at":1697610907,"path_handling":"v0","service":{"id":"23daaff7-2f0a-429b-9c8b-dcad287518e1"},"protocols":["http","https"],"tags":null,"methods":null,"sources":null,"preserve_host":false,"destinations":null,"hosts":null,"id":"d53c79e1-ef0f-4133-a40f-caaaebd57430","headers":null,"https_redirect_status_code":426,"paths":["/country"],"strip_path":true,"regex_priority":0,"name":"country"}
```

- Route Test
```

```