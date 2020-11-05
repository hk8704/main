![image](https://user-images.githubusercontent.com/70673885/97950284-bcf1bd00-1dd9-11eb-8c8a-b3459c710849.png)


# 서비스 시나리오

기능적 요구사항
1. 고객이 APP에서 폰을 주문한다.
1. 고객이 결제한다.
1. 주문이 되면 주문 내역이 대리점에 전달된다.
1. 대리점에 주문 정보가 도착하면 배송한다.
1. 배송이 되면 APP에서 배송상태를 조회할 수 있다.
1. 고객이 주문을 취소할 수 있다.
1. 주문이 취소되면 결제가 취소된다.
1. 고객이 결제상태를 APP에서  조회 할 수 있다.
1. 고객이 모든 진행내역을 볼 수 있어야 한다.

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다.> Sync 호출
    1. 주문이 취소되면 결제가 취소되고 주문정보에 업데이트가 되어야 한다.> SAGA, 보상 트랜젝션
1. 장애격리
    1. 대리점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다.> Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 주문을 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다> Circuit breaker, fallback
1. 성능
    1. 고객이 모든 진행내역을 조회 할 수 있도록 성능을 고려하여 별도의 view로 구성한다.> CQRS


추가 기능적 요구사항
1. 결제가 완료 마케팅에서 마일리지가 발급된다.
1. 포인트 발급이 완료되면 고객이 마일리지를 조회 할 수 있다.
1. 결제가 취소되면 마일리지 발급이 취소된다.
1. 마일리지 발급이 취소되면 고객이 마일리지를 조회 할 수 있다.

추가 비기능적 요구사항
1. 트랜잭션
    1. 결제취소가 되면 반드시 마일리지 발급이 취소 되어야 한다.> Sync 호출
    1. 결제가 완료되면 마일리지가 발급되고 주문정보에 업데이트가 되어야 한다.> SAGA, 보상 트랜젝션
1. 장애격리
    1. 마케팅관리 기능이 수행되지 않더라도 포인트 발급 요청은 365일 24시간 받을 수 있어야 한다.> Async (event-driven), Eventual Consistency
    1. 마케팅관리 시스템이 과중되면 결제 취소를 잠시후에 하도록 유도한다> Circuit breaker, fallback
1. 성능
    1. 고객이 모든 진행내역과 포인트 발급 내역을 조회 할 수 있도록 성능을 고려하여 별도의 view로 구성한다.> CQRS


# 체크포인트

1. Saga
1. CQRS
1. Correlation
1. Req/Resp
1. Gateway
1. Deploy/ Pipeline
1. Circuit Breaker
1. Autoscale (HPA)
1. Zero-downtime deploy (Readiness Probe)
1. Config Map/ Persistence Volume
1. Polyglot
1. Self-healing (Liveness Probe)


# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684159-3543c700-826a-11ea-8d5f-a3fc0c4cad87.png)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/8NDyL2Ej0kVOxpa23nqi8JXGk4Z2/mine/bd00984f1c0bdc41fd6f6bab6dd2e9a8/-MLNjkrdWBV7HaZxupPp


### 이벤트 도출
![image](https://user-images.githubusercontent.com/70673885/97949704-dc87e600-1dd7-11eb-9525-544b2411cc51.png)

### 부적격 이벤트 탈락
![image](https://user-images.githubusercontent.com/70673885/97949767-0a6d2a80-1dd8-11eb-8c2f-fa445fa61418.png)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
	- 폰종류가선택됨, 결제버튼클릭됨, 배송수량선택됨, 배송일자선택됨  :  UI 의 이벤트이지, 업무적인 의미의 이벤트가 아니라서 제외
	- 배송취소됨, 메시지발송됨  :  계획된 사업 범위 및 프로젝트에서 벗어서난다고 판단하여 제외
	- 주문정보전달됨  :  주문됨을 선택하여 제외
	
### 추가 이벤트 도출	
![image](https://user-images.githubusercontent.com/70673885/98251864-a061aa80-1fbc-11eb-8436-907a944f4f15.png)

### 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/73699193/97982030-82f2dc00-1e16-11eb-821d-27351387f8ad.png)

### 추가 커멘드 (액터는 없음)
![image](https://user-images.githubusercontent.com/70673885/98252579-765cb800-1fbd-11eb-9e65-243e9033e780.png)

### 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/73699193/97982108-a158d780-1e16-11eb-9270-6e9646268fd1.png)

    - 주문, 대리점관리, 결제 어그리게잇을 생성하고 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌
    
### 추가된 이벤트와 커맨드 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/70673885/98252758-a1dfa280-1fbd-11eb-95f1-70f3d3f39dbd.png)

### 바운디드 컨텍스트로 묶기
![image](https://user-images.githubusercontent.com/70673885/98253633-a193d700-1fbe-11eb-9f6d-6118a9784728.png)

    - 도메인 서열 분리 
        - Core Domain:  app(front), store : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 app 의 경우 1주일 1회 미만, store 의 경우 1개월 1회 미만
        - Supporting Domain:  customer(view), marketing : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:  pay : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![image](https://user-images.githubusercontent.com/73699193/97982278-e3821900-1e16-11eb-97f4-fa2f59fc7ae0.png)

### 폴리시의 이동

![image](https://user-images.githubusercontent.com/73699193/97982413-19bf9880-1e17-11eb-9720-cd82cf1060ff.png)

### 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![image](https://user-images.githubusercontent.com/73699193/97982527-45428300-1e17-11eb-8641-b658bab34fc6.png)

    - 컨텍스트 매핑하여 묶어줌.
    - 팀원 중 외국인이 투입되어 유비쿼터스 랭귀지인 영어로 변경	

### 완성된 모형

![image](https://user-images.githubusercontent.com/73699193/97982584-60ad8e00-1e17-11eb-8fb6-af87b7c6ff91.png)

    - View Model 추가

### 기능적 요구사항 검증

![image](https://user-images.githubusercontent.com/73699193/97982759-96527700-1e17-11eb-9144-f95de1e0d01e.png)

   	- 고객이 APP에서 폰을 주문한다. (ok)
   	- 고객이 결제한다. (ok)
	- 결제가 되면 주문 내역이 대리점에 전달된다. (ok)
	- 대리점에 주문 정보가 도착하면 배송한다. (ok)
	- 배송이 되면 APP에서 배송상태를 조회할 수 있다. (ok)

![image](https://user-images.githubusercontent.com/73699193/97982841-b2eeaf00-1e17-11eb-9f09-9b74f85a96ca.png)

	- 고객이 주문을 취소할 수 있다. (ok)
	- 주문이 취소되면 결제가 취소된다. (ok)
	- 고객이 결제상태를 APP에서  조회 할 수 있다. (ok)

![image](https://user-images.githubusercontent.com/73699193/97982928-d3b70480-1e17-11eb-957e-6a9093d2a0d7.png)
  
	- 고객이 모든 진행내역을 볼 수 있어야 한다. (ok)


### 비기능 요구사항 검증

![image](https://user-images.githubusercontent.com/73699193/97983019-f6e1b400-1e17-11eb-86ef-d43873ccbb7d.png)

    - 1) 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다. (Req/Res)
    - 2) 대리점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다. (Pub/sub)
    - 3) 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다. (Circuit breaker)
    - 4) 주문이 취소되면 결제가 취소되고 주문정보에 업데이트가 되어야 한다.  (SAGA, 보상트렌젝션)
    - 5) 고객이 모든 진행내역을 조회 할 수 있도록 성능을 고려하여 별도의 view로 구성한다. (CQRS, DML/SELECT 분리)


## 헥사고날 아키텍처 다이어그램 도출 (Polyglot)

![image](https://user-images.githubusercontent.com/73699193/98181638-162b2f00-1f47-11eb-81af-0b71ff811e1c.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
    - 대리점의 경우 Polyglot 검증을 위해 Hsql로 셜계


# 구현:

서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd app
mvn spring-boot:run

cd pay
mvn spring-boot:run 

cd store
mvn spring-boot:run  

cd customer
mvn spring-boot:run  
```

## DDD 의 적용

각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 app 마이크로 서비스). 
이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 
하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. 
(Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

![image](https://user-images.githubusercontent.com/73699193/98182350-e2e99f80-1f48-11eb-825c-da099795fe29.png)

Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

![image](https://user-images.githubusercontent.com/73699193/98182486-378d1a80-1f49-11eb-8e14-0de7296978b5.png)


## 폴리글랏 퍼시스턴스
대리점의 경우 H2 DB인 주문과 결제와 달리 Hsql으로 구현하여 MSA간 서로 다른 종류의 DB간에도 문제 없이 동작하여 다형성을 만족하는지 확인하였다. 


app, pay, customer의 pom.xml 설정

![image](https://user-images.githubusercontent.com/73699193/97972993-baf32280-1e08-11eb-8158-912e4d28d7ea.png)


store의 pom.xml 설정

![image](https://user-images.githubusercontent.com/73699193/97973735-e0346080-1e09-11eb-9636-605e2e870fb0.png)



## Gateway 적용

gateway > applitcation.yml 설정

![image](https://user-images.githubusercontent.com/73699193/98060621-5d54e980-1e8d-11eb-943c-692c5953c6a1.png)

gateway 테스트

```
http POST http://gateway:8080/orders item=test qty=1
```
![image](https://user-images.githubusercontent.com/73699193/98183284-2d6c1b80-1f4b-11eb-90ad-c95c4df1f36a.png)



## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문(app)->결제(pay) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 FeignClient 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 
```
# (app) external > PaymentService.java

package phoneseller.external;

@FeignClient(name="pay", url="${api.pay.url}")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void pay(@RequestBody Payment payment);

}
```
![image](https://user-images.githubusercontent.com/73699193/98065833-b1190000-1e98-11eb-9e44-84d4961011ed.png)


- 주문을 받은 직후 결제를 요청하도록 처리
```
# (app) Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

       phoneseller.external.Payment payment = new phoneseller.external.Payment();
        payment.setOrderId(this.getId());
        payment.setProcess("Ordered");
        
        AppApplication.applicationContext.getBean(phoneseller.external.PaymentService.class)
            .pay(payment);
    }
```
![image](https://user-images.githubusercontent.com/73699193/98066539-a6f80100-1e9a-11eb-8dd8-bf213d90e5fb.png)

- 동기식 호출이 적용되서 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:

```
#결제(pay) 서비스를 잠시 내려놓음 (ctrl+c)

#주문하기(order)
http http://localhost:8081/orders item=note20 qty=1   #Fail
```
![image](https://user-images.githubusercontent.com/73699193/98072284-04934a00-1ea9-11eb-9fad-40d3996e109f.png)

```
#결제(pay) 서비스 재기동
cd pay
mvn spring-boot:run

#주문하기(order)
http http://localhost:8081/orders item=note21 qty=2   #Success
```
![image](https://user-images.githubusercontent.com/73699193/98074359-9f8e2300-1ead-11eb-8854-0449a65ff55c.png)



## 비동기식 호출 / 시간적 디커플링 / 장애격리 


결제(pay)가 이루어진 후에 대리점(store)으로 이를 알려주는 행위는 비 동기식으로 처리하여 대리점(store)의 처리를 위하여 결제주문이 블로킹 되지 않아도록 처리한다.
 
- 결제승인이 되었다(payCompleted)는 도메인 이벤트를 카프카로 송출한다(Publish)
 
![image](https://user-images.githubusercontent.com/73699193/98075277-6f478400-1eaf-11eb-88c8-2b4a7736e56b.png)


- 대리점(store)에서는 결제승인(payCompleted) 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.
- 주문접수(OrderReceive)는 송출된 결제승인(payCompleted) 정보를 store의 Repository에 저장한다.:
 
![image](https://user-images.githubusercontent.com/73699193/98076059-e0d40200-1eb0-11eb-94ad-c4ea114cb3aa.png)


대리점(store)시스템은 주문(app)/결제(pay)와 완전히 분리되어있으며(sync transaction 없음), 이벤트 수신에 따라 처리되기 때문에, 대리점(store)이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다.(시간적 디커플링):
```
# 대리점(store) 서비스를 잠시 내려놓음 (ctrl+c)

#주문하기(order)
http http://localhost:8081/orders item=note30 qty=2  #Success

#주문상태 확인
http get http://localhost:8081/orders    # 상태값이 'Shipped'이 아닌 'Payed'에서 멈춤을 확인
```
![image](https://user-images.githubusercontent.com/73699193/98078301-2b577d80-1eb5-11eb-9d89-7c03a3fa27dd.png)
```
#대리점(store) 서비스 기동
cd store
mvn spring-boot:run

#주문상태 확인
http get http://localhost:8081/orders     # 'Payed' 였던 상태값이 'Shipped'로 변경된 것을 확인
```
![image](https://user-images.githubusercontent.com/73699193/98078837-2cd57580-1eb6-11eb-8850-a8c621410d61.png)

# 운영

## Deploy / Pipeline

- 네임스페이스 만들기
```
kubectl create ns phone82
kubectl get ns
```
![image](https://user-images.githubusercontent.com/73699193/97960790-6d20ef00-1df5-11eb-998d-d5591975b5d4.png)

- 폴더 만들기, 해당폴더로 이동
```
mkdir phone82
cd phone 82
```
![image](https://user-images.githubusercontent.com/73699193/97961127-0ea84080-1df6-11eb-81b3-1d5e460d4c0f.png)

- 소스 가져오기
```
git clone https://github.com/phone82/app.git
```
![image](https://user-images.githubusercontent.com/73699193/98089346-eb4cc680-1ec5-11eb-9c23-f6987dee9308.png)

- 빌드하기
```
cd app
mvn package -Dmaven.test.skip=true
```
![image](https://user-images.githubusercontent.com/73699193/98089442-19320b00-1ec6-11eb-88b5-544cd123d62a.png)

- 도커라이징: Azure 레지스트리에 도커 이미지 푸시하기
```
az acr build --registry admin02 --image admin02.azurecr.io/app:latest .
```
![image](https://user-images.githubusercontent.com/73699193/98089685-6dd58600-1ec6-11eb-8fb9-80705c854c7b.png)

- 컨테이너라이징: 디플로이 생성 확인
```
kubectl create deploy app --image=admin02.azurecr.io/app:latest -n phone82
kubectl get all -n phone82
```
![image](https://user-images.githubusercontent.com/73699193/98090560-83977b00-1ec7-11eb-9770-9cfe1021f0b4.png)

- 컨테이너라이징: 서비스 생성 확인
```
kubectl expose deploy app --type="ClusterIP" --port=8080 -n phone82
kubectl get all -n phone82
```
![image](https://user-images.githubusercontent.com/73699193/98090693-b80b3700-1ec7-11eb-959e-fc0ce94663aa.png)

- pay, store, customer, gateway에도 동일한 작업 반복




-(별첨)deployment.yml을 사용하여 배포 

- deployment.yml 편집
```
namespace, image 설정
env 설정 (config Map) 
readiness 설정 (무정지 배포)
liveness 설정 (self-healing)
resource 설정 (autoscaling)
```
![image](https://user-images.githubusercontent.com/73699193/98092861-8182eb80-1eca-11eb-87c5-afa22140ebad.png)

- deployment.yml로 서비스 배포
```
cd app
kubectl apply -f kubernetes/deployment.yml
```

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 단말앱(app)-->결제(pay) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml
feign:
  hystrix:
    enabled: true
    
hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```
![image](https://user-images.githubusercontent.com/73699193/98093705-a166df00-1ecb-11eb-83b5-f42e554f7ffd.png)

* siege 툴 사용법:
```
 siege가 생성되어 있지 않으면:
 kubectl run siege --image=apexacme/siege-nginx -n phone82
 siege 들어가기:
 kubectl exec -it pod/siege-5c7c46b788-4rn4r -c siege -n phone82 -- /bin/bash
 siege 종료:
 Ctrl + C -> exit
```
* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
siege -c100 -t60S -r10 -v --content-type "application/json" 'http://app:8080/orders POST {"item": "abc123", "qty":3}'
```
- 부하 발생하여 CB가 발동하여 요청 실패처리하였고, 밀린 부하가 pay에서 처리되면서 다시 order를 받기 시작 

![image](https://user-images.githubusercontent.com/73699193/98098702-07eefb80-1ed2-11eb-94bf-316df4bf682b.png)

- report

![image](https://user-images.githubusercontent.com/73699193/98099047-6e741980-1ed2-11eb-9c55-6fe603e52f8b.png)

- CB 잘 적용됨을 확인


### 오토스케일 아웃

- 대리점 시스템에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:

```
# autocale out 설정
store > deployment.yml 설정
```
![image](https://user-images.githubusercontent.com/73699193/98187434-44fbd200-1f54-11eb-9859-daf26f812788.png)

```
kubectl autoscale deploy store --min=1 --max=10 --cpu-percent=15 -n phone82
```
![image](https://user-images.githubusercontent.com/73699193/98100149-ce1ef480-1ed3-11eb-908e-a75b669d611d.png)


-
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
kubectl exec -it pod/siege-5c7c46b788-4rn4r -c siege -n phone82 -- /bin/bash
siege -c100 -t120S -r10 -v --content-type "application/json" 'http://store:8080/storeManages POST {"orderId":"456", "process":"Payed"}'
```
![image](https://user-images.githubusercontent.com/73699193/98102543-0d9b1000-1ed7-11eb-9cb6-91d7996fc1fd.png)

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy store -w -n phone82
```
- 어느정도 시간이 흐른 후 스케일 아웃이 벌어지는 것을 확인할 수 있다. max=10 
- 부하를 줄이니 늘어난 스케일이 점점 줄어들었다.

![image](https://user-images.githubusercontent.com/73699193/98102926-92862980-1ed7-11eb-8f19-a673d72da580.png)

- 다시 부하를 주고 확인하니 Availability가 높아진 것을 확인 할 수 있었다.

![image](https://user-images.githubusercontent.com/73699193/98103249-14765280-1ed8-11eb-8c7c-9ea1c67e03cf.png)


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscale 이나 CB 설정을 제거함


- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
kubectl apply -f kubernetes/deployment_readiness.yml
```
- readiness 옵션이 없는 경우 배포 중 서비스 요청처리 실패

![image](https://user-images.githubusercontent.com/73699193/98105334-2a394700-1edb-11eb-9633-f5c33c5dee9f.png)


- deployment.yml에 readiness 옵션을 추가 

![image](https://user-images.githubusercontent.com/73699193/98107176-75ecf000-1edd-11eb-88df-617c870b49fb.png)

- readiness적용된 deployment.yml 적용

```
kubectl apply -f kubernetes/deployment.yml
```
- 새로운 버전의 이미지로 교체
```
cd acr
az acr build --registry admin02 --image admin02.azurecr.io/store:v4 .
kubectl set image deploy store store=admin02.azurecr.io/store:v4 -n phone82
```
- 기존 버전과 새 버전의 store pod 공존 중

![image](https://user-images.githubusercontent.com/73699193/98106161-65884580-1edc-11eb-9540-17a3c9bdebf3.png)

- Availability: 100.00 % 확인

![image](https://user-images.githubusercontent.com/73699193/98106524-c152ce80-1edc-11eb-8e0f-3731ca2f709d.png)



## Config Map

- apllication.yml 설정

* default쪽

![image](https://user-images.githubusercontent.com/73699193/98108335-1c85c080-1edf-11eb-9d0f-1f69e592bb1d.png)

* docker 쪽

![image](https://user-images.githubusercontent.com/73699193/98108645-ad5c9c00-1edf-11eb-8d54-487d2262e8af.png)

- Deployment.yml 설정

![image](https://user-images.githubusercontent.com/73699193/98108902-12b08d00-1ee0-11eb-8f8a-3a3ea82a635c.png)

- config map 생성 후 조회
```
kubectl create configmap apiurl --from-literal=url=http://pay:8080 --from-literal=fluentd-server-ip=10.xxx.xxx.xxx -n phone82
```
![image](https://user-images.githubusercontent.com/73699193/98107784-5bffdd00-1ede-11eb-8da6-82dbead0d64f.png)

- 설정한 url로 주문 호출
```
http POST http://app:8080/orders item=dfdf1 qty=21
```

![image](https://user-images.githubusercontent.com/73699193/98109319-b732cf00-1ee0-11eb-9e92-ad0e26e398ec.png)

- configmap 삭제 후 app 서비스 재시작
```
kubectl delete configmap apiurl -n phone82
kubectl get pod/app-56f677d458-5gqf2 -n phone82 -o yaml | kubectl replace --force -f-
```
![image](https://user-images.githubusercontent.com/73699193/98110005-cf571e00-1ee1-11eb-973f-2f4922f8833c.png)

- configmap 삭제된 상태에서 주문 호출   
```
http POST http://app:8080/orders item=dfdf2 qty=22
```
![image](https://user-images.githubusercontent.com/73699193/98110323-42f92b00-1ee2-11eb-90f3-fe8044085e9d.png)

![image](https://user-images.githubusercontent.com/73699193/98110445-720f9c80-1ee2-11eb-851e-adcd1f2f7851.png)

![image](https://user-images.githubusercontent.com/73699193/98110782-f4985c00-1ee2-11eb-97a7-1fed3c6b042c.png)



## Self-healing (Liveness Probe)

- store 서비스 정상 확인

![image](https://user-images.githubusercontent.com/27958588/98096336-fb1cd880-1ece-11eb-9b99-3d704cd55fd2.jpg)


- deployment.yml 에 Liveness Probe 옵션 추가
```
cd ~/phone82/store/kubernetes
vi deployment.yml

(아래 설정 변경)
livenessProbe:
	tcpSocket:
	  port: 8081
	initialDelaySeconds: 5
	periodSeconds: 5
```
![image](https://user-images.githubusercontent.com/27958588/98096375-0839c780-1ecf-11eb-85fb-00e8252aa84a.jpg)

- store pod에 liveness가 적용된 부분 확인

![image](https://user-images.githubusercontent.com/27958588/98096393-0a9c2180-1ecf-11eb-8ac5-f6048160961d.jpg)

- store 서비스의 liveness가 발동되어 13번 retry 시도 한 부분 확인

![image](https://user-images.githubusercontent.com/27958588/98096461-20a9e200-1ecf-11eb-8b02-364162baa355.jpg)

