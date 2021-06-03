# 1조 > 개인프로젝트 : SirenOrder (Mini)

![Order1](https://user-images.githubusercontent.com/30651085/120064027-3e29dd80-c0a5-11eb-9baf-92aecca2729f.png)

SirenOrder 서비스를 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 프로젝트임

- 체크포인트 : http://labs.msaez.io/#/courses/assessment/running@cloud-final-aws-2nd


# Table of contents

- [예제 - SirenOrder](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
    - [Event Storming 결과](#Event-Storming-결과)
    - [바운디드 컨텍스트](#바운디드-컨텍스트)
    - [기능적 요구사항 검증](#기능적-요구사항을-커버하는지-검증)
    - [비기능적 요구사항 검증](#비기능-요구사항에-대한-검증)
    - [헥사고날 아키텍처 다이어그램 도출](#헥사고날-아키텍처-다이어그램-도출)
       
  - [구현:](#구현)
    - [DDD 의 적용](#ddd-의-적용)   
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#동기식-호출-과-Fallback-처리)
    
  - [운영](#운영)
    - [CI/CD 설정](#CICD-설정)
    - [Kubernetes 설정](#Kubernetes-설정)
    - [ConfigMap](#ConfigMap-설정)
    - [liveness Probe](### 셀프힐링-livenessProbe-설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출/서킷-브레이킹/장애격리)
    - [오토스케일 아웃](#Autoscale-HPA)
    - [무정지 재배포](#Zero-downtime-deploy)
 
 

# 서비스 시나리오

[ 기능적 요구사항 ]
1. 점원이 판매할 상품가격과 상태(Available, SoldOut)을 등록한다
2. 상품이 등록되면 상품상태를 주문DB에 전달한다
3. 점원에게 상품상태 정보를 조회할 수 있는 Report 서비스를 제공한다
4. 고객은 주문할 메뉴를 선택하여 주문한다
5. 주문이 되면 주문DB의 가격과 상품DB의 상품상태(Available, SoldOut)정보를 조회한다.
6. 가격이 0KRW이 아니고 상품상태가 Available할 경우 주문은 완료된다.

[ 비기능적 요구사항 ]
1. 트랜잭션
    1. 상품상태는 상품DB에서 조회한다 :  Sync 호출 
1. 장애격리
    1. Order 서비스가 중단되더라도 상품정보는 365일 24시간 등록할 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 상품등록이 완료되면 Order서비스가 과중되더라도 상품정보는 Order 서비스가 정상화된 이후에 수신한다 Circuit breaker, fallback
1. 성능
    1. 점원은 Report 서비스를 통해서 상품 정보를 확인할 수 있어야 한다  CQRS
    1. 상품상태가 바뀔때마다 고객에게 알림을 줄 수 있어야 한다  Event driven


# 분석/설계

## Event Storming 결과

* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/f2NszwGXcITtKN4MrX4BrDurru12/mine/58d983ae0915873145e4f53e60244278


### 이벤트 도출

![EventList](https://user-images.githubusercontent.com/30651085/120440146-41cba600-c3be-11eb-92ce-14c24896b345.png)

### 바운디드 컨텍스트

1. Event Storming for Team Assignment 

![TeamBoundedContext](https://user-images.githubusercontent.com/30651085/120570079-404bbd80-c452-11eb-8584-152550e21311.png)

2. Event Storming for Personal Assignment

![Modeling](https://user-images.githubusercontent.com/30651085/120438869-d208eb80-c3bc-11eb-85ca-3526468cb807.png)

    - 도메인 서열 분리 
        - Core Domain:  Order, Product : 없어서는 안될 핵심 서비스
        - Supporting Domain: Report : 경쟁력을 내기위한 서비스

### 기능적 요구사항을 커버하는지 검증

    - 점원이 판매할 상품가격과 상태(Available, SoldOut)을 등록한다 (ok)
    - 고객은 주문할 메뉴를 선택하여 주문한다 (OK)
    - 점원에게 상품상태 정보를 조회할 수 있는 Report 서비스를 제공한다 (ok)
    - 고객이 선택한 메뉴에 대해서 주문을 한다 (ok)
    - 주문이 되면 주문DB의 가격과 상품DB의 상품상태(Available, SoldOut)정보를 조회한다 (ok)
    - 가격이 0KRW이 아니고 상품상태가 Available할 경우 주문은 완료된다 ( ok )

  1) 동기식호출 (Publish/Subscribe)

![Sync](https://user-images.githubusercontent.com/30651085/120438894-d8976300-c3bc-11eb-8927-36284464714b.png)

  2) 비동기식호출 (Request/Response)
 
![Async](https://user-images.githubusercontent.com/30651085/120438768-bc93c180-c3bc-11eb-8399-04428903824a.png)

### 비기능 요구사항에 대한 검증

  3) CQRS
 
![CQRS](https://user-images.githubusercontent.com/30651085/120438809-c61d2980-c3bc-11eb-9bca-351cf2e31597.png)

    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
    - 판매 가능 상품 :  판매가 가능한 상품만 주문 메뉴에 노출됨 , ACID 트랜잭션, Request-Response 방식 처리
    - 상품 등록은 오더서비스와 분리운영 :  Product 서비스에서 Order 마이크로서비스로 상품정보등록요청이 전달되는 과정에 있어서 Order 마이크로 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.
    - Product, Report MicroService 트랜잭션:  상품 상태 등의 이벤트를 Kafka를 통해 Async 방식으로 처리


## 헥사고날 아키텍처 다이어그램 도출

![hexa](https://user-images.githubusercontent.com/30651085/120442837-04b4e300-c3c1-11eb-9995-bd95bceb49ca.png)

    - 주문DB에 Order, Product Repository 설계
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8083 이다)

```
cd product
mvn spring-boot:run 

cd order
mvn spring-boot:run 

cd report
mvn spring-boot:run  
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다.
```
package siren;

@Entity
@Table(name="Product_table")
public class Product {

    @Id
    // @GeneratedValue(strategy=GenerationType.AUTO)
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Long id;
    private String status;

    @PostPersist
    public void onPostPersist(){
        CheckedProductStatus checkedProductStatus = new CheckedProductStatus();
        BeanUtils.copyProperties(this, checkedProductStatus);
        checkedProductStatus.publishAfterCommit();
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}


```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package siren;

public interface ProductRepository extends PagingAndSortingRepository<Product, Long>{
    Optional<Product> findById(Long id);
}
```
- 적용 후 REST API 의 테스트
```
# 주문 처리
http POST http://localhost:8081/orders productId=1
http POST http://a2c3105e6832445d988f3dc034dacd5e-831620996.ap-northeast-2.elb.amazonaws.com:8080/orders productId=1

# 상품 상태 변경 처리
http PATCH http://localhost:8082/products/1 status=SoldOut
http PATCH http://a2c3105e6832445d988f3dc034dacd5e-831620996.ap-northeast-2.elb.amazonaws.com:8080/products/2 status="SoldOut"

# 주문 상태 확인
http GET http://localhost:8081/orders/1
http GET http://a2c3105e6832445d988f3dc034dacd5e-831620996.ap-northeast-2.elb.amazonaws.com:8080/orders/1
```

## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문(order)->상품(product) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 상품 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
package siren.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

import java.util.Date;

@FeignClient(name="product", url="${feign.client.url.productUrl}")
public interface ProductService {

    @RequestMapping(method = RequestMethod.GET, path="/products/checkProduct")
    public Integer checkProduct(@RequestParam("productId") Long productId);

}
```

- 주문 받은 즉시 상품 가격을 조회하도록 구현
```
@RequestMapping(value = "/products/checkProduct", method = RequestMethod.GET, produces = "application/json;charset=UTF-8")

public Integer checkProduct(@RequestParam("productId") Long productId)
        throws Exception {
        System.out.println("##### /product/checkProduct  called #####");
        Integer price = 0;
        Optional<Product> productOptional = productRepository.findById(productId);
        Product product = productOptional.get();

                if (product.getPrice() > 0) {
                        price = product.getPrice();
                } 
                return price;
        }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 상품 시스템이 장애가 나면 주문도 못받는다는 것을 확인:


```
# 상품 (product) 서비스를 잠시 내려놓음 (ctrl+c, replicas 0 으로 설정)

#주문처리 
http POST http://localhost:8081/orders productId=1   #Fail

#상품서비스 재기동
cd 상품
mvn spring-boot:run

#주문처리
http POST http://localhost:8081/orders productId=1   #Success
```



## 비동기식 호출 publish-subscribe

상품이 등록된 후, 주문 시스템에 상품상태를 전달하는 행위는 동기식이 아닌 비동기식으로 처리한다.
- 이를 위하여 상품이 등록/변경되면 곧바로 상품 상태를 주문 시스템으로 전달하기 위한 이벤트를 카프카로 송출한다(Publish)
 
```
package siren;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="Product_table")
public class Product {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String status;
    private Integer price;

    @PostPersist
    public void onPostPersist(){
        ProductRegistered productRegistered = new ProductRegistered();
        BeanUtils.copyProperties(this, productRegistered);
        productRegistered.publishAfterCommit();
    }

    @PostUpdate
    public void onPostUpdate(){
        UpdatedProductStatus updatedProductStatus = new UpdatedProductStatus();
        BeanUtils.copyProperties(this, updatedProductStatus);
        updatedProductStatus.publishAfterCommit();
    }
```
- 오더 서비스에서는 상품 상태 전달 이벤트를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package siren;

@Service
public class PolicyHandler{
    @Autowired OrderRepository orderRepository;
    @Autowired ProductRepository productRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverProductRegistered_UpdatedProductStatus(@Payload ProductRegistered productRegistered){
        if(!productRegistered.validate()) return;

        System.out.println("\n\n##### listener UpdatedProductStatus : " + productRegistered.toJson() + "\n\n");

        Product product = new Product();
        product.setId(productRegistered.getId());
        product.setStatus(productRegistered.getStatus());
        productRepository.save(product);  
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverUpdatedProductStatus_UpdatedProductStatus(@Payload UpdatedProductStatus updatedProductStatus){
        if(!updatedProductStatus.validate()) return;

        System.out.println("\n\n##### listener UpdatedProductStatus : " + updatedProductStatus.toJson() + "\n\n");

        Product product = new Product();
        product.setId(updatedProductStatus.getId());
        product.setStatus(updatedProductStatus.getStatus());
        productRepository.save(product);      
    }
}
```

주문 시스템은 상품 시스템과 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 오더시스템이 유지보수로 인해 잠시 내려간 상태라도 상품을 등록하는데 문제가 없다:
```
# 오더 서비스 (order) 를 잠시 내려놓음 (ctrl+c)

#상품등록
http POST http://localhost:8082/products price=3500 status=Available   #Success

#상품정보 확인
http GET http://localhost:8082/products/1     # 상품상태 registeredProduct 확인

#오더 서비스 기동
cd order
mvn spring-boot:run

#주문서비스의 상품상태 확인
http GET http://localhost:8081/products/1     # 상품 상태 Available로 변경 확인
```


# 운영

## CICD 설정
SirenOrder의 ECR 구성은 아래와 같다.
![ECR](https://user-images.githubusercontent.com/30651085/120596800-22488200-c47f-11eb-93fc-443b7b8fb2d3.png)

사용한 CI/CD 도구는 AWS CodeBuild
![CodeBuild](https://user-images.githubusercontent.com/30651085/120596645-f88f5b00-c47e-11eb-93f3-de5969e21055.png)
GitHub Webhook이 동작하여 Docker image가 자동 생성 및 ECR 업로드 된다.
(pipeline build script 는 report 폴더 이하에 buildspec.yaml 에 포함)
![CodeBuildVersion](https://user-images.githubusercontent.com/30651085/120596687-02b15980-c47f-11eb-97ef-5c006c1958f5.png)


## Kubernetes 설정
AWS EKS를 활용했으며, 추가한 namespace는 coffee와 kafka로 아래와 같다.

###EKS Deployment

namespace: siren
![EKS](https://user-images.githubusercontent.com/30651085/120596742-10ff7580-c47f-11eb-9f69-20717e0519a2.png)

namespace: kafka
![Kafka](https://user-images.githubusercontent.com/30651085/120596561-df86aa00-c47e-11eb-8958-d9e86111ed50.png)

###EKS Service
gateway가 아래와 같이 LoadBalnacer 역할을 수행한다  

    ➜  ~ kubectl get service -o wide -n siren
    NAME      TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)          AGE   SELECTOR
    gateway   LoadBalancer   10.100.236.235   a2c3105e6832445d988f3dc034dacd5e-831620996.ap-northeast-2.elb.amazonaws.com   8080:30387/TCP   16m   app=gateway
    order     ClusterIP      10.100.236.23    <none>                                                                        8080/TCP         19m   app=order
    product   ClusterIP      10.100.220.191   <none>                                                                        8080/TCP         23m   app=product
    report    ClusterIP      10.100.39.1      <none>                                                                        8080/TCP         17m   app=report


## ConfigMap 설정
특정값을 k8s 설정으로 올리고 서비스를 기동 후, kafka 정상 접근 여부 확인한다.
```
    ➜  ~ kubectl describe cm report-config -n siren
    Name:         report-config
    Namespace:    siren
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    NS:
    ----
    siren
    PORT:
    ----
    8080
    TEXT1:
    ----
    my-kafka.kafka.svc.cluster.local:9092
    TEXT2:
    ----
    Welcome
    Events:  <none>
```
관련된 application.yml 파일 설정은 다음과 같다. 
```
    spring:
      profiles: docker
      cloud:
        bindings:
          event-in:
            group: report
            destination: ${NS} #siren
            contentType: application/json
          event-out:
            destination: ${NS} #siren
            contentType: application/json
```
EKS 설치된 kafka에 정상 접근된 것을 확인할 수 있다. (해당 configMap TEXT1 값을 잘못된 값으로 넣으면 kafka WARN)
```
    2021-05-20 13:42:11.773 INFO 1 --- [pool-1-thread-1] o.a.kafka.common.utils.AppInfoParser : Kafka commitId : fa14705e51bd2ce5
    2021-05-20 13:42:11.785 INFO 1 --- [pool-1-thread-1] org.apache.kafka.clients.Metadata : Cluster ID: kJGw05_iTNOfms7RJu0JSw
    2021-05-20 13:42:14.049 INFO 1 --- [container-0-C-1] o.a.k.c.c.internals.AbstractCoordinator : [Consumer clientId=consumer-3, groupId=report] Attempt to heartbeat failed since group is rebalancing
    2021-05-20 13:42:14.049 INFO 1 --- [container-0-C-1] o.a.k.c.c.internals.ConsumerCoordinator : [Consumer clientId=consumer-3, groupId=report] Revoking previously assigned partitions []
    2021-05-20 13:42:14.049 INFO 1 --- [container-0-C-1] o.s.c.s.b.k.KafkaMessageChannelBinder$1 : partitions revoked: []
    2021-05-20 13:42:14.049 INFO 1 --- [container-0-C-1] o.a.k.c.c.internals.AbstractCoordinator : [Consumer clientId=consumer-3, groupId=report] (Re-)joining group
    2021-05-20 13:42:14.056 INFO 1 --- [container-0-C-1] o.a.k.c.c.internals.AbstractCoordinator : [Consumer clientId=consumer-3, groupId=report] Successfully joined group with generation 3
    2021-05-20 13:42:14.057 INFO 1 --- [container-0-C-1] o.a.k.c.c.internals.ConsumerCoordinator : [Consumer clientId=consumer-3, groupId=report] Setting newly assigned partitions [coffee-0]
    2021-05-20 13:42:14.064 INFO 1 --- [container-0-C-1] o.s.c.s.b.k.KafkaMessageChannelBinder$1 : partitions assigned: [coffee-0]
```

## 셀프힐링 livenessProbe 설정
- order deployment livenessProbe (gateway:5/order:3/product:8/report:5) 
```
          livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 120  //초기delay시간
            timeoutSeconds: 1         //timeout시간내응답점검
            periodSeconds: 5          //점검주기
            failureThreshold: 5       //실패5번이후에는 RESTART
```
livenessProbe 기능 점검은 HPA적용되지 않은 상태에서 진행한다.
```
Pod 의 변화를 살펴보기 위하여 watch
```
➜  ~ kubectl get -n siren po -w
NAME                           READY   STATUS    RESTARTS   AGE
pod/gateway-6449f7459-bcgz6    1/1     Running   0          31m
pod/order-74f45d958f-qnnz5     1/1     Running   0          5m48s
pod/product-698dd8fcc4-5frqp   1/1     Running   0          42m
pod/report-86d9f7b89-knl6h     1/1     Running   0          140m
pod/siege                      1/1     Running   0          119m
```
order 서비스를 다운시키기 위한 부하 발생
```
➜  ~ siege -c100 -t60S -r10 -v --content-type "application/json" 'http://af353bfd8fcc047ee927ad7315ecbd10-155124666.ap-northeast-2.elb.amazonaws.com:8080/orders POST {"productId": "4"}'
```
order Pod의 liveness 조건 미충족에 의한 RESTARTS 횟수 증가 확인
```
➜  ~ kubectl get -n siren po -w
NAME                       READY   STATUS    RESTARTS   AGE
gateway-6449f7459-bcgz6    1/1     Running   0          36m
order-74f45d958f-qnnz5     0/1     Running   1          10m
product-698dd8fcc4-5frqp   1/1     Running   0          46m
report-86d9f7b89-knl6h     1/1     Running   0          144m
siege                      1/1     Running   0          124m
```
kubectl get -n siren po -w
order-74f45d958f-qnnz5     1/1     Running             0          2m6s
order-74f45d958f-qnnz5     0/1     Running             1          9m28s
order-74f45d958f-qnnz5     1/1     Running             1          11m
```



## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함
-  (gateway:5/order:4/product:6/report:5) 

시나리오는 주문(order)-->상품(product) 연결을 RestFul Request/Response 로 연동하여 구현이 되어있고, 주문이 과도할 경우 CB 를 통하여 장애격리.

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
- 상품(product) 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
        @RequestMapping(value = "/products/checkProductStatus", method = RequestMethod.GET, produces = "application/json;charset=UTF-8")
        public Integer checkProductStatus(@RequestParam("productId") Long productId) throws Exception {
                
                //FIXME 생략
                
                //임의의 부하를 위한 강제 설정
                try {
                        Thread.currentThread();
                        Thread.sleep((long) (400 + Math.random() * 220));
                } catch (InterruptedException e) {
                        e.printStackTrace();
                }

                return price;
        }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
siege -c100 -t60S -r10 -v --content-type "application/json" 'http://af353bfd8fcc047ee927ad7315ecbd10-155124666.ap-northeast-2.elb.amazonaws.com:8080/orders POST {"productId": "4"}'
```
```
![C1](https://user-images.githubusercontent.com/30651085/120637724-33f34f00-c4aa-11eb-8b54-def5b0fed1e3.png)
```
```
![C2](https://user-images.githubusercontent.com/30651085/120637803-4cfc0000-c4aa-11eb-8eb8-ec30feebfeea.png)
```
```
![C3](https://user-images.githubusercontent.com/30651085/120637828-54bba480-c4aa-11eb-9a12-d08564db93d2.png)
```

```
NAME                       READY   STATUS              RESTARTS   AGE
order-58bc967c7c-4s9r4     0/1     ContainerCreating   0          0s
order-58bc967c7c-4s9r4     0/1     Running             0          4s
order-58bc967c7c-4s9r4     1/1     Running             0          2m4s

order-58bc967c7c-4s9r4     0/1     Running             1          6m15s

order-58bc967c7c-4s9r4     1/1     Running             1          8m19s
```

- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호. 
  시스템의 안정적인 운영을 위해 HPA 적용 필요.



### Autoscale HPA

- 주문서비스에 대해 HPA를 설정한다. 설정은 CPU 사용량이 5%를 넘어서면 pod를 5개까지 추가한다.(memory 자원 이슈로 10개 불가)
```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: product
  namespace: coffee
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 5

➜  ~ kubectl get hpa -n coffee
NAME      REFERENCE            TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
order     Deployment/order     30%/5%          1         5         5          17h
product   Deployment/product   31%/10%         1         5         5          132m
```
- 부하를 2분간 유지한다.
```
➜  ~ siege -c30 -t60S -r10 --content-type "application/json" 'http://ac4ff02e7969e44afbe64ede4b2441ac-1979746227.ap-northeast-2.elb.amazonaws.com:8080/orders POST {"customerId":2, "productId":1}'
```
- 오토스케일이 어떻게 되고 있는지 확인한다.
```
➜  ~ kubectl get deploy -n coffee
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
customer   1/1     1            1           8h
delivery   1/1     1            1           8h
gateway    2/2     2            2           6h24m
order      1/1     1            1           8h
product    1/1     1            1           8h
report     1/1     1            1           4h51m
```
- 어느정도 시간이 흐르면 스케일 아웃이 동작하는 것을 확인
```
➜  ~ kubectl get deploy -n coffee
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
customer          1/1     1            1           23h
delivery          1/1     1            1           23h
gateway           2/2     2            2           21h
order             5/5     5            5           23h
product           5/5     5            5           23h
report            1/1     1            1           19h
```

- Availability 가 높아진 것을 확인 (siege)
```
Transactions:		         995 hits
Availability:		       82.64 %
Elapsed time:		       59.85 secs
Data transferred:	        0.29 MB
Response time:		        5.11 secs
Transaction rate:	       16.62 trans/sec
Throughput:		        0.00 MB/sec
Concurrency:		       84.94
Successful transactions:         995
Failed transactions:	         209
Longest transaction:	       15.26
Shortest transaction:	        0.02
```


## Zero-downtime deploy
k8s의 무중단 서비스 배포 기능을 점검한다.
```
    ➜  ~ kubectl describe deploy order -n coffee
    Name:                   order
    Namespace:              coffee
    CreationTimestamp:      Thu, 20 May 2021 12:59:14 +0900
    Labels:                 app=order
    Annotations:            deployment.kubernetes.io/revision: 8
    Selector:               app=order
    Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
    StrategyType:           RollingUpdate
    MinReadySeconds:        0
    RollingUpdateStrategy:  50% max unavailable, 50% max surge
    Pod Template:
        Labels:       app=order
        Annotations:  kubectl.kubernetes.io/restartedAt: 2021-05-20T12:06:29Z
        Containers:
            order:
                Image:        740569282574.dkr.ecr.ap-northeast-2.amazonaws.com/order:v1
                Port:         8080/TCP
                Host Port:    0/TCP
                Liveness:     http-get http://:8080/actuator/health delay=120s timeout=2s period=5s #success=1 #failure=5
                Readiness:    http-get http://:8080/actuator/health delay=10s timeout=2s period=5s #success=1 #failure=10
```
기능 점검을 위해 order Deployment의 replicas를 4로 수정했다. 
그리고 위 Readiness와 RollingUpdateStrategy 설정이 정상 적용되는지 확인한다.
```
    ➜  ~ kubectl rollout status deploy/order -n coffee

    ➜  ~ kubectl get po -n coffee
    NAME                        READY   STATUS    RESTARTS   AGE
    customer-785f544f95-mh456   1/1     Running   0          5h40m
    delivery-557f4d7f49-z47bx   1/1     Running   0          5h40m
    gateway-6886bbf85b-58ms8    1/1     Running   0          4h56m
    gateway-6886bbf85b-mg9fz    1/1     Running   0          4h56m
    order-7978b484d8-6qsjq      1/1     Running   0          62s
    order-7978b484d8-h4hjs      1/1     Running   0          62s
    order-7978b484d8-rw2zk      1/1     Running   0          62s
    order-7978b484d8-x622v      1/1     Running   0          62s
    product-7f67966577-n7kqk    1/1     Running   0          5h40m
    report-5c6fd7b477-w9htj     1/1     Running   0          4h27m
    
    ➜  ~ kubectl get deploy -n coffee
    NAME       READY   UP-TO-DATE   AVAILABLE   AGE
    customer   1/1     1            1           8h
    delivery   1/1     1            1           8h
    gateway    2/2     2            2           6h1m
    order      2/4     4            2           8h
    product    1/1     1            1           8h
    report     1/1     1            1           4h28m
    
    ➜  ~ kubectl get po -n coffee
    NAME                        READY   STATUS    RESTARTS   AGE
    customer-785f544f95-mh456   1/1     Running   0          5h41m
    delivery-557f4d7f49-z47bx   1/1     Running   0          5h41m
    gateway-6886bbf85b-58ms8    1/1     Running   0          4h57m
    gateway-6886bbf85b-mg9fz    1/1     Running   0          4h57m
    order-7978b484d8-6qsjq      1/1     Running   0          115s
    order-7978b484d8-rw2zk      1/1     Running   0          115s
    order-84c9d7c848-mmw4b      0/1     Running   0          18s
    order-84c9d7c848-r64lc      0/1     Running   0          18s
    order-84c9d7c848-tbl8l      0/1     Running   0          18s
    order-84c9d7c848-tslfc      0/1     Running   0          18s
    product-7f67966577-n7kqk    1/1     Running   0          5h41m
    report-5c6fd7b477-w9htj     1/1     Running   0          4h28m
    
    ➜  ~ kubectl get po -n coffee
    NAME                        READY   STATUS    RESTARTS   AGE
    customer-785f544f95-mh456   1/1     Running   0          5h42m
    delivery-557f4d7f49-z47bx   1/1     Running   0          5h42m
    gateway-6886bbf85b-58ms8    1/1     Running   0          4h58m
    gateway-6886bbf85b-mg9fz    1/1     Running   0          4h58m
    order-84c9d7c848-mmw4b      1/1     Running   0          65s
    order-84c9d7c848-r64lc      1/1     Running   0          65s
    order-84c9d7c848-tbl8l      1/1     Running   0          65s
    order-84c9d7c848-tslfc      1/1     Running   0          65s
    product-7f67966577-n7kqk    1/1     Running   0          5h42m
    report-5c6fd7b477-w9htj     1/1     Running   0          4h29m
```
배포시 pod는 위의 흐름과 같이 생성 및 종료되어 서비스의 무중단을 보장했다.



