# Repository instructions
 
장바구니 할인 계산기 — Python 3.12, pytest 기반 Dual-Track TDD(C2C 추적) 프로젝트.
이 파일은 백과사전이 아니라 “지도”다.
 
## 구조 (ECB)
- src/cart.py : Entity(순수 로직·불변식). Flask 등 Boundary를 import 하지 않는다.
- src/app.py  : Boundary(Flask 주문 폼). cart.py(Entity)를 import 해 재사용한다.
- tests/entity/    : 불변식(INV-*) 검증
- tests/boundary/  : 입력/UI 계약(E-*, UC-*) 검증

## MomTest
MomTest는 기능 요구사항을 직접 확인하기 위한 도구가 아니라,
실제 사례에서 테스트 가능한 계약을 발견하기 위한 Discovery 도구이다.

AC는 사용자가 받아들이는 조건이고,
Invariant는 어떤 입력에서도 깨지면 안 되는 규칙이다.

모든 테스트와 구현은 계약 ID를 참조한다.
ID가 없는 동작은 구현하지 않는다.

## 테스트 명령
- 전체 : `pytest -q`
- Entity만 : `pytest tests/entity -q`
- Boundary만 : `pytest tests/boundary -q`
 
## 워크플로 — ARRR (RED → GREEN → REFACTOR)
- 테스트를 먼저 쓴다. 구현보다 “실패하는 테스트”가 항상 앞선다.
- RED: tests/ 만 수정. src/ 는 건드리지 않는다.
- GREEN: 실패를 통과시키는 최소 구현만. 구현 줄에 충족한 계약 ID(INV-*/E-*)를 주석으로 단다.
- REFACTOR: 전부 통과한 뒤에만 구조 정리. 정리 전후로 `pytest -q`로 동작 불변 확인.
 
## Tidy First / 커밋
- 구조적 변경과 동작적 변경을 한 커밋에 섞지 않는다(구조 먼저).
- 커밋 분리: test(RED) / feat(GREEN) / refactor(구조). 메시지에 계약 ID를 적는다.
 
## 금지 (Don't)
- assert True · pytest.skip · 예외 삼키기 등 우회로 통과시키지 않는다.
- 요청받지 않은 기존 테스트를 수정·삭제·비활성화하지 않는다.
- 요청하지 않은 기능을 미리 만들지 않는다(과잉 구현 금지).
- 함수 시그니처를 임의로 바꾸지 않는다.
