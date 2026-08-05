# 용어집 (Glossary)

> 상태: ✅ 확정 · 최종수정: 2026-07-03

프로젝트 전반에서 쓰는 용어를 통일한다.

## 제품 / 조직
- **SSW / SWorkAgency** — 이 데모를 만든 주체(에이전시). 데모의 진짜 목적은 이 조직의 AI 오케스트레이션·멀티플랫폼 역량 쇼케이스.
- **SSW E-Commerce Demo** — 본 프로젝트. 이커머스 핵심 플로우(탐색 → 장바구니 → mock 결제) 시연 데모.

## 딜리버러블 / 역할별 앱
접두어 규칙 — 고객용은 무접두어(`web`/`android`), 관리·직원용은 `admin-*`.

- **web** — 고객 웹. `ssw-e-commerce-demo-web/`, Next.js 16 App Router (실제 UI).
- **server** — 백엔드. `ssw-e-commerce-demo-server/`, Spring Boot 3.4.1 REST API(`/api/v1`).
- **android** — 고객 안드로이드. `ssw-e-commerce-demo-android/`, WebView 셸 + 네이티브 하단탭.
- **admin-web** — 관리자 웹. 대시보드·주문/상품/재고/회원/리뷰/쿠폰 관리. (신규 개발)
- **admin-windows** — 직원용 Windows WPF 데스크톱 앱. PC 대량관리·AI 운영보조. (신규 개발)
- **prototype** — `design/prototype/shop-demo/…dc.html`. 포팅 원본(시각 충실도 **기준**).
- ~~admin-android~~ — 관리자용 안드로이드 앱. **보류**(후속 검토, 이번 범위 제외).

## 동작 모드 / 식별
- **Mock 모드 / HTTP 모드** — 웹 repository 팩토리 스위치(`NEXT_PUBLIC_API_MODE`). 기본 `mock`(클라 목데이터), `http`면 백엔드 연동. 현재 catalog/order만 http.
- **mock 결제** — 실제 PG 없이 status를 바로 `paid`로 처리하는 데모 결제.
- **X-Demo-User** — 현재 사용자 식별 헤더(기본 `mock_user_001`). JWT 도입 전 임시.
- **JS 브릿지** — 안드 WebView ↔ 웹 통신. `window.__sswNavigate`(네비, 미구현), `AndroidBridge.onRouteChanged`/`onThemeChanged`(통지).

## 도메인 엔티티
바운디드 컨텍스트 6분할(v2.1 데이터 모델 기준):
- **AUTH**: users
- **COMMERCE**: categories · products · orders · order_items · reviews · coupons · user_coupons · wishlists · order_status_history · inquiries · user_addresses
- **REWARD**: point_ledger
- **FILE**: file_assets
- **NOTIFICATION**: device_tokens · notifications
- **STORE**: stores

※ 커머스 도메인 **내부**는 실제 FK, **서비스 경계를 넘는 참조**(user_id, 파일, 알림 등)는 FK 없는 논리 참조.
※ 대부분의 엔티티는 실삭제 대신 상태 컬럼 변경(소프트 삭제)으로 관리한다.
※ 상세 정본: 내부 데이터 모델 설계 문서(비공개, v2.2).

## 기획 / 회의 용어
- **SDLC-7** — 이 프로젝트 문서의 7영역 표준 구조(01-planning ~ 07-maintenance).
- **승격** — 확정 문서를 회의 임시폴더에서 `docs`(정본)로 이동.
- **AX** — Agent/AI eXperience, "AI 주도 워크플로 작업 효율" — 이 데모가 증명하려는 셀링포인트.
- **P0 / P1 / P2** — 우선순위 (P0 = MVP 필수, P1/P2 = 후속).
