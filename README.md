# SSW E-Commerce Demo — 문서·아키텍처 공개본

> 이 레포는 SSW 이커머스 데모의 **문서·아키텍처 공개본**입니다.
> 전체 구현(서버 · 사용자 웹 · 관리자 웹 · Windows 앱 · Android 앱 5개 레포)은
> 비공개로 관리되며, 여기에는 기획·설계 문서와 다이어그램만 담았습니다.

멀티플랫폼 이커머스 데모의 설계 기록입니다. 표면은 종합몰이지만 실제 목적은
**멀티플랫폼 구현 역량과 AI 오케스트레이션 개발 방식을 보여주는 포트폴리오**입니다.

> ⚠️ 시연용 데모입니다. 등장하는 **상품·가격·주문·리뷰·계정은 전부 예시 시드 데이터**이고,
> 결제(SSW PAY)·소셜 로그인·배송사 연동 등 **외부 연동은 전부 목업**이라 실제 외부
> 시스템으로 나가는 호출은 하나도 없습니다.

## 시스템 구성

```mermaid
%%{init: {"theme":"base","fontFamily":"Pretendard, Malgun Gothic, sans-serif","themeVariables":{"fontSize":"14px","primaryColor":"#DBEAFE","primaryBorderColor":"#1D4ED8","primaryTextColor":"#0F172A","lineColor":"#1D4ED8","secondaryColor":"#FEF3C7","tertiaryColor":"#DCFCE7","clusterBkg":"#F8FAFC","clusterBorder":"#CBD5E1","noteBkgColor":"#FEF3C7","noteBorderColor":"#D97706","actorBkg":"#DBEAFE","actorBorder":"#1D4ED8","actorTextColor":"#0F172A","signalColor":"#1D4ED8","signalTextColor":"#0F172A","labelBoxBkgColor":"#DBEAFE","labelBoxBorderColor":"#1D4ED8","altSectionBkgColor":"#F8FAFC"},"flowchart":{"curve":"basis","htmlLabels":true,"padding":12}}}%%
%% 공통 브랜드 테마 — architecture/*.md 전 다이어그램에 동일한 init 블록이 들어간다. 색을 바꿀 때는 이 폴더 전 파일을 일괄 치환할 것.
graph TB
    subgraph clients["클라이언트"]
        android["안드로이드 앱<br/>Kotlin · WebView 셸"]
        web["사용자 웹<br/>Next.js<br/>운영: Cloudflare Workers"]
        adminweb["관리자 웹<br/>Next.js<br/>운영: Cloudflare Workers"]
        wpf["관리자 Windows<br/>WPF (.NET 8)"]
    end

    server["API 서버<br/>Spring Boot 3.4<br/>운영: 클라우드 VM · systemd"]
    db[("MariaDB 11")]
    llm["자체 LLM 서빙 서버<br/>vLLM · OpenAI 호환<br/>※ 평소 오프라인 → 자동 강등"]

    subgraph infra["공용 인프라 — 데모가 위임한 3축"]
        gw["게이트웨이<br/>WAF IP 허용목록"]
        auth["auth-server<br/>RS256 발급 · JWKS"]
        file["file-server · media<br/>이미지 원본 · 파생본"]
        chat["chat-server<br/>실시간 상담"]
        gw --> auth
        gw --> file
        gw --> chat
    end

    android -->|"WebView 로드"| web
    web -->|"REST /api/v1"| server
    adminweb -->|"REST /api/v1 (admin)"| server
    wpf -->|"REST /api/v1 (admin)"| server
    server --> db

    web -.->|"챗봇 분류·RAG 답변"| llm
    adminweb -.->|"AI 브리핑·문의 초안"| llm

    server -.->|"인증 위임 · 이미지 업로드<br/>상담 룸 생성"| gw
    web -.->|"이미지 GET · 무인증 공개 경로"| file
    web -.->|"상담 WebSocket · 브라우저 직접 접속"| chat

    subgraph mocks["데모 목업 (외부 연동 없음)"]
        pay["SSW PAY 결제"]
        login["데모 계정 로그인"]
    end
    web --- mocks

    classDef step fill:#DBEAFE,stroke:#1D4ED8,stroke-width:1.5px,color:#0F172A
    classDef ext fill:#F1F5F9,stroke:#94A3B8,stroke-width:1.5px,color:#0F172A
    class android,web,adminweb,wpf,server,db step
    class gw,auth,file,chat,llm,pay,login ext
```

실선은 데모 동작에 반드시 필요한 경로, 점선은 **없어도 기능이 강등된 채로 동작**하는
선택 경로입니다. 상세는 [시스템 아키텍처 문서](docs/architecture/01-system-architecture.md).

## 스택

| 딜리버러블 | 역할 | 스택 | 포트 |
|---|---|---|---|
| 서버 | 백엔드 API(`/api/v1`) — 인증·주문·상품·포인트·쿠폰·알림 | Java 17 · Spring Boot 3.4 · JPA · MariaDB 11 |
| 사용자 웹 | 고객 화면 — 카탈로그·주문·마이페이지·챗봇 | Next.js 16 · React 19 · TypeScript · Tailwind v4 |
| 관리자 웹 | 대시보드·주문/리뷰/문의/상품 운영·AI 실험실 | Next.js 16 · React 19 · TypeScript · Tailwind v4 |
| 관리자 Windows | 직원용 데스크톱 관리 앱 | WPF(`net8.0-windows`) · .NET 8 · MVVM(CommunityToolkit) |
| 안드로이드 | 고객 앱 — WebView 앱 셸 + 네이티브 하단 네비게이션 | Kotlin · Gradle |

클라이언트 4종이 **하나의 백엔드**(`/api/v1`)를 공유하고, 인증은 JWT + 역할 게이트
(`CUSTOMER` · `ADMIN` · `SELLER_SUPPORT` · `CUSTOMER_SUPPORT`)로 통일했습니다.

## 기능 하이라이트

**고객 웹** — 카탈로그(그리드/리스트 전환 애니메이션), 상품 상세(리뷰·포토 리뷰 라이트박스·관련
상품), 장바구니, 주문/결제(SSW PAY 목업), 주문 상세 타임라인, 배송지 관리(자동 채움), 알림함,
1:1 문의, 찜, 포인트·쿠폰, 매장 안내, 플로팅 메뉴(레이아웃 × 등장 효과 설정), 다크/라이트 +
accent 6종 테마.

**관리자 웹** — 대시보드(매출·추이), 주문 칸반, 리뷰 블라인드, 문의 답변, 상품 CRUD,
커맨드 팔레트(Ctrl+K), CSV 내보내기, 재고 위젯, 감사 로그, 주문 상태 변경 히트맵, 라이브 시연 모드,
챗봇 세션 관리(회수 조정 · 최근 접속 IP · 세션 삭제).

**관리자 Windows (WPF)** — 로그인 창 + 메인 창 7개 탭(대시보드 · 주문 · 상품 · 리뷰 · 매장 ·
문의 · 감사로그). 관리자 웹과 같은 백엔드를 바라보며 **역할별 쓰기 권한 차이**를 그대로 반영합니다.

**안드로이드** — 사용자 웹을 감싸는 WebView 앱 셸에 네이티브 하단 탭 5종을 얹고, 웹 SPA 라우팅과
탭 상태를 브릿지로 동기화합니다. 라우트·URL 처리 정책과 만보기 계산 규칙은 순수 함수로 분리해
JVM 단위 테스트로 검증합니다.

**AI 기능** — 챗봇(키워드 프리필터 → 분류 게이트 → 미니 RAG → 답변 생성 4단계), AI 브리핑,
문의 답변 초안. LLM 서버는 **평소 꺼진 상태가 기본**이고, 응답이 없으면 세 기능 모두 규칙 기반
**오프라인 폴백으로 자동 강등**되어 시연이 끊기지 않습니다. 자연어 통계 실험실은 관리자 웹을
Cloudflare Workers로 옮기면서 **비활성**됐습니다 — MariaDB TCP 직결이 워커 런타임에서 성립하지
않기 때문입니다. 복구는 옛 NL→SQL 구조를 되살리는 대신 **고정 집계 + 해석**으로 다시 짭니다.
서버가 미리 정의한 지표만 실행하고 LLM은 지표 선택과 결과 해석만 맡으므로 임의 SQL이 실행되는
표면 자체가 없습니다. 서버 쪽 지표 API는 이미 신설됐고 화면 작업이 남았습니다. 챗봇에는 세션당 대화 회수 제한이 붙어 있는데, 잔여 회수를 클라이언트가 아니라 **서버가
정본으로 들고 있어** 관리자가 값을 조정하면 고객이 새로고침하지 않아도 다음 메시지부터 반영됩니다.

**상담원 연결** — 챗봇에서 상담사 **1:1 실시간 채팅**으로 넘어갈 수 있습니다. 연결 앞에 필수 문진
(카테고리 → 관련 주문 → 상세 설명)을 두어 유입을 조절하면서 상담원이 첫 줄부터 맥락을 갖고 시작하게
하고, 대기·진행 상태는 위젯을 닫았다 열어도 복원됩니다. 채팅 자체는 목업이 아니라 공용 인프라
chat-server에 위임한 실동작 연동입니다.

**리워드** — 포인트 원장(`point_ledger`)이 언제나 진실이고 잔액 컬럼은 같은 트랜잭션에서 갱신되는
캐시입니다. 멤버십 등급 산정, 만보기(걸음수) 미션, 가입 웰컴 혜택이 이 원장 위에 얹힙니다.

**데이터 규모** — 상품 50종, 카테고리 2-depth(대분류 12 · 소분류 49), 시드 주문 320건(최근 90일 분산).

## 배포

세 앱 모두 **`main` 머지가 곧 배포**이고, 사람이 손으로 올리는 경로는 없습니다(2026-08-06 전환).

| 앱 | 실행 환경 | 배포 주체 |
|---|---|---|
| 고객 웹 · 관리자 웹 | Cloudflare Workers | 깃 연동 빌드(Cloudflare Workers Builds) |
| API 서버 | 클라우드 VM · systemd | GitHub Actions |

**빌드 명령이 곧 검증 게이트입니다.** 프런트 2종은 의존성 설치 → 단위 테스트 → 타입 검사 → 린트 →
워커 번들 빌드가 한 명령의 `&&` 연쇄로 묶여 있어, 앞 단계가 하나라도 실패하면 산출물 자체가 생기지
않고 따라서 배포도 없습니다. 서버 쪽은 **적용 원장을 대조해 미적용 마이그레이션만 순서대로 실행**한 뒤
애플리케이션을 교체하고, 헬스 체크가 끝내 통과하지 못하면 직전 빌드로 자동 롤백합니다. 마이그레이션이
실패하면 애플리케이션을 교체하기 **전에** 멈춥니다 — 새 스키마를 기대하는 코드가 옛 스키마 위에서 도는
상태가 가장 나쁘기 때문입니다.

**CI에 보관하는 시크릿은 0개입니다.** 클라우드 접근은 키 파일 대신 OIDC 기반 연합 인증으로 그때그때
단기 자격증명을 받고, DB 자격증명·서명 키는 서버 호스트에만, LLM 키는 워커 secret에만 둡니다.
배포 경로도 포트를 열지 않고 터널을 통해 붙습니다.

## 다이어그램

모든 다이어그램은 **실제 소스와 대조해** 그렸고, 설계 문서와 구현이 어긋난 지점은 각 문서에
"구현 주의"로 표시했습니다.

### 바운디드 컨텍스트 경계

```mermaid
%%{init: {"theme":"base","fontFamily":"Pretendard, Malgun Gothic, sans-serif","themeVariables":{"fontSize":"14px","primaryColor":"#DBEAFE","primaryBorderColor":"#1D4ED8","primaryTextColor":"#0F172A","lineColor":"#1D4ED8","secondaryColor":"#FEF3C7","tertiaryColor":"#DCFCE7","clusterBkg":"#F8FAFC","clusterBorder":"#CBD5E1","noteBkgColor":"#FEF3C7","noteBorderColor":"#D97706","actorBkg":"#DBEAFE","actorBorder":"#1D4ED8","actorTextColor":"#0F172A","signalColor":"#1D4ED8","signalTextColor":"#0F172A","labelBoxBkgColor":"#DBEAFE","labelBoxBorderColor":"#1D4ED8","altSectionBkgColor":"#F8FAFC"},"flowchart":{"curve":"basis","htmlLabels":true,"padding":12}}}%%
%% 공통 브랜드 테마 — architecture/*.md 전 다이어그램에 동일한 init 블록이 들어간다. 색을 바꿀 때는 이 폴더 전 파일을 일괄 치환할 것.
flowchart LR
    subgraph auth["AUTH · 로그인 서버로 분리 예정"]
        users["users"]
    end

    subgraph commerce["COMMERCE · 커머스 본체"]
        direction TB
        cOrders["orders · reviews<br/>user_coupons · wishlists<br/>inquiries · user_addresses<br/>order_status_history · chat_usage"]
        cCore["categories → products<br/>coupons · order_items"]
    end

    subgraph reward["REWARD"]
        ledger["point_ledger<br/>user_daily_steps"]
    end

    subgraph file["FILE · 파일 서버로 분리 예정"]
        assets["file_assets"]
    end

    subgraph noti["NOTIFICATION · 알림 서버로 분리 예정"]
        notis["notifications<br/>device_tokens"]
    end

    subgraph ops["운영"]
        audit["admin_audit_events"]
    end

    subgraph store["STORE · 완전 독립"]
        stores["stores"]
    end

    users -. "user_id (FK 없음)" .-> cOrders
    users -. "user_id" .-> ledger
    users -. "owner_user_id" .-> assets
    users -. "user_id" .-> notis
    users -. "actor_user_id" .-> audit
    assets -. "profile_image_asset_id" .-> users
    cOrders --- cCore

    classDef ctxAuth fill:#DBEAFE,stroke:#1D4ED8,stroke-width:1.5px,color:#0F172A
    classDef ctxCore fill:#DCFCE7,stroke:#16A34A,stroke-width:1.5px,color:#052E16
    classDef ctxSplit fill:#FEF3C7,stroke:#D97706,stroke-width:1.5px,color:#451A03
    classDef ctxIndep fill:#F1F5F9,stroke:#94A3B8,stroke-width:1.5px,color:#0F172A
    class users ctxAuth
    class cOrders,cCore,ledger ctxCore
    class assets,notis ctxSplit
    class stores,audit ctxIndep
```

### 주문 상태머신

```mermaid
%%{init: {"theme":"base","fontFamily":"Pretendard, Malgun Gothic, sans-serif","themeVariables":{"fontSize":"14px","primaryColor":"#DBEAFE","primaryBorderColor":"#1D4ED8","primaryTextColor":"#0F172A","lineColor":"#1D4ED8","secondaryColor":"#FEF3C7","tertiaryColor":"#DCFCE7","clusterBkg":"#F8FAFC","clusterBorder":"#CBD5E1","noteBkgColor":"#FEF3C7","noteBorderColor":"#D97706","actorBkg":"#DBEAFE","actorBorder":"#1D4ED8","actorTextColor":"#0F172A","signalColor":"#1D4ED8","signalTextColor":"#0F172A","labelBoxBkgColor":"#DBEAFE","labelBoxBorderColor":"#1D4ED8","altSectionBkgColor":"#F8FAFC"},"flowchart":{"curve":"basis","htmlLabels":true,"padding":12}}}%%
%% 공통 브랜드 테마 — architecture/*.md 전 다이어그램에 동일한 init 블록이 들어간다. 색을 바꿀 때는 이 폴더 전 파일을 일괄 치환할 것.
stateDiagram-v2
    direction LR
    [*] --> pending : 주문 생성<br/>재고 차감·쿠폰 사용·포인트 차감
    pending --> paid : mock 결제 완료<br/>(같은 트랜잭션 즉시 전환)
    paid --> shipped : 관리자 전환<br/>trackingNo·carrier 필수
    shipped --> delivered : 관리자 전환<br/>→ ORDER_EARN 적립
    pending --> cancelled : 고객 취소
    paid --> cancelled : 고객 취소
    delivered --> [*]
    cancelled --> [*]

    note right of shipped
        shipped 이후에는 취소 불가.
        delivered·cancelled는 종착 상태로
        나가는 전이가 모두 거부된다.
    end note
```

### 챗봇 파이프라인

```mermaid
%%{init: {"theme":"base","fontFamily":"Pretendard, Malgun Gothic, sans-serif","themeVariables":{"fontSize":"14px","primaryColor":"#DBEAFE","primaryBorderColor":"#1D4ED8","primaryTextColor":"#0F172A","lineColor":"#1D4ED8","secondaryColor":"#FEF3C7","tertiaryColor":"#DCFCE7","clusterBkg":"#F8FAFC","clusterBorder":"#CBD5E1","noteBkgColor":"#FEF3C7","noteBorderColor":"#D97706","actorBkg":"#DBEAFE","actorBorder":"#1D4ED8","actorTextColor":"#0F172A","signalColor":"#1D4ED8","signalTextColor":"#0F172A","labelBoxBkgColor":"#DBEAFE","labelBoxBorderColor":"#1D4ED8","altSectionBkgColor":"#F8FAFC"},"flowchart":{"curve":"basis","htmlLabels":true,"padding":12}}}%%
%% 공통 브랜드 테마 — architecture/*.md 전 다이어그램에 동일한 init 블록이 들어간다. 색을 바꿀 때는 이 폴더 전 파일을 일괄 치환할 것.
flowchart TD
    q(["POST /api/chat<br/>{message}"]) --> parse["요청 검증<br/>본문 4KB · 입력 300자 이하<br/>허용 키는 message 하나"]
    parse --> sess{"대화 회수 소비<br/>서버 chat_usage 정본 (§4)"}
    sess -->|"잔여 없음"| limit["429 · mode=limit<br/>상담 연결 유도"]
    sess -.->|"사용량 서버 불통<br/>fail-open"| pre

    sess -->|"소비 성공"| pre{"① 키워드 프리필터<br/>명백한 비쇼핑 요청?<br/>(코딩·레시피·글쓰기·주식 등)"}
    pre -->|"해당"| refuse["mode=refusal<br/>고정 거절 문구 · 모델 호출 없음"]

    pre -->|"아님"| cls["② 분류 게이트<br/>temperature 0 · 512토큰<br/>쇼핑 질문인가 YES/NO"]
    cls --> yes{"YES?"}
    yes -->|"아니오"| refuse

    yes -->|"예"| rag["③ 미니 RAG (병렬)"]
    rag --> faq["정적 FAQ·정책 10건<br/>키워드 점수 상위 4건"]
    rag --> prod["상품 API 조회 후<br/>토큰 스코어링 상위 3건<br/>→ 상세 재조회 (3초 타임아웃)"]
    faq --> gen
    prod --> gen["④ 답변 생성<br/>temperature 0.2 · 2048토큰<br/>근거 컨텍스트 주입"]
    gen --> mk["⑤ 상품 마커 치환<br/>허용목록 대조 후 블록 버튼 행<br/>문장 경계에서만 분리 (§3.3)"]
    mk --> ok["mode=answer<br/>+ 상품 블록 · 관련 링크"]

    cls -.->|"연결 실패 · 타임아웃 · non-OK"| off["mode=offline (HTTP 200)<br/>고정 문구 3종 회전"]
    gen -.->|"동일"| off
    gen -.->|"응답이 비었거나<br/>추론 태그 유출"| refuse

    llm[["자체 LLM 서빙 서버<br/>OpenAI 호환 /v1<br/>모델은 /v1/models 로 감지<br/>5분 캐시"]]
    cls -.-> llm
    gen -.-> llm

    classDef step fill:#DBEAFE,stroke:#1D4ED8,stroke-width:1.5px,color:#0F172A
    classDef gate fill:#FEF3C7,stroke:#D97706,stroke-width:1.5px,color:#451A03
    classDef fail fill:#FFE4E6,stroke:#E11D48,stroke-width:1.5px,color:#4C0519
    classDef ok fill:#DCFCE7,stroke:#16A34A,stroke-width:1.5px,color:#052E16
    classDef ext fill:#F1F5F9,stroke:#94A3B8,stroke-width:1.5px,color:#0F172A
    class q,parse,cls,rag,faq,prod,gen,mk step
    class sess,pre,yes gate
    class refuse,limit fail
    class ok ok
    class off,llm ext
```

### 태스크 → 구현 → 검증 루프

```mermaid
%%{init: {"theme":"base","fontFamily":"Pretendard, Malgun Gothic, sans-serif","themeVariables":{"fontSize":"14px","primaryColor":"#DBEAFE","primaryBorderColor":"#1D4ED8","primaryTextColor":"#0F172A","lineColor":"#1D4ED8","secondaryColor":"#FEF3C7","tertiaryColor":"#DCFCE7","clusterBkg":"#F8FAFC","clusterBorder":"#CBD5E1","noteBkgColor":"#FEF3C7","noteBorderColor":"#D97706","actorBkg":"#DBEAFE","actorBorder":"#1D4ED8","actorTextColor":"#0F172A","signalColor":"#1D4ED8","signalTextColor":"#0F172A","labelBoxBkgColor":"#DBEAFE","labelBoxBorderColor":"#1D4ED8","altSectionBkgColor":"#F8FAFC"},"flowchart":{"curve":"basis","htmlLabels":true,"padding":12}}}%%
%% 공통 브랜드 테마 — architecture/*.md 전 다이어그램에 동일한 init 블록이 들어간다. 색을 바꿀 때는 이 폴더 전 파일을 일괄 치환할 것.
flowchart TD
    req(["오너 요구사항"]) --> design["총괄 세션: 태스크 설계<br/>목표 · 완료 기준(검증 항목) · 제약"]
    design --> dispatch["인박스에 태스크 투입<br/>ssw-inbox/&lt;수신자&gt;/ 루트에 파일 생성"]

    dispatch --> impl["구현<br/>코덱스 대표 세션 또는 클로드 opus 서브에이전트"]
    impl --> report["구현 보고<br/>'기동 확인' 수준까지만<br/>요구사항 충족 판정은 하지 않는다"]

    report --> vcode["① 코드 검증 (sonnet)<br/>변경 파일 대조 · 정적 점검<br/>gradlew test · dotnet test<br/>npm test · tsc --noEmit · npm run lint"]
    vcode --> vrun["② 런타임 검증<br/>브라우저(웹·관리자 웹)<br/>API 직접 호출<br/>WPF 데스크톱 · 안드로이드 에뮬레이터"]

    vrun --> verdict{"완료 기준을<br/>모두 충족했는가?"}
    verdict -->|"실패"| fail["실패 항목만 회신<br/>검증 세션은 고치지 않는다"]
    fail --> impl
    verdict -->|"통과"| commit["구현 주체가 커밋"]
    commit --> push["push (오너 승인 영역)"]

    push --> ci["③ 파이프라인 검증 (사람 개입 없음)<br/>프런트: Workers 빌드 게이트<br/>서버: Actions test·bootJar 잡"]
    ci --> pass{"통과?"}
    pass -->|"아니오"| fail
    pass -->|"예"| brc{"어느 브랜치인가?"}
    brc -->|"dev"| only["검증까지 — 배포 없음"]
    brc -->|"main"| deploy["자동 배포"]

    classDef step fill:#DBEAFE,stroke:#1D4ED8,stroke-width:1.5px,color:#0F172A
    classDef gate fill:#FEF3C7,stroke:#D97706,stroke-width:1.5px,color:#451A03
    classDef failc fill:#FFE4E6,stroke:#E11D48,stroke-width:1.5px,color:#4C0519
    classDef ok fill:#DCFCE7,stroke:#16A34A,stroke-width:1.5px,color:#052E16
    classDef ext fill:#F1F5F9,stroke:#94A3B8,stroke-width:1.5px,color:#0F172A
    class design,dispatch,impl,report,vcode,vrun,ci step
    class verdict,pass,brc gate
    class fail failc
    class commit,push,deploy ok
    class req,only ext
```

전체 다이어그램은 [`docs/architecture/`](docs/architecture/)의 각 문서 본문에 **mermaid 인라인**으로
들어 있습니다 — 깃허브가 그대로 렌더하므로 별도의 이미지 익스포트는 두지 않습니다.

## 개발 방식 — 멀티 에이전트 협업

이 데모는 여러 AI 에이전트 세션이 나눠 만들었습니다. 규약의 핵심은 하나 — **구현과 검증을
분리한다**는 것입니다. 한쪽이 구현하면 검증은 다른 세션이 맡고, 검증에서 실패가 나오면 검증
세션이 직접 고치지 않고 실패 항목만 구현 쪽에 돌려보냅니다. 같은 세션이 자기 코드를 검증하면
구현할 때 한 착각을 검증에서도 그대로 반복하기 때문입니다. 반대 방향 제약도 있습니다 — 구현
보고는 "기동 확인" 수준까지만 적고 요구사항 충족 판정은 내리지 않습니다. 구현 쪽이 먼저 판정을
내려버리면 검증이 그 결론을 따라가기 때문입니다. 세션 간 소통은 **파일 기반 협업 프로토콜**로
하고(보낸 메시지는 수정하지 않고 정정도 새 메시지로), 태스크 문서의 "완료 기준(검증 항목)" 절이
그대로 검증 체크리스트가 됩니다.

자세한 규약과 다이어그램은 [검증 파이프라인 문서](docs/architecture/08-verification-pipeline.md)에 있습니다.

## 문서

| 경로 | 내용 |
|---|---|
| [`docs/architecture/01-system-architecture.md`](docs/architecture/01-system-architecture.md) | 전체 구성 · 구성 요소 · 인증 흐름 · 챗봇 파이프라인 · 배포 개요 |
| [`docs/architecture/02-data-model-erd.md`](docs/architecture/02-data-model-erd.md) | 전체 ERD · 바운디드 컨텍스트 경계와 논리 참조 |
| [`docs/architecture/03-order-flow.md`](docs/architecture/03-order-flow.md) | 주문 상태머신 · SSW PAY 결제 시퀀스 · 취소 원복 |
| [`docs/architecture/04-review-flow.md`](docs/architecture/04-review-flow.md) | 리뷰 라이프사이클 · 작성 자격 검증 |
| [`docs/architecture/05-reward-flows.md`](docs/architecture/05-reward-flows.md) | 멤버십 등급 산정 · 만보기 미션 · 가입 웰컴 혜택 |
| [`docs/architecture/06-ops-flows.md`](docs/architecture/06-ops-flows.md) | 재고 임계 알림 래치 · 관리 행위 감사 기록 · 주문확인 메일 · 챗봇 대화 회수 관리 |
| [`docs/architecture/07-auth-and-chatbot.md`](docs/architecture/07-auth-and-chatbot.md) | JWT 인증·역할 게이트 · 소셜 가상 로그인 · 챗봇 파이프라인 · 대화 회수 · 상담원 연결 (구현 수준) |
| [`docs/architecture/08-verification-pipeline.md`](docs/architecture/08-verification-pipeline.md) | 태스크→구현→검증 루프 · 인박스 협업 프로토콜 |
| [`docs/architecture/09-infra-integration.md`](docs/architecture/09-infra-integration.md) | 공용 인프라 위임 — 인증 강등 · 이미지 파이프라인 · 관리자 업로드 |
| [`docs/planning/01-project-brief.md`](docs/planning/01-project-brief.md) | 프로젝트 목적 · 메인 포인트 · 확정 사항 |
| [`docs/planning/02-scope.md`](docs/planning/02-scope.md) | 플랫폼 범위 · P0~P2 · Out-of-Scope |
| [`docs/requirements/01-functional.md`](docs/requirements/01-functional.md) | 기능 요구사항 |
| [`docs/requirements/02-non-functional.md`](docs/requirements/02-non-functional.md) | 비기능 요구사항 |
| [`docs/requirements/03-user-stories.md`](docs/requirements/03-user-stories.md) | 유저 스토리 |
| [`docs/planning/03-glossary.md`](docs/planning/03-glossary.md) | 용어집 |
| [`docs/demo-scenario.md`](docs/demo-scenario.md) | 5분/15분 시연 대본 |

원본 워크스페이스는 SDLC-7(01-planning ~ 07-maintenance) 구조로 문서를 관리하며, 이 공개본에는
그중 기획·요구사항·설계 영역을 실었습니다. 배포·유지보수 문서는 포함하지 않습니다.

---

내부 도메인·자격증명 등 공개에 부적절한 값은 플레이스홀더로 치환했습니다.
