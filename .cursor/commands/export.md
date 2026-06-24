# export — 세션 보고서 + Transcript Export (`/export-session` 별칭)

**추가 입력 없이 즉시 실행.** 사용자가 `/export` 만 입력했다. 세션 주제·산출물·대화 내용은 **현재 채팅 전체**에서 자동 추출한다. 추가 질문·확인 요청 금지.

## 자동 추출 (사용자에게 묻지 말 것)
- **세션 주제**: 이번 대화의 핵심 작업 (예: TDD RED, Command 작성, pytest 실행)
- **산출물**: 생성·수정된 파일 목록
- **Transcript**: User/Cursor 턴 전체

## 번호 규칙
1. `Report/`·`Prompting/`의 기존 `NN.*` 파일을 확인한다.
2. 가장 큰 번호 + 1을 다음 번호로 쓴다. (예: 04까지 있으면 → 05)
3. 번호는 **2자리** (`01`, `02`, … `05`).

## 생성 파일 (반드시 2개)

| 파일 | 설명 |
|------|------|
| `Report/NN.REPORT.md` | 세션 요약 보고서 |
| `Prompting/NN.Export-Transcript.md` | 대화 전문 Export |

## 보고서 형식 (`Report/NN.REPORT.md`)
- 제목: `# MagicSquare_1004 — {자동 추출한 세션 주제}`
- 상단 메타 표: 프로젝트, 단계, 보고서 생성일, 목적
- 섹션: 1. 요약 / 2. 핵심 결정·산출물 / 3. 다음 단계
- 관련 Transcript 링크: `Prompting/NN.Export-Transcript.md`
- 마지막 줄: `*본 문서는 Report/NN.REPORT.md — …입니다.*`

## Transcript 형식 (`Prompting/NN.Export-Transcript.md`)
- 제목 + `_Exported on {오늘 날짜} from Cursor_`
- **User** / **Cursor** 턴별로 대화 재구성 (요약이 아닌 전문)
- 마지막에 생성·변경 파일 목록 표
- 관련 보고서 링크: `Report/NN.REPORT.md`
- 마지막 줄: `*본 문서는 Prompting/NN.Export-Transcript.md — …입니다.*`

## 절차
1. `Report/`, `Prompting/`에서 다음 번호(NN)를 결정한다.
2. 현재 대화에서 주제·내용을 추출한다.
3. 위 형식으로 두 파일을 **직접 생성**한다.
4. 짧게 보고: 번호, 파일 경로, 세션 주제 한 줄.

## 금지
- 사용자에게 세션 주제·번호·형식 **추가 질문**
- 기존 번호 파일 덮어쓰기
- 번호 없이 저장 (`REPORT.md` 단독명 금지)
- 보고서만 만들고 Transcript 생략 (또는 그 반대)
