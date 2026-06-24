# PRD — 장바구니 할인 계산기 (cursor-tdd-cart)

| 항목 | 내용 |
|------|------|
| **문서 버전** | 1.0 |
| **작성일** | 2026-06-24 |
| **근거 자료** | [Report/01.REPORT.md](../Report/01.REPORT.md), [Report/02.REPORT.md](../Report/02.REPORT.md), [Prompting/01.Export-Transcript.md](../Prompting/01.Export-Transcript.md), [Prompting/02.Export-Transcript.md](../Prompting/02.Export-Transcript.md) |
| **구현 지도** | [AGENTS.md](../AGENTS.md) |

---

## 1. 제품 개요

### 1.1 한 줄 정의

**장바구니 소계를 계산하고, 문턱 할인(10%)·VIP 할인(5%)을 적용한 결제 금액을 산출하는 주문 계산기**이다. Flask 주문 폼(Boundary)과 순수 할인 로직(Entity)으로 분리하며, 모든 동작은 **계약 ID(INV-*, E-*)** 로 추적한다.

### 1.2 문제 정의 (MomTest 근거)

| 출처 | 확인된 문제 |
|------|-------------|
| 인터뷰 메모 (2024-03-12) | A×3 + B×1 = 66,000원, 5만 원 넘으면 10% 할인 기대 59,400원 |
| 주문 #1107 | 소계 60,000 → 결제 54,000 (10% 할인) |
| 주문 #1042 | 소계 48,000 → 할인 없음 |
| CS 클레임 | 수량 -1 입력 시 화면 0원 (기대: 오류·거부) |
| CS 클레임 | items 없이 제출 시 500 에러 |

### 1.3 제품 원칙

1. **MomTest**: 과거 행동·검산·주문 사례만 계약 근거로 삼는다.
2. **ID 없는 동작은 구현하지 않는다** ([AGENTS.md](../AGENTS.md)).
3. **Dual-Track TDD**: RED → GREEN → REFACTOR, Entity/Boundary 테스트 분리.
4. **과잉 구현 금지**: 쿠폰·배송비 등 언급 없는 기능은 OOS.

---

## 2. 범위

### 2.1 In Scope (MVP)

- 품목별 `price × qty` 소계 합산
- 소계 **50,000원 초과** 시 10% 할인 (`round(subtotal × 0.9)`)
- VIP 고객: 문턱 할인 **후** 추가 5% (`round(after_threshold × 0.95)`)
- 할인 적용 순서: **문턱 → VIP** (고정)
- 음수 수량 입력 거부
- 빈 장바구니 제출 시 명시적 오류 (HTTP 500 금지)
- 화면 표시 합계 = 계산 결과

### 2.2 Out of Scope (OOS)

| OOS ID | 제외 항목 | 제외 이유 |
|--------|-----------|-----------|
| OOS-1 | 쿠폰 할인 | 인터뷰 메모 “언급 없음” |
| OOS-2 | 배송비 | 인터뷰 메모 “언급 없음” |
| OOS-3 | 소계 == 50,000원 시 할인 여부 | “넘으면”만 확인, 동일 금액 사례 없음 |
| OOS-4 | VIP + 문턱 미달(48,000) 시 5% 단독 | 사례·금액 없음 |
| OOS-5 | 포인트·등급·세금 | 근거 없음 |
| OOS-6 | 문턱/VIP 적용 순서 변경 | “문턱 → VIP 고정”만 확인 |
| OOS-7 | `qty < 0`을 0으로 보정해 합계 표시 | CS 버그 — E-1로 금지 |
| OOS-8 | `items` 없음 시 HTTP 500 | CS 버그 — E-3로 금지 |

### 2.3 Discovery 아카이브 (본 레포 구현 범위 외)

[Report/02.REPORT.md](../Report/02.REPORT.md) 세션에서 **1인 가구 밀키트/배달 MomTest 인터뷰**로 도출한 계약(INV/E/UC/AC/OOS)은 별도 제품 Discovery 자료이다. 본 PRD의 구현 범위에 **포함하지 않는다**. 요약은 [부록 A](#부록-a-밀키트-도메인-discovery-아카이브) 참조.

---

## 3. 사용자·증거

### 3.1 검증된 사례 (AC → INV 승격)

| ID | 사례 | 기대 결과 |
|----|------|-----------|
| AC-1 | 주문 #1107, 소계 60,000 | 결제 54,000 |
| AC-2 | 주문 #1042, 소계 48,000 | 할인 없음 48,000 |
| AC-3 | 2024-03-12, A(12,000)×3 + B(30,000)×1 | 소계 66,000 → 59,400 (비VIP) |

### 3.2 엑셀 검산 (VIP)

```
66,000 × 0.9 = 59,400 (ROUND)
59,400 × 0.95 = 56,430 (VIP)
```

---

## 4. 계약 레지스트리

> **Entity** (`src/cart.py`, `tests/entity/`): 어떤 입력에서도 깨지면 안 되는 규칙  
> **Boundary** (`src/app.py`, `tests/boundary/`): 입력·제출·표시에서 막아야 할 조건

### 4.1 불변식 (INV)

| ID | 계약 | 근거 레벨 | 계층 | 테스트 아이디어 |
|----|------|-----------|------|-----------------|
| INV-1 | `subtotal(items) == sum(price[i] * qty[i])` | L1 | Entity | `[(12000,3),(30000,1)]` → `66000` |
| INV-2 | `subtotal <= 50_000`이면 `payable == subtotal` | L0 | Entity | `48000` → `48000` (#1042) |
| INV-3 | `subtotal > 50_000`이면 `payable == round(subtotal * 0.9)` | L0 | Entity | `60000` → `54000` (#1107); `66000` → `59400` |
| INV-4 | `is_vip`이고 문턱 할인 후 `after_threshold`이면 `payable == round(after_threshold * 0.95)` | L2 | Entity | VIP `66000` → `56430` |
| INV-5 | VIP 할인은 **항상** 문턱 할인 **이후** 금액에만 적용 (`threshold_first`) | L2 | Entity | `round(round(66000*0.9)*0.95)` ≠ `round(66000*0.95*0.9)` |
| INV-6 | `is_vip == False`이면 VIP 5% 단계 미적용 | L1 | Entity | 비VIP `66000` → `59400` |
| INV-7 | `display_total == payable(subtotal, is_vip)` | L3 | Entity | UI 바인딩 값 = 계산 결과 |

### 4.2 에러 계약 (E)

| ID | 계약 | 근거 레벨 | 계층 | 테스트 아이디어 |
|----|------|-----------|------|-----------------|
| E-1 | `qty[i] < 0`이면 인덱스 `i`를 포함한 `ValueError` | L0 | Boundary | `qty=-1` → 거부, 0원 표시 금지 |
| E-2 | `qty[i] == 0` 정책 | L0 | Boundary | **보류** — MomTest 질문 답 전 RED 금지 |
| E-3 | `items is None` 또는 `len(items)==0`이면 명시 오류, **HTTP 500 금지** | L0 | Boundary | 빈 제출 → 400/422 |

### 4.3 미확인 → 추가 인터뷰 질문

[Prompting/02.Export-Transcript.md](../Prompting/02.Export-Transcript.md) Step 1 기준:

1. 소계 **정확히 50,000원** 주문의 결제 금액·할인 문구
2. 소계 **50,001원** vs **49,999원** 각각의 결제액
3. VIP + 소계 **48,000원** 주문 사례 (#1042와 동일한지)
4. 할인 후 **비정수** 금액의 반올림·절사·올림 규칙
5. **수량 0**·빈 문자열 입력 시 화면·결제 각각의 결과

---

## 5. 기능 요구사항 (MVP)

### 5.1 Entity — 할인 계산

| 기능 | 충족 계약 | 설명 |
|------|-----------|------|
| 소계 합산 | INV-1 | 품목 리스트에서 `price × qty` 합 |
| 문턱 할인 | INV-2, INV-3 | 50,000원 **초과** 시 10% |
| VIP 할인 | INV-4, INV-5, INV-6 | 문턱 후 5%, 비VIP 미적용 |
| 결제액 산출 | INV-1~6 | `payable(subtotal, is_vip)` 단일 진입점 권장 |

### 5.2 Boundary — 주문 폼

| 기능 | 충족 계약 | 설명 |
|------|-----------|------|
| 음수 수량 검증 | E-1 | 폼·API 모두 거부 |
| 빈 장바구니 검증 | E-3 | 500 대신 4xx + 메시지 |
| 합계 표시 | INV-7 | 계산 결과와 동기화 |

---

## 6. 기술·구조

### 6.1 ECB ([AGENTS.md](../AGENTS.md))

```
src/cart.py      Entity    — INV-1~7, Flask import 금지
src/app.py       Boundary  — E-1, E-3, Flask 주문 폼
tests/entity/    pytest -m entity
tests/boundary/  pytest -m boundary
```

### 6.2 TDD 구현 순서

| 순서 | 작업 | 커밋 예시 |
|------|------|-----------|
| 1 | `tests/entity/` INV-1 RED → GREEN | `test(RED): INV-1` / `feat(GREEN): INV-1` |
| 2 | INV-2, INV-3 | 문턱 할인 |
| 3 | INV-4, INV-5, INV-6 | VIP + 순서 |
| 4 | `tests/boundary/` E-1, E-3 | 입력 검증 |
| 5 | INV-7 | 표시=계산 |
| 6 | REFACTOR | `pytest -q` 동작 불변 확인 |

### 6.3 테스트 명령

```bash
pytest -q                  # 전체
pytest tests/entity -q     # INV
pytest tests/boundary -q   # E
```

---

## 7. 성공 기준

- [ ] AC-1~3 주문·검산 사례가 `pytest`로 재현됨
- [ ] INV-1~7 전부 `tests/entity/` 통과
- [ ] E-1, E-3 `tests/boundary/` 통과
- [ ] OOS-1~8에 해당하는 코드·UI 없음
- [ ] 구현 줄에 계약 ID 주석 (`# INV-3` 등)
- [ ] `pytest -q` 전체 통과

---

## 8. 릴리스·문서 이력

| 날짜 | 세션 | 산출 | 참조 |
|------|------|------|------|
| 2026-06-24 | 01 | Export 워크플로 확립 | [Report/01](../Report/01.REPORT.md) |
| 2026-06-24 | 02 | MomTest 계약 도출, 할인 도메인 확정 | [Report/02](../Report/02.REPORT.md) |
| 2026-06-24 | — | 본 PRD 초안 | `docs/PRD.md` |

---

## 부록 A. 밀키트 도메인 Discovery 아카이브

> [Report/02.REPORT.md](../Report/02.REPORT.md) · [Prompting/02.Export-Transcript.md](../Prompting/02.Export-Transcript.md)  
> **본 레포(cursor-tdd-cart) 구현 범위 외.** 별도 제품 기획 시 참고.

### A.1 핵심 결론

- “프리미엄 밀키트 정기구독”은 행동 증거 없음 → **OOS**
- 검증된 니즈: 배달 선택 피로(15~20분), 최소주문·2인분 과주문, 대기(40분+), 혼밥 남음·쓰레기
- MVP 후보(별도 제품): 원탭 재주문, 1인분, 사전 주문, 저녁+야식 번들

### A.2 도출 계약 요약 (미구현)

| 유형 | ID 범위 | 계층 |
|------|---------|------|
| Invariant | INV-1~10 (솔로 1인분, 재주문, ETA 등) | Entity |
| Error | E-1~8 (과주문 유도, 구독 강제 등) | Boundary |
| Use Case | UC-1~5 | Boundary |
| Out of Scope | OOS-1~10 (밀키트 구독, 레시피, 건강·친환경 등) | — |

### A.3 다음 Discovery 질문 (4항)

1. 최소주문·배달비로 지난 1달 **구체 과지출 금액**
2. 배달 외 **시도했다 포기한 대안**
3. 15분 스크롤 줄이려 **유료 앱** 사용 여부
4. “출시하면”이 아니라 **“지난달 이 문제 때문에 산 것”**

---

*본 문서는 docs/PRD.md — Report/·Prompting/ 아카이브 및 AGENTS.md를 근거로 작성된 제품 요구사항 문서입니다.*
