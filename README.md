# EliasticSearch 설치

```shell
ubuntu@elk-dev-server:~$ sudo apt-get update
ubuntu@elk-dev-server:~$ sudo apt-get upgrade
ubuntu@elk-dev-server:~$ sudo apt-get install openjdk-11-jdk
ubuntu@elk-dev-server:~$ curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch |sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
ubuntu@elk-dev-server:~$ echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
ubuntu@elk-dev-server:~$ sudo apt update
ubuntu@elk-dev-server:~$ sudo apt install elasticsearch
ubuntu@elk-dev-server:~$ sudo systemctl start elasticsearch
ubuntu@elk-dev-server:~$ sudo systemctl enable elasticsearch
ubuntu@elk-dev-server:~$ curl -X GET "localhost:9200"
```
## 설치 구성
```shell
ubuntu@elk-dev-server:~$ sudo vi /etc/elasticsearch/elasticsearch.yml

network.host: 0.0.0.0
#
# By default Elasticsearch listens for HTTP traffic on the first free port it
# finds starting at 9200. Set a specific HTTP port here:
#
http.port: 9299


sudo systemctl restart elasticsearch
```
- 테스트
```shell
ubuntu@elk-dev-server:~$ curl -X GET "localhost:9299"
{
  "name" : "node-1",
  "cluster_name" : "my-application",
  "cluster_uuid" : "K8_U9rFxROSz2g_RBl4IYQ",
  "version" : {
    "number" : "7.17.12",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "e3b0c3d3c5c130e1dc6d567d6baef1c73eeb2059",
    "build_date" : "2023-07-20T05:33:33.690180787Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```
## 데이터 구성
```shell
ubuntu@elk-dev-server:~$ sudo apt install logstash

ubuntu@elk-dev-server:~$ cd /etc/logstash/conf.d

ubuntu@elk-dev-server:~$ sudo vi elasticsearch-pipeline.conf

input {
  jdbc {
    jdbc_driver_library => "/etc/logstash/mysql-connector-java-8.0.28.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://146.56.131.211:3306/fantoo?useSSL=false&characterEncoding=UTF-8&serverTimezone=Asia/Seoul&autoReconnect=true&autoReconnectForPolls=true&allowMultiQueries=true&allowPublicKeyRetrieval=true"
    jdbc_user => "fantoo_system"
    jdbc_password => "QoQofh!#99"
    jdbc_paging_enabled => true
    tracking_column => "unix_ts_in_secs"
    use_column_value => true
    tracking_column_type => "numeric"
    schedule => "*/5 * * * * *"
    statement => "SELECT *, UNIX_TIMESTAMP(update_date) AS unix_ts_in_secs FROM fantoo.club_post WHERE (UNIX_TIMESTAMP(update_date) > :sql_last_value AND update_date < NOW()) ORDER BY update_date ASC"
    last_run_metadata_path => "/var/lib/logstash/jdbc_last_run_metadata.log"
  }
}
filter {
  mutate {
    copy => { "club_post_id" => "[@metadata][_id]"}
    remove_field => ["id", "@version", "unix_ts_in_secs"]
  }
}
output {
  # stdout { codec =>  "rubydebug"}
  elasticsearch {
      hosts => ["localhost:9299"]
      index => "club_post"
      document_id => "%{[@metadata][_id]}"
  }
}

ubuntu@elk-dev-server:~$ sudo cp /home/ubuntu/mysql-connector-java-8.0.28.jar /etc/logstash/

ubuntu@elk-dev-server:~$ sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t

ubuntu@elk-dev-server:~$ sudo systemctl enable logstash
ubuntu@elk-dev-server:~$ sudo systemctl start logstash

curl -X GET "localhost:9299/club_post/_search"
```
## Troubleshooting
- Index 잘 못 구성되었을 때나 다시 데이터를 가져와서 구성하려면 우선 index가 생성되었으면 데이터를 모두 삭제해야 한다.
- 그리고 나서 다시 데이터를 가져오기 위해서는 위에서 선언한 last_run_metadata_path의 파일을 초기화 한다.
- 그리고 나서 service logstash restart 하면 다시 데이터를 구성한다. 
```shell
ubuntu@elk-dev-server:/var/lib/logstash$ sudo service logstash stop
ubuntu@elk-dev-server:/var/lib/logstash$ sudo vi /var/lib/logstash/jdbc_last_run_metadata.log  -- clear
ubuntu@elk-dev-server:~$ sudo systemctl start logstash

ubuntu@elk-dev-server:/var/lib/logstash$ sudo service logstash start
ubuntu@elk-dev-server:/var/lib/logstash$ sudo service logstash status
● logstash.service - logstash
     Loaded: loaded (/etc/systemd/system/logstash.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2023-09-18 07:03:28 UTC; 3s ago
   Main PID: 130721 (java)
      Tasks: 15 (limit: 19098)
     Memory: 333.7M
        CPU: 10.303s
     CGroup: /system.slice/logstash.service
             └─130721 /usr/share/logstash/jdk/bin/java -Xms1g -Xmx1g -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djdk.io.File.enableADS=true -Djruby.compile.invokedynamic=true -Djruby.jit.th>

Sep 18 07:03:28 elk-dev-server systemd[1]: Started logstash.
Sep 18 07:03:28 elk-dev-server logstash[130721]: Using bundled JDK: /usr/share/logstash/jdk
Sep 18 07:03:28 elk-dev-server logstash[130721]: OpenJDK 64-Bit Server VM warning: Option UseConcMarkSweepGC was deprecated in version 9.0 and will likely be removed in a future release.

ubuntu@elk-dev-server:/var/lib/logstash$ sudo tail -f /var/lib/logstash/logstash-plane.log

[2023-09-18T07:03:46,607][INFO ][logstash.javapipeline    ][main] Pipeline Java execution initialization time {"seconds"=>0.71}
[2023-09-18T07:03:46,656][INFO ][logstash.javapipeline    ][main] Pipeline started {"pipeline.id"=>"main"}
[2023-09-18T07:03:46,732][INFO ][logstash.agent           ] Pipelines running {:count=>1, :running_pipelines=>[:main], :non_running_pipelines=>[]}
[2023-09-18T07:03:51,567][INFO ][logstash.inputs.jdbc     ][main][7c568058db0ca6822664b65ea7827f27c3efc9d8e61dd4b5ff4bb11c99490443] (0.017023s) SELECT version()
[2023-09-18T07:03:51,589][INFO ][logstash.inputs.jdbc     ][main][7c568058db0ca6822664b65ea7827f27c3efc9d8e61dd4b5ff4bb11c99490443] (0.000937s) SELECT version()
[2023-09-18T07:03:51,672][INFO ][logstash.inputs.jdbc     ][main][7c568058db0ca6822664b65ea7827f27c3efc9d8e61dd4b5ff4bb11c99490443] (0.014808s) SELECT count(*) AS `count` FROM (SELECT *, UNIX_TIMESTAMP(update_date) AS unix_ts_in_secs FROM fantoo.club_post WHERE (UNIX_TIMESTAMP(update_date) > 0 AND update_date < NOW()) ORDER BY update_date ASC) AS `t1` LIMIT 1
[2023-09-18T07:03:51,858][INFO ][logstash.inputs.jdbc     ][main][7c568058db0ca6822664b65ea7827f27c3efc9d8e61dd4b5ff4bb11c99490443] (0.166313s) SELECT * FROM (SELECT *, UNIX_TIMESTAMP(update_date) AS unix_ts_in_secs FROM fantoo.club_post WHERE (UNIX_TIMESTAMP(update_date) > 0 AND update_date < NOW()) ORDER BY update_date ASC) AS `t1` LIMIT 100000 OFFSET 0
```
- 데이터가 몇개 없어서 100000 개 안에 들어와서 한번만 수행 될 것이다. 

```shell
ubuntu@elk-dev-server:/var/lib/logstash$ curl -s http://localhost:9299/club_post/_search
```

## Community Post Index 추가하기
```shell
cd /var/lib/logstash
sudo touce com_post_last_run_metadata.log
sudo chown logstash.logstash com_post_last_run_metadata.log
```
### logstash 파일 추가
```shell
ubuntu@elk-dev-server:/var/lib/logstash$ cd /etc/logstash/
ubuntu@elk-dev-server:/etc/logstash$ cd conf.d/
ubuntu@elk-dev-server:/etc/logstash/conf.d$ sudo cp elasticsearch-pipeline.conf elasticsearch-pipeline-com-post.conf
ubuntu@elk-dev-server:/etc/logstash/conf.d$ sudo vi elasticsearch-pipeline-com-post.conf

input {
  jdbc {
    jdbc_driver_library => "/etc/logstash/mysql-connector-java-8.0.28.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://146.56.131.211:3306/fantoo?useSSL=false&characterEncoding=UTF-8&serverTimezone=Asia/Seoul&autoReconnect=true&autoReconnectForPolls=true&allowMultiQueries=true&allowPublicKeyRetrieval=true"
    jdbc_user => "fantoo_system"
    jdbc_password => "QoQofh!#99"
    jdbc_paging_enabled => true
    tracking_column => "unix_ts_in_secs"
    use_column_value => true
    tracking_column_type => "numeric"
    schedule => "*/5 * * * * *"
    statement => "SELECT *, UNIX_TIMESTAMP(create_date) AS unix_ts_in_secs FROM fantoo.com_post WHERE (UNIX_TIMESTAMP(create_date) > :sql_last_value AND create_date < NOW()) ORDER BY create_date ASC"
    last_run_metadata_path => "/var/lib/logstash/com_post_last_run_metadata.log"
  }
}
filter {
  mutate {
    copy => { "com_post_id" => "[@metadata][_id]"}
    remove_field => ["id", "@version", "unix_ts_in_secs"]
  }
}
output {
  # stdout { codec =>  "rubydebug"}
  elasticsearch {
      hosts => ["localhost:9299"]
      index => "com_post"
      document_id => "%{[@metadata][_id]}"
  }
}


ubuntu@elk-dev-server:/etc/logstash/conf.d$ sudo service logstash restart
ubuntu@elk-dev-server:/etc/logstash/conf.d$ sudo service logstash status

ubuntu@elk-dev-server:/etc/logstash/conf.d$ sudo tail -f /var/log/logstash/logstash-plain.log

[2023-09-18T07:20:16,965][INFO ][logstash.inputs.jdbc     ][main][3d9796ff7e61ed4eee759075f577c9b867b3c20cc29c6182174eab83eaf95875] (0.366605s) SELECT * FROM (SELECT *, UNIX_TIMESTAMP(create_date) AS unix_ts_in_secs FROM fantoo.com_post WHERE (UNIX_TIMESTAMP(create_date) > 0 AND create_date < NOW()) ORDER BY create_date ASC) AS `t1` LIMIT 100000 OFFSET 0
```

```shell
ubuntu@elk-dev-server:/var/lib/logstash$ curl -s http://localhost:9299/com_post/_search
```