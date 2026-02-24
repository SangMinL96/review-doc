# 상품개편 플랫폼 개발 리뷰

---

## 1. 컨테이너 분기 처리

| 구분            | 분기 유형                                          |
| --------------- | -------------------------------------------------- |
| **UI/UX**       | PC / MOBILE(JSX 분기) / PC 멀티차량 광고등록       |
| **초기 데이터** | 기본 / 화물특장(1.5톤 이상·이하) / 안드로이드 인앱 |

---

## 2. 초기 데이터 흐름

### ① Readside 차량정보 API

- **역할**: 차량의 대기·판매상태, 화물 여부 조회
- **Dispatch**: `dispatchGetCarDetails`
- **URL**
  ```
  /v1/readside/vehicles?vehicleIds=...&include=SPEC,ADVERTISEMENT,PHOTOS,CATEGORY,MANAGE,CONTACT,VIEW
  ```

### ② 패키지 리스트 (일반 / 화물특장 분기)

- **화물**: `handleTruckPackageList`
- **일반**: `dispatchGetPackageList`
- **URL**
  ```
  /v1/product/packages/amts?siteType=A0101&userType=U0103&carType=C01C1
  ```

### ③ 그 외

- 나머지 로직은 **공통** 실행

---

## 3. API 요약

| 용도                                        | 엔드포인트                                       |
| ------------------------------------------- | ------------------------------------------------ |
| 패키지 리스트                               | `GET /v1/product/packages/amts`                  |
| 상품 리스트 (개별상품)                      | `GET /v1/product/amts/cache`                     |
| 이용가능 상품정보 (available)               | `GET /legacy/service/cars/advertisements`        |
| 제휴 여부 (effectivePartnershipPaymentFlag) | `GET /v1/partnership/payment/effective-payments` |
| 유저 이용권 (registrations)                 | `GET /user/v4/using-service/{userId}/payments`   |

---

## 4. 데이터 상태 관리

| Hook                  | 역할                                        |
| --------------------- | ------------------------------------------- |
| **useAdPackageData**  | dispatch를 모아둔 hook                      |
| **useAdPackageState** | store 데이터를 정규화해 UI에 쓰기 위한 hook |

---

## 5. 특이사항

1. **화물 특장**: 타입 확인을 위해 **차량마다** API 요청 필요
2. **오마이콜**: 차량 기준 ❌ → **유저 기준** (이용기간 60일 이상이면 선택 불가)
3. **모바일 프리미엄**: 차량 사진 **1개 이상**일 때 사용 가능 (변경 전: 4개)
