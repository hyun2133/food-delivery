![image](https://user-images.githubusercontent.com/487999/79708354-29074a80-82fa-11ea-80df-0db3962fb453.png)

# 예제 - 음식배달

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 - 음식배달](#---)
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
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

배달의 민족 커버하기 - https://1sung.tistory.com/106

기능적 요구사항
1. 고객이 메뉴를 선택하여 주문한다.
2. 고객이 선택한 메뉴에 대해 결제한다.
3. 주문이 되면 주문 내역이 상점주인에게 주문정보가 전달된다
4. 상점주는 주문을 수락하거나 거절할 수 있다
5. 상점주는 요리시작때와 완료 시점에 시스템에 상태를 입력한다
6. 고객은 아직 요리가 시작되지 않은 주문은 취소할 수 있다
7. 요리가 완료되면 고객의 지역 인근의 라이더들에 의해 배송건 조회가 가능하다
8. 라이더가 해당 요리를 Pick한 후, 앱을 통해 통보한다.
9. 고객이 주문상태를 중간중간 조회한다
10. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다
11. 고객이 요리를 배달 받으면 배송확인 버튼을 탭하여, 모든 거래가 완료된다.
12. 고객이 요리에 대한 만족도에 대한 리뷰을 작성한다.
13. 상점주는 리뷰의 댓글을 올린다.


비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다  Sync 호출 
1. 장애격리
    1. 상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 자주 상점관리에서 확인할 수 있는 배달상태를 주문시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS
    1. 배달상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다  Event driven

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![스크린샷_20221206_095841](https://user-images.githubusercontent.com/119825871/205927669-90a95d03-b3b0-4f7c-942b-0399bfd6f67a.png)

    - 고객이 메뉴를 선택하여 주문한다 (ok)
    - 고객이 결제한다 (ok)
    - 주문이 되면 주문 내역이 상점주인에게 전달된다 (ok)
    - 상점주인이 확인하여 요리해서 배달 출발한다 (ok)  

![스크린샷_20221206_095842](https://user-images.githubusercontent.com/119825871/205928305-c8292f3a-2678-4ae0-b33e-c66209553269.png)

    - 고객이 주문을 취소할 수 있다 (ok)
    - 고객이 주문상태를 중간중간 조회한다 (View-green sticker 의 추가로 ok) 
    - 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다.
    
![image](https://user-images.githubusercontent.com/119825871/205928635-eb3a62da-3486-4047-a3bb-6fb9a61971ef.png)  

    - 고객이 리뷰을 작성한다.
    - 상점주인이 댓글을 올린다.



## 체크포인트
## 1. Saga (Pub / Sub)
### 구현 : Order
![스크린샷_20221206_022606](https://user-images.githubusercontent.com/119825871/205823983-57340537-5eeb-4725-b250-72db913467c4.png)

### Order 실행
![스크린샷_20221206_032107](https://user-images.githubusercontent.com/119825871/205836807-522b42de-aa36-4178-9448-62e6e1e40a0f.png)

### 구현: Store
![image](https://user-images.githubusercontent.com/119825871/205917131-0da35007-2a17-45e8-a45e-e3b6e5d1abd8.png)

### Store실행 : 상점에서 주문을 확인한다.
![스크린샷_20221206_031330](https://user-images.githubusercontent.com/119825871/205835621-ae00566f-4c6c-4202-8acb-1c5e87e79663.png)

### kafka 확인
![스크린샷_20221206_032316](https://user-images.githubusercontent.com/119825871/205837076-c39e8119-9678-4d31-b57f-eba88450b9ab.png)

## 2. CQRS
### 속성
![image](https://user-images.githubusercontent.com/119825871/205920177-6949f9f9-965a-4608-ba3f-f59def9150c1.png)
### CRUD 상세설계
![image](https://user-images.githubusercontent.com/119825871/205921077-f4e1ba98-a828-47b7-a924-4f59f254219d.png)
### 구현
```
@Service
public class MyPageViewHandler {

    @Autowired
    private MyPageRepository myPageRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrderPlaced_then_CREATE_1 (@Payload OrderPlaced orderPlaced) {
        try {

            if (!orderPlaced.validate()) return;

            // view 객체 생성
            MyPage myPage = new MyPage();
            // view 객체에 이벤트의 Value 를 set 함
            myPage.setStatus("주문됨");
            // view 레파지 토리에 save
            myPageRepository.save(myPage);

        }catch (Exception e){
            e.printStackTrace();
        }
    }


    @StreamListener(KafkaProcessor.INPUT)
    public void whenPaid_then_UPDATE_1(@Payload Paid paid) {
        try {
            if (!paid.validate()) return;
                // view 객체 조회
            Optional<MyPage> myPageOptional = myPageRepository.findById(Long.valueOf(paid.getOrderId()));

            if( myPageOptional.isPresent()) {
                 MyPage myPage = myPageOptional.get();
            // view 객체에 이벤트의 eventDirectValue 를 set 함
                myPage.setStatus("결재됨");    
                // view 레파지 토리에 save
                 myPageRepository.save(myPage);
                }


        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrderAccepted_then_UPDATE_2(@Payload OrderAccepted orderAccepted) {
        try {
            if (!orderAccepted.validate()) return;
                // view 객체 조회

                List<MyPage> myPageList = myPageRepository.findByStatus(orderAccepted.getOrderId());
                for(MyPage myPage : myPageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    myPage.setStatus("접수됨");
                // view 레파지 토리에 save
                myPageRepository.save(myPage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }

}
```

## 3. Compensation / Correlation
### 주문
```
    @PostPersist
    public void onPostPersist(){


        OrderPlaced orderPlaced = new OrderPlaced(this);
        orderPlaced.publishAfterCommit();
        
    }
 ```
  ### 주문취소  
  ```
    @PreRemove
    public void onPreRemove(){
        OrderCancel orderCancel = new OrderCancel(this);
        orderCancel.publishAfterCommit();

    }
  ```
 ###  MyPageViewHandler.java
 ```
 @StreamListener(KafkaProcessor.INPUT)
    public void whenOrderAccepted_then_DELETE_1(@Payload OrderAccepted orderAccepted) {
        try {
            if (!orderAccepted.validate()) return;
            // view 레파지 토리에 삭제 쿼리
            myPageRepository.deleteById(Long.valueOf(orderAccepted.getOrderId()));
        }catch (Exception e){
            e.printStackTrace();
        }
    }
  ```

## 4. Request / Response
### 구현
![스크린샷_20221206_041555](https://user-images.githubusercontent.com/119825871/205846190-73cf48a6-9ca7-4bb1-b81c-978ef0bc9220.png)
### 실행
![스크린샷_20221206_041714](https://user-images.githubusercontent.com/119825871/205846241-d3854f8a-28b3-4f16-b940-bde07f11e93e.png)

## 5. Circuit Breaker

## 6. Gateway / Ingress
- 구현
![스크린샷_20221206_020802](https://user-images.githubusercontent.com/119825871/205822195-92d8cff9-d2d7-4ace-8ee7-04b65f047ae2.png)

- 실행
![스크린샷_20221206_032107](https://user-images.githubusercontent.com/119825871/205836832-4f81c8a5-5d6e-4de7-9948-0475ebcdfe2f.png)







