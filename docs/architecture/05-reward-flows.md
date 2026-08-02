# 리워드 플로우 (Reward Flows)

> 상태: ✅ 확정 · 최종수정: 2026-08-02

멤버십 등급 산정, 만보기 미션, 가입 웰컴 혜택 세 갈래를 다룬다.
포인트는 언제나 `point_ledger` 원장이 진실이고 `users.points_balance`는 같은 트랜잭션에서 갱신되는 캐시다.

> 이미지 버전: [`assets/diagrams/05-reward-flows-1.svg`](assets/diagrams/05-reward-flows-1.svg) · [`-2.svg`](assets/diagrams/05-reward-flows-2.svg) · [`-3.svg`](assets/diagrams/05-reward-flows-3.svg)

---

## 1. 멤버십 등급 산정과 적립

```mermaid
%%{init: {"theme":"base","fontFamily":"Pretendard, Malgun Gothic, sans-serif","themeVariables":{"fontSize":"14px","primaryColor":"#DBEAFE","primaryBorderColor":"#1D4ED8","primaryTextColor":"#0F172A","lineColor":"#1D4ED8","secondaryColor":"#FEF3C7","tertiaryColor":"#DCFCE7","clusterBkg":"#F8FAFC","clusterBorder":"#CBD5E1","noteBkgColor":"#FEF3C7","noteBorderColor":"#D97706","actorBkg":"#DBEAFE","actorBorder":"#1D4ED8","actorTextColor":"#0F172A","signalColor":"#1D4ED8","signalTextColor":"#0F172A","labelBoxBkgColor":"#DBEAFE","labelBoxBorderColor":"#1D4ED8","altSectionBkgColor":"#F8FAFC"},"flowchart":{"curve":"basis","htmlLabels":true,"padding":12}}}%%
%% 공통 브랜드 테마 — architecture/*.md 전 다이어그램에 동일한 init 블록이 들어간다. 색을 바꿀 때는 이 폴더 전 파일을 일괄 치환할 것.
flowchart TD
    trig(["주문이 delivered 로 전환"]) --> flip["먼저 상태를 delivered 로 바꾼다<br/>→ 이 주문도 누적액에 포함된다"]
    flip --> sum["누적 결제액 집계<br/>SUM(orders.total)<br/>WHERE status = delivered<br/>기간 제한 없음 · 저장 컬럼 없음"]

    sum --> tier{"누적액 임계"}
    tier -->|"15,000,000원 이상"| gold["GOLD · 2.0%"]
    tier -->|"5,000,000원 이상"| silver["SILVER · 1.5%"]
    tier -->|"그 미만"| basic["BASIC · 1.0%"]

    gold --> base
    silver --> base
    basic --> base["적립 기준액<br/>= subtotal − 쿠폰할인 − 사용포인트<br/>(배송비 제외, 음수면 0)"]
    base --> calc["적립액 = 기준액 × 등급률<br/>basis point 정수연산, 소수점 버림"]
    calc --> zero{"적립액 > 0?"}
    zero -->|"아니오"| skip["적립 생략"]
    zero -->|"예"| ledger["point_ledger ORDER_EARN (+)<br/>ref_id = 주문 id<br/>users.points_balance 가산"]

    classDef step fill:#DBEAFE,stroke:#1D4ED8,stroke-width:1.5px,color:#0F172A
    classDef gate fill:#FEF3C7,stroke:#D97706,stroke-width:1.5px,color:#451A03
    classDef ok fill:#DCFCE7,stroke:#16A34A,stroke-width:1.5px,color:#052E16
    classDef mute fill:#F1F5F9,stroke:#94A3B8,stroke-width:1.5px,color:#0F172A
    class trig,flip,sum,base,calc step
    class tier,zero gate
    class gold,silver,basic,ledger ok
    class skip mute
```

| 등급 | 누적 기준(배송완료 `total` 합) | 적립률 |
|---|---|---|
| `BASIC` | 0원 ~ | 1.0% |
| `SILVER` | 5,000,000원 ~ | 1.5% |
| `GOLD` | 15,000,000원 ~ | 2.0% |

등급은 **저장하지 않고 요청 때마다 집계**한다 — `status=delivered` 주문의 `total` 합계를 전 기간에 걸쳐 더한
값 하나로 결정된다. 적립 기준액은 `total`이 아니라 `subtotal − 쿠폰할인 − 사용포인트`라 **배송비는 적립에서
빠진다**. 계산은 부동소수점 오차를 피하려고 basis point(1/10000) 정수 연산으로 하고 소수점 이하는 버린다.
`delivered` 전환 시 상태를 먼저 바꾸고 등급을 조회하므로 **그 주문 자신도 누적액에 포함**되며, `delivered`
이후에는 취소가 불가능하니 적립을 회수하는 역분개 로직은 필요하지 않다. 등급은 적립률 외에 로그인·내 정보
응답의 `tier`·`tierProgress`로도 노출된다.

---

## 2. 만보기 미션

```mermaid
%%{init: {"theme":"base","fontFamily":"Pretendard, Malgun Gothic, sans-serif","themeVariables":{"fontSize":"14px","primaryColor":"#DBEAFE","primaryBorderColor":"#1D4ED8","primaryTextColor":"#0F172A","lineColor":"#1D4ED8","secondaryColor":"#FEF3C7","tertiaryColor":"#DCFCE7","clusterBkg":"#F8FAFC","clusterBorder":"#CBD5E1","noteBkgColor":"#FEF3C7","noteBorderColor":"#D97706","actorBkg":"#DBEAFE","actorBorder":"#1D4ED8","actorTextColor":"#0F172A","signalColor":"#1D4ED8","signalTextColor":"#0F172A","labelBoxBkgColor":"#DBEAFE","labelBoxBorderColor":"#1D4ED8","altSectionBkgColor":"#F8FAFC"},"flowchart":{"curve":"basis","htmlLabels":true,"padding":12}}}%%
%% 공통 브랜드 테마 — architecture/*.md 전 다이어그램에 동일한 init 블록이 들어간다. 색을 바꿀 때는 이 폴더 전 파일을 일괄 치환할 것.
sequenceDiagram
    autonumber
    participant SEN as 걸음 센서<br/>TYPE_STEP_COUNTER
    participant APP as 안드로이드 앱<br/>StepCounter
    participant BR as JS 브릿지<br/>window.AndroidBridge
    participant WEB as 웹 /steps<br/>useSteps
    participant API as API 서버<br/>StepService
    participant DB as MariaDB

    SEN->>APP: 부팅 후 누적 걸음수
    APP->>APP: baseline 보정<br/>날짜 바뀌면 baseline 재설정<br/>재부팅 감지 시 현재값을 새 baseline
    WEB->>BR: getStepInfo()
    BR-->>WEB: {available, permission, steps, date}

    alt permission ≠ granted
        WEB->>BR: requestStepPermission()
        Note over WEB: ACTIVITY_RECOGNITION 권한 요청<br/>(안드로이드 10 / SDK 29 이상)
    else 브릿지 없음(일반 브라우저)
        Note over WEB: 수동 입력 데모 패널 노출
    end

    WEB->>API: POST /api/v1/steps<br/>{date:"YYYY-MM-DD", steps}
    Note over API: 역할 CUSTOMER 전용<br/>steps 0~100000<br/>date는 오늘 기준 ±1일

    API->>DB: 사용자 행 잠금 (FOR UPDATE)
    API->>DB: user_daily_steps 행 잠금<br/>UK(user_id, step_date)
    API->>API: 제출값 ≤ 기존값이면 무시(ignored)<br/>아니면 그 날의 최댓값으로 갱신

    alt 목표 상승 엣지 + goal_rewarded_at IS NULL
        API->>DB: goal_rewarded_at = now (래치)
        API->>DB: point_ledger STEP_GOAL (+100)<br/>ref_id = "YYYY-MM-DD"
        API->>DB: users.points_balance 가산
        API->>DB: notifications STEP_MISSION<br/>"걸음 목표 달성"
        API-->>WEB: rewardGranted = true
        WEB->>WEB: 토스트 + 포인트·알림 캐시 무효화
    else 이미 지급됐거나 목표 미달
        API-->>WEB: rewardGranted = false
    end
```

앱의 역할은 **수집·제공까지**다 — 서버 제출은 로그인 토큰을 쥔 웹이 한다. 앱은 `window.AndroidBridge`라는
JavaScript 인터페이스로 `getStepInfo()`·`requestStepPermission()`을 노출하고, 웹은 브릿지 존재 여부로
앱 안인지 일반 브라우저인지 판단해 브라우저면 수동 입력 데모 패널을 띄운다. 서버는 같은 날짜에 대해
**그날의 최댓값만 유지**하고(제출값이 기존값 이하면 무시), 보상은 "직전엔 목표 미달 → 이번에 목표 달성"이라는
상승 엣지에 `goal_rewarded_at IS NULL` 래치를 더한 네 조건이 모두 맞을 때만 한 번 나간다. 목표 걸음수와 보상
포인트는 행을 만들 때 스냅샷으로 굳어져서, 이후 설정을 바꿔도 이미 만들어진 날짜의 판정 기준은 흔들리지 않는다.

| 설정 키 | 기본값 | 비고 |
|---|---|---|
| `app.steps.zone-id` | `Asia/Seoul` | 오늘 날짜 판정 기준 시간대 |
| `app.steps.daily-goal` | `10000` | 1~100,000 |
| `app.steps.reward-points` | `100` | 1 이상 |

> ⚠️ **구현 주의 2건**: ① 원장 사유는 `STEP_REWARD`가 아니라 **`STEP_GOAL`** 이다(두 값 모두 enum에 존재하나
> `STEP_REWARD`는 미사용). ② `UserCoupon.Source.STEP_MISSION` enum 값은 정의만 되어 있고 **만보기로 쿠폰을
> 발급하는 코드는 없다** — 걸음 보상은 포인트와 알림뿐이다(`Notification.Type.STEP_MISSION`과 혼동 주의).

---

## 3. 가입 웰컴 혜택

```mermaid
%%{init: {"theme":"base","fontFamily":"Pretendard, Malgun Gothic, sans-serif","themeVariables":{"fontSize":"14px","primaryColor":"#DBEAFE","primaryBorderColor":"#1D4ED8","primaryTextColor":"#0F172A","lineColor":"#1D4ED8","secondaryColor":"#FEF3C7","tertiaryColor":"#DCFCE7","clusterBkg":"#F8FAFC","clusterBorder":"#CBD5E1","noteBkgColor":"#FEF3C7","noteBorderColor":"#D97706","actorBkg":"#DBEAFE","actorBorder":"#1D4ED8","actorTextColor":"#0F172A","signalColor":"#1D4ED8","signalTextColor":"#0F172A","labelBoxBkgColor":"#DBEAFE","labelBoxBorderColor":"#1D4ED8","altSectionBkgColor":"#F8FAFC"},"flowchart":{"curve":"basis","htmlLabels":true,"padding":12}}}%%
%% 공통 브랜드 테마 — architecture/*.md 전 다이어그램에 동일한 init 블록이 들어간다. 색을 바꿀 때는 이 폴더 전 파일을 일괄 치환할 것.
flowchart TD
    start(["POST /api/v1/auth/signup"]) --> dup{"이메일 중복?"}
    dup -->|"예"| e409["409 EMAIL_DUPLICATED"]
    dup -->|"아니오"| create["users 생성<br/>role = CUSTOMER 고정<br/>password_hash = BCrypt<br/>points_balance = 0"]

    create --> bonus["① point_ledger SIGNUP_BONUS<br/>+5,000P · ref_id = NULL<br/>잔액 캐시 가산"]
    bonus --> couponLock["② WELCOME10 쿠폰 정의 행 잠금"]
    couponLock --> usable{"쿠폰이<br/>사용 가능 상태인가?"}
    usable -->|"아니오"| einv["400 INVALID_COUPON"]
    usable -->|"예"| already{"이미 이 쿠폰을<br/>보유 중인가?"}
    already -->|"예"| skip["발급 생략"]
    already -->|"아니오"| issue["user_coupons 발급<br/>source = SIGNUP<br/>source_ref = WELCOME10<br/>10,000원 정액 · 최소주문 없음"]
    issue --> noti["③ notifications PROMOTION<br/>'회원가입 웰컴 쿠폰 지급'"]
    skip --> token
    noti --> token["JWT 발급 + 사용자 요약 응답<br/>(tier · tierProgress 포함)"]

    classDef step fill:#DBEAFE,stroke:#1D4ED8,stroke-width:1.5px,color:#0F172A
    classDef gate fill:#FEF3C7,stroke:#D97706,stroke-width:1.5px,color:#451A03
    classDef fail fill:#FFE4E6,stroke:#E11D48,stroke-width:1.5px,color:#4C0519
    classDef ok fill:#DCFCE7,stroke:#16A34A,stroke-width:1.5px,color:#052E16
    classDef mute fill:#F1F5F9,stroke:#94A3B8,stroke-width:1.5px,color:#0F172A
    class start,create,couponLock,bonus step
    class dup,usable,already gate
    class e409,einv fail
    class issue,noti,token ok
    class skip mute
```

가입은 하나의 트랜잭션에서 **계정 생성 → 포인트 5,000P 적립 → `WELCOME10` 쿠폰 발급 → 알림 → 토큰 발급**
순서로 진행된다. 일반 가입으로 만들어지는 계정의 역할은 언제나 `CUSTOMER`로 고정되고, staff 계정은 시드나
별도 발급 경로로만 생긴다. 쿠폰 발급은 정의 행을 잠근 뒤 중복 보유를 확인해서 이미 갖고 있으면 조용히
건너뛴다 — 이 경우 알림도 나가지 않는다. 쿠폰 지갑에 들어오는 경로는 `SIGNUP`(가입)·`CODE_REDEEM`(코드 입력)·
`ADMIN_ISSUE`(시드) 세 가지가 실제로 쓰이고, 주문에 적용 가능한 쿠폰은 **오직 내 지갑에 있는 미사용 쿠폰**뿐이다.

### 포인트 원장 사유

| `reason` | 부호 | 발생 지점 | `ref_id` |
|---|---|---|---|
| `SIGNUP_BONUS` | + | 회원가입 | `NULL` |
| `STEP_GOAL` | + | 만보기 목표 달성 | `YYYY-MM-DD` |
| `ORDER_USE` | − | 주문 생성 시 포인트 사용 | 주문 id |
| `ORDER_EARN` | + | `delivered` 전환 적립 | 주문 id |
| `ORDER_REFUND` | + | 주문 취소 시 사용 포인트 환불 | 주문 id |
| `ADMIN_ADJUST` | ± | 운영 조정(정정용) | — |
| `STEP_REWARD` | — | **미사용**(enum에만 존재) | — |

원장은 append-only라 수정·삭제가 없고, 정정이 필요하면 상쇄 엔트리를 새로 넣는다.
원장 삽입과 `users.points_balance` 갱신은 항상 같은 트랜잭션에서 함께 일어난다.
