![image](https://user-images.githubusercontent.com/122003216/223020573-106d30f4-4d8d-45ac-afc3-13dff5160b22.png)

1팀 커피숍

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [커피숍](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)

# 서비스 시나리오

기능적 요구사항
1. 고객이 커피메뉴를 선택하여 주문한다
1. 고객이 결제한다
1. 결제가 완료되면 주문 내역이 커피숍주인에게 전달된다
1. 주인이 확인하여 커피를 제조한다.
1. 고객이 주문을 취소할 수 있다
1. 취소되면 결제가 취소된다.
1. 고객이 주문상태를 중간중간 조회한다
1. 주문상태가 바뀔 때 마다 고객에게 카톡으로 알림을 보낸다.(주문시,취소시,커피제조완료시,결재취소시)

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다  Sync 호출 
1. 장애격리
    1. 상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 자주 상점관리에서 확인할 수 있는 배달상태를 확인할 수 있어야 한다  CQRS
    1. 배달상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다  Event driven





# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: 
*** 이미지 추가해야함


### 완성된 1차 모형

![image](https://user-images.githubusercontent.com/122003216/223299792-5b9e882e-c184-4ea4-ba1d-7f39f98733e7.png)

### 완성본에 대한 기능적 요구사항을 커버하는지 검증
![image](https://user-images.githubusercontent.com/122003216/223300507-04fa6f51-0bf6-4ddf-807b-926ac69237e1.png)


# 구현:

분석/설계 단계에서 도출된 모형에 따라 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 
구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

## 1. Saga
## 2. CQRS
## 3. Compensation & Correlation 
## 4. Request / Response  (제외)
## 5. Circuit Breaker   (제외)
## 6. API Gateway 적용
## 7. Deploy (O) / Pipeline 
## 8. Autoscale (HPA)
## 9. Zero-downtime deploy (Readiness probe)
## 10. Persistence Volume/ConfigMap/Secret

1) Dockerfile  수정
 
     /workspace/team1CapstoneCafe/dashboard/Dockerfile  
    수정

ENTRYPOINT ["java","-Xmx400M","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar","--spring.profiles.active=docker"]
끝에   ,"--spring.config.location=file:/application.yml" 추가 

![image](https://user-images.githubusercontent.com/122003216/223557878-1731b0d3-f4b2-4be3-ab18-f328de850dc3.png)

2) 빌드하여 dashboard:database 라는 이미지명으로 레지스트리에 등록 

cd dashboard
mvn package -B 

docker build -t wisemaninatown/dashboard:database .

![image](https://user-images.githubusercontent.com/122003216/223558455-0927478a-3cb4-4a9b-a32a-ada545b124c2.png)


docker push wisemaninatown/dashboard:database
>>
gitpod /workspace/team1CapstoneCafe/dashboard (main) $ docker push wisemaninatown/dashboard:database
The push refers to repository [docker.io/wisemaninatown/dashboard]
cd22e4ee7702: Pushed 
06c45a7351a8: Pushed 
ca35920ce48a: Layer already exists 
a9711b2e31f2: Layer already exists 
50644c29ef5a: Layer already exists 
database: digest: sha256:79b3d2714960bc2214df35fcacd878e98c77a4bd15097ec90c0ed8c1d9921a4b size: 1370


3) Database 환경정보를 설정하기 위해 deployment.yaml 를 변경한다.
  : 이미지 변경 ,  ports 다음에 env 추가
 ![image](https://user-images.githubusercontent.com/122003216/223598340-9a96eb26-bd92-46b5-94c4-1ab02c9f59e4.png)
   
4) Deployment.yaml 에 별도의 Configuration 을 위한 쿠버네티스 객체인 Secret 스펙을 추가  

![image](https://user-images.githubusercontent.com/122003216/223588108-c66d430a-be82-4037-805e-509ab12cac65.png)

5) 생성된 secret 확인
   kubectl get secrets
![image](https://user-images.githubusercontent.com/122003216/223588646-6758871c-d2f9-4da3-9b1a-2d580c3c89b9.png)


6) 이 Secret을 Order Deployment 에 반영하기 위해  deployment.yaml 의  env: 수정
  ![image](https://user-images.githubusercontent.com/122003216/223588837-38e8978e-d386-4a10-9bd7-0ea3d2aec3f4.png)
                  
7) Database 서비스의 생성 (MySQL) :  mysql-deployment.yaml  ,추가
  ![image](https://user-images.githubusercontent.com/122003216/223589611-91d045de-fdf6-4140-a826-a460dd9834bf.png)

  ![image](https://user-images.githubusercontent.com/122003216/223589801-8aabe9bd-9d06-45c9-9834-ae48f6db3ae8.png)
  
 8) Pod 실행을 확인
  ![image](https://user-images.githubusercontent.com/122003216/223590064-96c39414-0ba9-4b58-9536-c664d9e9c2e6.png)

 9) 새 터미널에서 Pod 에 접속하여 dashboarddb 데이터베이스 공간을 만들어주고 데이터베이스가 잘 동작하는지 확인
   ![image](https://user-images.githubusercontent.com/122003216/223590470-bd6e8272-7495-4cca-87f2-549e2b364852.png)
   ![image](https://user-images.githubusercontent.com/122003216/223590644-3ea74f5d-a1ad-4f5e-b1e1-776993952729.png)
   
   ![image](https://user-images.githubusercontent.com/122003216/223590717-e5479c7b-fffb-4660-b010-0800971ea08f.png)
   
 10) dashboard 마이크로 서비스를 쿠버네티스 DNS 체계내에서 접근가능하게 하기 위해 ClusterIP 로 서비스를 생성.
     dashboard 서비스에서 mysql 접근을 위하여 "mysql"이라는 도메인명으로 접근하고 있으므로, 같은 이름으로 서비스를 만들어줌.
     ( mysql-deployment.yaml 에 내용 추가,  적용 )
     
     ![image](https://user-images.githubusercontent.com/122003216/223591445-72891fce-1e3f-40c6-ad8d-a580a3d4ea97.png)
     
     ![image](https://user-images.githubusercontent.com/122003216/223591521-dcf90225-de52-4789-ba27-ac6ae3fc875c.png)
   
 11) dashboard 마이크로 서비스만을 새로 재기동. ( dashboard po 를 삭제하면 deployment에 의해서 알아서 재시작 )
     ![image](https://user-images.githubusercontent.com/122003216/223592406-156a4341-c1e5-4b23-99c6-e16952003b1d.png)


## 11. Self-healing (liveness probe)
## 12. Apply Service Mesh
## 13. Loggregation / Monitoring


#주문처리
http localhost:8081/orders item=통닭 storeId=1   #Fail
http localhost:8081/orders item=피자 storeId=2   #Fail

#주문처리
http POST :8081/orders productId="10" qty="1" customerId="cdgri" amount="2" status="ordered" pickupTime="20230306150010" orderId=1
( orderId 는 자동증가되지 않으므로 max+1 하여 호출한다. )


