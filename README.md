## [JConsole을 이용한 Apache Kafka Monitoring 방법 - Consumer 구성]

<br>

### [환경 구성도]

![image](https://user-images.githubusercontent.com/30817824/172536067-f8c369c5-9937-4fcd-bba1-8b84d9b17cf7.png)

(refer: https://github.com/freepsw/kafka-metrics-monitoring)

<br><br>


### [중요] Producer와 Consumer를 동일한 서버에 설치할 경우 STG.01, STG.02 과정을 생략하고 STG.03으로 이동

<br><br>

----------------------------------------------------------

<br>

> ##  STG.01 GCP VM Instance 생성

<br>

####  GCP에 접속하여 VM Instance를 생성한다
#### client-01 이라는 이름의 인스턴스를 생성(Seoul Region > e2-standard-4)

![image](https://user-images.githubusercontent.com/30817824/172522602-e2f8c980-8abc-48ab-b9c5-d77c9c969365.png)

<br>

#### Boot Disk는 Centos로 선택

<br>

![image](https://user-images.githubusercontent.com/30817824/172509380-4ad29e2c-00ba-4ed0-aee9-f5ab8a5f5f1b.png)

<br><br>

> ## STG.02. Install & Configure

<br>


#### Java 설치 및 JAVA_HOME 설정
```
sudo yum install -y java

# 현재 OS 설정이 한글인지 영어인지 확인한다. 
alternatives --display java

# 아래와 같이 출력되면 한글임. 
슬레이브 unpack200.1.gz: /usr/share/man/man1/unpack200-java-1.8.0-openjdk-1.8.0.312.b07-1.el7_9.x86_64.1.gz
현재 '최고' 버전은 /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.312.b07-1.el7_9.x86_64/jre/bin/java입니다.

### 한글인 경우 
alternatives --display java | grep '현재 /'| sed "s/현재 //" | sed 's|/bin/java로 링크되어 있습니다||'
export JAVA_HOME=$(alternatives --display java | grep '현재 /'| sed "s/현재 //" | sed 's|/bin/java로 링크되어 있습니다||')

### 영문인 경우
alternatives --display java | grep current | sed 's/link currently points to //' | sed 's|/bin/java||' | sed 's/^ //g'
export JAVA_HOME=$(alternatives --display java | grep current | sed 's/link currently points to //' | sed 's|/bin/java||' | sed 's/^ //g')

# 제대로 java 경로가 설정되었는지 확인
echo $JAVA_HOME
echo "export JAVA_HOME=$JAVA_HOME" >> ~/.bash_profile
source ~/.bash_profile
```

-------------------------------------------------------------------

<br><br>

> ##  STG.03 Run logstash & start monitoring Via JConsole

<br>

#### 1. Run the logstash 
```
## Add Consumer config file 
mkdir ~/logstash_conf
```


> #### vi ~/logstash_conf/consumer.conf
```
input {
    kafka {
      bootstrap_servers => "broker-01:9092"
      group_id => "consumer_group_1"
      topics => ["redpolex-topic"]
      consumer_threads => 1
    }
}

filter {
    sleep {
        time => "1"   # Sleep 1 second
        every => 1   # on every 1th event
    }
}

output {
    stdout{
        codec => rubydebug
    }
}
```

```
## Run logstash as a kafka consumer
> export LS_JAVA_OPTS='-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false 
  -Dcom.sun.management.jmxremote.ssl=false 
  -Dcom.sun.management.jmxremote.port=9997 
  -Dcom.sun.management.jmxremote.rmi.port=9997 
  -Djava.rmi.server.hostname=34.64.200.217'


## thread 1개, batch size 1개로 데이터를 kafka로 전송
## logstash 실행시 -b (batch.size) 옵션을 1로 설정해야, 
## broker에 데이터가 1건이라로 도착하면, logstash에서 consumer thread로 데이터를 전송하고, 이를 화면에 출력한다. 
## -b 128로 하면, broker에서 128건을 가져올 때 까지 기다린 후 화면에 출력함. 
## path.data 옵션 : 각 logstash 프로세스에서 내부적으로 관리하기 위한 데이터를 저장하기 위한 공간 
## 이전 producer에서 이미 default 경로(./data)를 사용하고 있으므로, consumer용 logstash에서 사용하기 위한 경로를 지정한다. 

mkdir ~/data
 ~/logstash-7.15.0/bin/logstash -w 1 -b 1 --path.data ~/data/consumer_data -f ~/logstash_conf/consumer.conf
```

![image](https://user-images.githubusercontent.com/30817824/172538187-01ce8b49-eac9-4118-8f3d-37eee556d76e.png)



#### extra. LS_JAVA_OPT 설정을 logstash 시작시에 기본으로 적용하는 방법
```
vi logstash-7.15.0/config/jvm.options

## 위 파일 내용에 아래 jmx 관련된 내용을 LS_JAVA_OPTS에 추가
-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=9998 -Dcom.sun.management.jmxremote.rmi.port=9998 -Djava.rmi.server.hostname=34.64.200.217
```


#### 2. GCP VPC Network Firewall 설정 (9997 port 허용)

![image](https://user-images.githubusercontent.com/30817824/172518925-0a960d8a-8a56-4d92-9af6-bf24231fcf53.png)

![image](https://user-images.githubusercontent.com/30817824/172518946-6441c4be-f390-4cba-a2b7-b78768e90f4e.png)



#### 3. JConsole에서 확인

- VS Code Terminal에서 jdk/bin dir에 있는 jconsole.exe 실행 (jdk 설치 후)

![image](https://user-images.githubusercontent.com/30817824/172519146-4645b5d8-b14c-4c76-a70f-5f1acd2ca8fe.png)


```
./jconsole.exe
```

#### 7. JConsole 접속 (GCP VM Instance의 External IP + :9997 포트 입력)

![image](https://user-images.githubusercontent.com/30817824/172537907-ec848370-8105-419c-a5ab-ee99adb1ab9b.png)

#### 8. MBeans에서 Kafka Consumer의 각종 metrics들을 확인할 수 있다

![image](https://user-images.githubusercontent.com/30817824/172538057-e6802cf7-5aac-472e-9436-61216f67eac1.png)

