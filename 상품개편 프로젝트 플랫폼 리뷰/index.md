# 상품개편 플랫폼 개발 리뷰

### # 컨테이너 분기 처리
- UI/UX: PC / MOBILE(jsx분기) / PC멀티차량 광고등록
- 초기데이터: 기본 / 화물특장(1.5톤 이상,이하) / 안드로이드 인앱


### # 초기 데이터 설명
1. readside 차량정보 api가져옴(차량의 대기,판매상태 / 화물인지 아닌지)
- > dispatchGetCarDetails

- > url: /v1/readside/vehicles?vehicleIds=28707689,28707583,28707460&include=SPEC,ADVERTISEMENT,PHOTOS,CATEGORY,MANAGE,CONTACT,VIEW

2. 일반/화물특장 분기처리로 패키지 리스트 가져옴
- > 화물: handleTruckPackageList, 일반: dispatchGetPackageList
- > url: /v1/product/packages/amts?siteType=A0101&userType=U0103&carType=C01C1

3. 그외 나머지 공통적으로 실행


### # api 간단 설명
#### 패키지리스트
> /v1/product/packages/amts
#### 상품리스트(개별상품)
> /v1/product/amts/cache
#### 이용가능 상품정보(available)
> /legacy/service/cars/advertisements
#### 제휴여부(effectivePartnershipPaymentFlag)
> /v1/partnership/payment/effective-payments

#### 유저 이용권(registrations)
> /user/v4/using-service/fjsekf2/payments


### 데이터 상태관리
- dispatch를 모아둔 hook => useAdPackageData
- UI를 만들기 위해 store데이터를 정규화하는 hook => useAdPackageState


### 특이사항
1. 화물 특장 타입을 알아내려면 api를 차량마다 요청해야한다
2. 오마이콜은 차량기준이 아니라 유저 기준(오마이콜 이용기간 60일이상이면 선택불가)
3. 모바일 프리미엄은 차량 사진 1개이상 일때 사용 가능하도록 수정됨(이전엔 4개)