# Web Planning Automation SaaS 구축 제안서

## 1) 목표 요약

클라이언트가 **설문형 질문**에 답하면,
1. AI가 업종/주제 기반으로 기획안(문서) 생성
2. 브랜드 컬러/디자인 방향 자동 제안
3. 정해진 랜딩페이지 템플릿에 데이터 주입
4. 단시간 내 랜딩페이지 자동 배포

까지 한 번에 처리하는 SaaS를 구축합니다.

---

## 2) 권장 아키텍처 (요구사항 기준)

- **Frontend**: Next.js (Vercel 배포)
- **Data Store**: Google Sheets (설문 응답/기획안 메타데이터 저장)
- **Automation**: n8n (NAS 서버 self-hosted)
- **AI Agent**: OpenAI API (기획안/브랜드 톤/컬러/카피 생성)
- **Notification**: Email + Slack + Telegram (n8n에서 동시 처리)
- **Landing Generator**:
  - 방법 A: Vercel Deploy Hook + 템플릿 repo 데이터 주입
  - 방법 B: 템플릿 앱에서 동적 라우팅으로 즉시 페이지 생성

---

## 3) 사용자 흐름 (End-to-End)

1. 사용자가 설문 시작
2. 업종/주제 선택
3. 기본 질문 + 업종별 추가 질문 응답
4. 제출 시 Next.js API Route가 payload 생성
5. payload를 n8n Webhook으로 전달
6. n8n이 다음 작업 수행
   - Google Sheets에 원본 응답 저장
   - AI 호출로 기획안/디자인 제안 생성
   - 결과를 Google Sheets에 업데이트
   - 알림(Email/Slack/Telegram) 전송
   - 랜딩페이지 생성 트리거 (Vercel hook 또는 API)
7. 완료 URL을 사용자/운영자에게 회신

---

## 4) 설문 설계 전략

### 4.1 공통 질문(고정)
- 서비스/제품명
- 한 줄 소개
- 주요 타겟 고객
- 해결하려는 문제
- 원하는 행동(문의, 구매, 예약 등)
- 레퍼런스 사이트(선택)

### 4.2 업종별 질문(동적)
- 예: 교육, 뷰티, 병원, IT SaaS, 커머스
- 업종별 KPI/필수 섹션 질문 분기

### 4.3 AI 보조 질문
- 답변 누락/모호한 항목 감지
- AI가 후속 질문 2~3개 제안
- 최소 입력만으로도 기획안 품질 보정

---

## 5) AI Agent 기능 정의

### 5.1 출력물
- 페이지 목적/핵심 메시지
- 섹션 구성(히어로, 신뢰요소, 기능, FAQ, CTA)
- 카피 초안(헤드라인/서브카피/버튼)
- 브랜드 톤(전문적/친근함/프리미엄 등)
- 컬러 팔레트(Primary/Secondary/Accent + Hex)
- 디자인 가이드(폰트 스타일, 이미지 톤)

### 5.2 프롬프트 구조 권장
- System: "랜딩페이지 기획 전문가"
- Developer: 출력 JSON 스키마 강제
- User: 설문 응답 + 업종 + 목표

### 5.3 품질 보완
- 금지어/과장표현 필터
- 업종별 규제(의료/금융) 주의 문구 템플릿
- 결과 JSON validation 실패 시 재시도

---

## 6) Google Sheets 데이터 모델 (최소)

### 시트 1: `responses`
- submission_id
- created_at
- client_email
- industry
- topic
- answers_json
- status (`received`, `planned`, `deployed`, `failed`)

### 시트 2: `plans`
- submission_id
- summary
- sections_json
- copy_json
- brand_tone
- palette_json
- design_notes
- landing_url
- updated_at

### 시트 3: `logs`
- submission_id
- step
- result
- error_message
- timestamp

---

## 7) n8n 워크플로우 설계 (NAS 기준)

### Workflow A: 설문 접수 → 기획안 생성
1. **Webhook Trigger** (Next.js에서 POST)
2. **Set/Function**: 데이터 정규화
3. **Google Sheets Append** (`responses`)
4. **OpenAI Node / HTTP Request**: 기획안 생성
5. **IF Node**: JSON schema 유효성 체크
6. **Google Sheets Update** (`plans`, `responses.status=planned`)
7. **Slack/Telegram/Email Node**: 생성 완료 알림

### Workflow B: 기획안 확정 → 랜딩 생성
1. Trigger (A 후속 또는 수동 승인)
2. 템플릿 입력 데이터 생성
3. Vercel Deploy Hook 호출 또는 템플릿 API 호출
4. 배포 URL 획득
5. Google Sheets 업데이트 (`landing_url`, `status=deployed`)
6. 운영자/클라이언트 최종 알림

### 장애 대응
- 실패 시 `logs` 시트 기록
- 3회 재시도 + 최종 실패 알림
- submission_id 기준 idempotency 처리

---

## 8) Vercel 배포 패턴

### 패턴 1: 동적 렌더링(권장 초기)
- 하나의 Next.js 앱에서 `/lp/[submissionId]` 라우트 제공
- Google Sheets 또는 캐시DB에서 데이터 조회
- 장점: 빠른 구현, 재배포 필요 없음

### 패턴 2: 개별 정적 빌드
- 제출 건마다 템플릿 데이터를 주입해 별도 배포
- 장점: 독립 URL/성능 최적화
- 단점: 파이프라인 복잡도 증가

---

## 9) 보안/운영 체크리스트

- n8n Webhook secret 토큰 검증
- Google Service Account 키는 NAS/Vercel Secret으로 관리
- OpenAI API Key 권한 분리
- PII(이메일/전화번호) 저장 최소화
- 요청량 제한(rate limiting)
- 감사 로그(누가/언제/무엇을 생성했는지)

---

## 10) 단계별 구축 로드맵 (4주 예시)

### 1주차: MVP 설문 + 저장
- Next.js 설문 UI
- Google Sheets 연동
- n8n Webhook 수신

### 2주차: AI 기획안 생성
- 프롬프트/JSON 스키마 고정
- plans 시트 자동 기록
- 알림(슬랙/텔레그램/이메일)

### 3주차: 랜딩 자동생성
- 템플릿 페이지 연결
- 배포 URL 자동 반환
- 실패 재처리 로직

### 4주차: 고도화
- 업종별 질문 분기 강화
- 관리자 대시보드(진행상태/재실행)
- 품질평가 기준(완성도 스코어)

---

## 11) 구현 시 추천 기술 스택 상세

- Frontend: Next.js (App Router), TypeScript, Tailwind
- API: Next.js Route Handlers
- Automation: n8n (Docker on NAS)
- Integration: Google Sheets API, Slack Bot, Telegram Bot, SMTP or Gmail API
- AI: OpenAI Responses API (JSON schema output)

---

## 12) 바로 시작 가능한 MVP 범위

**입력**: 업종 + 10개 내외 질문
**출력**: 기획안 JSON + 단일 랜딩 템플릿 URL
**자동화**: 시트 저장 + 알림 3종 + URL 회신

이 범위면 작은 팀 기준으로도 빠르게 검증이 가능합니다.
