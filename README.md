# Container준비
docker run -it --name u1 --rm -v ${HOME}/df:/df oraclelinux:9
  cp lotte.net.crt /etc/pki/ca-trust/source/anchors/lotte.net.crt
  update-ca-trust enable
  update-ca-trust extract
# 1.3. 실습: Kafka 환경 설정 및 기본 사용법
## 실습1: Kafka 설치 및 기본 환경 설정 방법
* 실습 목표
    - i1에 Kafka 클라이언트를 설치하고, s1에 Kafka 서버를 설정하여 기본 환경 구성.
    - Ansible 사용 설치는 다음 쳅터에서 진행
    - 아래와 같이 개별 설치하는 경우 많음. 
* 실습 단계
    - Step 1: i1에 Kafka 클라이언트 설치
        ```bash
        dnf install -y java-1.8.0-openjdk-devel wget
        cd /df
        wget https://dlcdn.apache.org/kafka/3.9.0/kafka_2.12-3.9.0.tgz
        tar -xvf kafka_2.12-3.9.0.tgz
        mv kafka_2.12-3.9.0 /opt/kafka
        echo 'export PATH=$PATH:/opt/kafka/bin' >>  /etc/bashrc
        source  /etc/bashrc
        pip3 install confluent-kafka
        pip3 install kafka-python
        ```
    - Step 2: s1에 Kafka 서버 설치
        ```
        dnf install -y java-1.8.0-openjdk-devel 
        cd /df
        wget https://dlcdn.apache.org/kafka/3.9.0/kafka_2.12-3.9.0.tgz
        tar -xvf kafka_2.12-3.9.0.tgz
        mv kafka_2.12-3.9.0 /opt/kafka
        echo 'export PATH=$PATH:/opt/kafka/bin' >>  /etc/bashrc
        source  /etc/bashrc
        ```

    - Step 3: s1의 환경 설정
        - Kafka 브로커 설정 파일: `/opt/kafka/config/server.properties`에서 다음 설정 확인:
            `broker.id=1`
            `zookeeper.connect=s1:2181`
            `log.dirs=/tmp/kafka-logs`

* 결과 확인
    - Kafka 클라이언트(i1)와 Kafka 서버(s1)의 설치 및 초기 설정 완료.
    * i1에서 확인
        ```bash
        kafka-topics.sh --help
        python3 -c "from confluent_kafka import Producer"     # 에러 안나면 성공
        ```
    * s1에서 확인 
        ```bash
        which kafka-server-start.sh
        ```

## 실습2: Kafka 서버 및 클라이언트 구성
* 실습 목표
    - i1에서 클라이언트를 실행하고 s1에서 서버를 실행하여 메시지 전송 및 소비 환경 구축.
* 실습 단계
    - Step 1: s1에서 Zookeeper 실행
        ```bash
        # Zookeeper 실행
        tmux 
            /opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
            <ctl+b,d>
        ```
    - Step 2: s1에서 Kafka 브로커 실행
        ```bash
        tmux 
            /opt/kafka/kafka-server-start.sh /opt/kafka/config/server.properties
            <ctl+b,d>
        ```
    - Step 3: i1에서 Kafka 클라이언트 실행
        ```bash
        # Topic 생성:
        kafka-topics.sh --create --topic test-topic --bootstrap-server s1:9092
        # Topic 확인:
        kafka-topics.sh --list --bootstrap-server s1:9092
        ```

* 결과 확인
    - s1에서 Zookeeper와 Kafka 서버가 정상적으로 실행되는지 로그 확인. : ls /tmp/kafka-logs
    - i1에서 클라이언트 도구가 정상적으로 동작하는지 확인.

## 실습3: 간단한 Kafka 클러스터 설정 및 테스트
* 실습 목표
    - s1에서 Kafka 클러스터를 설정하고 i1에서 메시지를 테스트.
* 실습 단계
    - Step 1: s1에서 Topic 생성
        - 명령어: `kafka-topics.sh --create --topic test-topic2 --bootstrap-server s1:9092`
    - Step 2: i1에서 메시지 생성
        - Producer 실행: `kafka-console-producer.sh --topic test-topic2 --bootstrap-server s1:9092`
        - 메시지 입력: "Hello Kafka from i1!"
    - Step 3: i1에서 메시지 소비(두번째 창)
        - Consumer 실행: `kafka-console-consumer.sh --topic test-topic2 --from-beginning --bootstrap-server s1:9092`
* 결과 확인
    - s1에서 생성한 Topic이 정상적으로 작동하는지 확인.`ls /tmp/kafka-logs/test-topic2-0/`
    - i1에서 전송한 메시지가 정상적으로 소비되는지 확인.

## 실습4: 간단한 Kafka 프로듀서 및 컨슈머 애플리케이션 구성 및 실행
* 실습 목표
    - 여러 토픽과 파티션을 구성하여 병렬 스트리밍 데이터를 처리.
* 실습 단계
    - Step 1: s1에서 다중 파티션 토픽 생성
        - 명령어: 
        ```bash
        # kafka-topics.sh --delete --topic multi-partition-topic --bootstrap-server s1:9092
        kafka-topics.sh --create --topic multi-partition-topic --partitions 2 --bootstrap-server s1:9092
        ```
    - Step 2: i1에서 프로듀서에서 다중 메시지 전송
        - 명령어: `kafka-console-producer.sh --topic multi-partition-topic --bootstrap-server s1:9092`
        - 메시지 입력:
            ```
            key1:value1
            key2:value2
            key3:value3
            ```
    - Step 3: 다중 컨슈머 그룹 실행
        - 명령어: 
        ```bash
        kafka-console-consumer.sh --topic multi-partition-topic --group group1 --bootstrap-server s1:9092
        ```
        - 다른 터미널에서 동일한 명령어로 실행해 병렬 소비 확인.
* 결과 확인
    - 파티션별로 메시지가 컨슈머 그룹에 할당되어 병렬 처리가 이루어지는지 확인 : 
    ```bash
    # Topic확인
    kafka-topics.sh --describe --topic multi-partition-topic --bootstrap-server s1:9092
    
    # Group1은 모두 전송된 것 확인
    kafka-consumer-groups.sh --describe --group group1 --bootstrap-server s1:9092
    
    # Group2는 Lag발생된 것 확인
    kafka-consumer-groups.sh --describe --group group2 --bootstrap-server s1:9092

    # Group2의 Consumer Group에서 메세지 받음. 
    tmux
        kafka-console-consumer.sh --topic multi-partition-topic --group group1 --bootstrap-server s1:9092
        <ctl+b,d>
    
    # Group2도 모두 전송되어 Lag 사라진 것 확인    
    kafka-consumer-groups.sh --describe --group group2 --bootstrap-server s1:9092
    ```
