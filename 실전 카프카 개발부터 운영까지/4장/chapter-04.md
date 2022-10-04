# 4장 카프카의 내부 동작 원리와 구현
* 카프카의 내부 동작 원리 및 구현에 대한 내용 설명
* 중요: 리플리케이션 동작
    * 일반적인 분산 시스템: 애플리케이션의 고가용성을 위해 내부적으로 리플리케이션 동작, but 구현이 매우 어려움 + 애플리케이션의 성능 저하 일으킴
    * 카프카가 어떻게 해결했을까
* 내용: 리플리케이션 -> 리더, 팔로워의 역할 -> 리더에포크 + 복구 동작 -> 리플리케이션 관련 컨트롤러 및 컨트롤러의 동작 -> 로그, 로그 컴팩션
----
## 카프카 리플리케이션
* 무수히 많은 데이터 파이프라인의 정중앙에 위치하는 메인 허브 역할
![](https://images.ctfassets.net/gt6dp23g0g38/1ntqDwcP1q5VgraE0XUdug/445158bc26777f83d1ac9911219a8ef0/confluent-cloud-networking-intro.jpg)
* 심각한 문제
    * 하드웨어의 문제 / 점검
    * 연결된 전체 데이터 파이프라인에 영향을 미칠 수 있음
* 리플리케이션
    * 카프카의 초기 설계 단계에서부터 하드웨어 이슈 등으로 브로커에서 장애가 발생하더라도 안정적으로 서비스가 운영될 수 있도록 구상된 복제 방식
----
### 카프카 리플리케이션 동작 개요
* 동작 설정: 토픽 생성 시 replication factor 옵션 설정(필수)
    * 생성: 
        ```./bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 3 --partitions 1 --topic peter-test01```
    * 생성 시 직접 브로커에 접속해서 하거나, 접근이 가능하다면 외부에서 `kafka-topics.sh` 파일로 처리
    * 확인
        ``` shell
            ./bin/kafka-topics.sh --bootstrap-server localhost:9092 \
            --topic peter-test01 \
            --describe

            Topic: peter-test01 PartitionCount: 1 ReplicationFactor: 3 Configs: segment.bytes=1073741824 
            Topic: peter-test01 Partition: 0 Leader: 1 Replicas: 1,2,3 Isr: 1,2,3
        ``` 
* 저장 확인
    * producer를 통해 메시지 전송
    ```console
    /usr/local/kafka/bin/kafka-console-producer.sh --boot-server peter-kafka01.foo.bar:9092 \
    --topic peter-test

    > test message1
    ```  
    * 브로커에 저장된 데이터 확인(세그먼트 파일)    
    ```console
    /usr/local/kafka/bin/kafka-dump-log.sh \  
    --print-data-log--files /data/kafka-logs/peter-test01-0/00000000000000000000.log
    ```
    * 세그먼트 파일 내용 확인
        ```console
        Dumping /data/kafka-logs/peter-test01-0/00000000000000000000.log  
        Starting offset: 0
        baseOffset: 0 lastOffset: 0 count: 1 baseSequence: -1 lastSequence: -1 produerId: -1 
        producerEpch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false
        position: 0 CreateTime: 161008070323 size 81 magic: 2 compresscodec: NONE   
        crc: 3417270022 isvalid: true
        | offset: 0 CreateTime: 161008070323 ketysize: -1 valuesize: 13 sequence: -1   
        headerKeys[] payload: test message1
        ``` 
        * 각각 접속해서 확인해보면 동일한 메시지 저장된 것 확인 가능
* 정리
    * 리플리케이션 팩터라는 옵션을 통해 관리자(운영자)가 지정한 수의 리플리케이션을 가질 수 있음
    * N개의 리플리케이션이 있는 경우 N-1개까지의 브로커 장애가 발생하더라도 메시지 손실 없이 안정적으로 메시지를 주고받을 수 있음(ISR에 대한 이해 필요)
-----
### 리더와 팔로워
* topic describe 확인
    ``` shell
        ./bin/kafka-topics.sh --bootstrap-server localhost:9092 \
        --topic peter-test01 \
        --describe

        Topic: peter-test01 PartitionCount: 1 ReplicationFactor: 3 Configs: segment.bytes=1073741824 
        Topic: peter-test01 Partition: 0 Leader: 1 Replicas: 1,2,3 Isr: 1,2,3
    ```
    * Leader, Replicas, Isr 옵션 존재
* Leader: 리플리케이션 중 하나에서 선출됨
    * 모든 RW(Read/Write) 작업이 리더를 통해서만 이루어짐
    * Producer: 리더에게만 메시지를 전송
    * Consumer: 리더를 통해서만 메시지를 전송받음
* Follower
    * 리더에 문제가 발생하거나 이슈가 있을 경우를 대비해 새로운 리더가 될 준비를 해야 함
    * 리더가 메시지를 받았는지 확인 + 새로운 메시지가 있다면 해당 메시지를 리더로부터 복제
* 구조        
![](https://t1.daumcdn.net/cfile/tistory/99B734465B40705E17?download)
----
