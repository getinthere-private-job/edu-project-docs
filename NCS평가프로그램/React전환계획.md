# NCS-SSR → React + Spring 전환 계획

## 전환 이유

| 현재 (Mustache SSR) | 전환 후 (React + Spring) |
|---------------------|------------------------|
| 페이지 이동마다 전체 새로고침 | SPA로 빠른 화면 전환 |
| 복잡한 UI 구현 한계 | 컴포넌트 기반으로 복잡한 폼/대시보드 구현 용이 |
| JS 직접 DOM 조작 | React 상태 관리로 깔끔한 코드 |
| PDF 출력이 서버 렌더링 의존 | 클라이언트 PDF 생성 가능 |
| 모바일 대응 어려움 | 반응형 UI 구현 쉬움 |
| 신규 기능 추가시 .mustache + JS 이중 작업 | 프론트/백엔드 완전 분리 |

---

## 아키텍처 변경

```
[현재]
Browser → Spring Controller → Mustache 렌더링 → HTML 응답

[전환 후]
Browser → React SPA → REST API → Spring Boot → JSON 응답
```

---

## 기술 스택

### Backend (Spring Boot 유지 + API 전환)

| 항목 | 현재 | 전환 후 |
|------|------|---------|
| Framework | Spring Boot 3.2 | Spring Boot 3.2 (유지) |
| 템플릿 | Mustache (SSR) | 제거 → REST API만 |
| 응답 형태 | HTML | JSON (ResponseEntity) |
| 인증 | HttpSession | JWT (이미 의존성 있음) |
| API 문서 | 없음 | Swagger/SpringDoc 추가 |
| DB | MySQL + JPA | 유지 |

### Frontend (신규)

| 항목 | 선택 |
|------|------|
| Framework | React 18 |
| 언어 | TypeScript |
| 빌드 | Vite |
| 라우팅 | React Router v6 |
| 상태관리 | Zustand (경량) |
| HTTP 클라이언트 | Axios |
| UI 라이브러리 | Tailwind CSS + shadcn/ui |
| 폼 관리 | React Hook Form |
| 테이블 | TanStack Table |
| PDF 생성 | react-pdf 또는 html2canvas + jsPDF |
| 서명 | react-signature-canvas |

---

## 전환 전략: 점진적 마이그레이션

### Phase 1: API 레이어 분리

기존 Controller를 유지하면서 REST API Controller를 병행 추가

```
현재 Controller (Mustache 반환)     → 유지 (기존 서비스 운영)
신규 API Controller (JSON 반환)     → 추가 (/api/v2/*)
```

**작업 내용**
- 기존 Service 로직은 그대로 재사용
- Controller만 `@RestController`로 새로 작성
- Request/Response DTO는 기존 것 활용 or JSON용으로 정리
- JWT 인증 필터 활성화

### Phase 2: React 프론트엔드 구축

**프로젝트 구조**
```
ncs-web/
├── src/
│   ├── api/              # API 호출 함수
│   ├── components/       # 공통 컴포넌트
│   │   ├── ui/           # shadcn 컴포넌트
│   │   ├── layout/       # Header, Sidebar, Footer
│   │   └── common/       # Table, Modal, Form 등
│   ├── pages/            # 페이지 컴포넌트
│   │   ├── auth/         # 로그인, 회원가입
│   │   ├── course/       # 과정 관리
│   │   ├── subject/      # 교과목 관리
│   │   ├── paper/        # 시험지 관리
│   │   ├── exam/         # 시험 응시/결과 (학생)
│   │   ├── grading/      # 채점 (강사)
│   │   ├── observation/  # 관찰일지 (신규)
│   │   ├── evaluation/   # 프로젝트 평가표 (신규)
│   │   ├── mentoring/    # 멘토링일지 (신규)
│   │   ├── team/         # 팀 관리 (신규)
│   │   ├── document/     # 보고서
│   │   └── dashboard/    # 대시보드 (신규)
│   ├── hooks/            # 커스텀 훅
│   ├── store/            # Zustand 스토어
│   ├── types/            # TypeScript 타입
│   └── utils/            # 유틸리티
```

### Phase 3: 신규 기능 동시 개발

Phase 2와 병행하여 신규 기능(관찰일지, 평가표, 멘토링일지, 팀관리)을 React + API로 바로 개발

### Phase 4: Mustache 제거

모든 화면이 React로 전환 완료되면:
- Mustache 템플릿 삭제
- 기존 SSR Controller 삭제
- Spring Boot는 순수 API 서버로 운영

---

## 화면별 전환 매핑

### 기존 화면 → React 페이지

| 기존 Mustache | React 페이지 | 비고 |
|--------------|-------------|------|
| login-form | /login | |
| join-form | /join | |
| sign-form | /sign | react-signature-canvas |
| course/list | /courses | 페이징 + 검색 |
| course/detail | /courses/:id | 탭 (교과목/학생/팀) |
| course/save-form | /courses/new | React Hook Form |
| paper/course-list | /papers | 과정 선택 |
| paper/subject-list | /papers/course/:id | 교과목 선택 |
| paper/paper-list | /papers/subject/:id | 시험지 목록 |
| paper/save-form | /papers/new | 평가방법 선택 |
| paper/mcq-detail | /papers/:id | 조건부 렌더링 |
| paper/rubric-detail | /papers/:id | 조건부 렌더링 |
| question/save-form | /papers/:id/questions/new | 동적 폼 |
| exam/list | /grading/subject/:id | 결과 목록 |
| exam/detail | /grading/exam/:id | 채점 UI |
| student/paper/list | /student/exams | 응시 가능 목록 |
| student/mcq-start | /student/exams/:id/start | 시험 UI |
| student/exam-result | /student/results | 결과 목록 |
| document/* | /documents/* | PDF 미리보기 |

### 신규 화면 (React만)

| React 페이지 | 설명 |
|-------------|------|
| /teams | 팀 목록/관리 (노션 URL, GitHub URL 등록) |
| /teams/:id | 팀 상세 (노션/GitHub 링크, 팀원 역할) |
| /observation | 관찰일지 목록 |
| /observation/new | 관찰일지 입력 폼 |
| /observation/:id | 관찰일지 상세/PDF |
| /project-eval | 프로젝트 평가표 목록 |
| /project-eval/new | 평가표 입력 (17항목 + 총평) |
| /project-eval/:id | 평가표 상세/PDF |
| /mentoring | 멘토링일지 목록 |
| /mentoring/new | 멘토링일지 입력 |
| /mentoring/:id | 멘토링일지 상세/PDF |
| **/dashboard** | **단일 진입점 - 15종 서류 대시보드** |
| /dashboard/:teamId | 팀별 서류 상세 (PDF보기 + 노션/GitHub 링크) |

### 대시보드 (단일 진입점) 상세

> 국가 심사관/강사가 이 화면 하나로 모든 서류를 확인

**과정별 팀 목록 (/dashboard)**
- 팀명, 프로젝트명, 노션 링크, GitHub 링크
- 팀별 서류 진행률 프로그레스 바 (n/15)
- 팀 클릭 → 서류 상세로 이동

**팀별 서류 상세 (/dashboard/:teamId)**
- 15종 서류를 3개 그룹으로 구분 표시
  - 강사 서류 (홈페이지 내부) → [PDF 보기] 버튼 → 인라인 PDF 뷰어
  - 학생 서류 (노션) → [노션 열기] 버튼 → 새 탭에서 노션 페이지 열림
  - 소스코드 (GitHub) → [GitHub 열기] 버튼 → 새 탭에서 GitHub 열림
- 서류별 상태 아이콘 (미작성/작성중/완료)
- [PDF 일괄 다운로드] → 강사 서류 전체 ZIP 파일

---

## API 전환 설계

### 인증 변경

```
[현재] HttpSession 기반
POST /login → 세션 생성 → 쿠키

[전환] JWT 기반
POST /api/v2/auth/login → JWT 발급 → Authorization 헤더
POST /api/v2/auth/refresh → 토큰 갱신
```

### 기존 엔드포인트 전환

```
현재 (SSR)                              전환 (REST API)
────────────────────────              ──────────────────────────
GET  /api/course-menu/course          → GET  /api/v2/courses
POST /api/course-menu/course/save     → POST /api/v2/courses
GET  /api/paper-menu/paper/{id}       → GET  /api/v2/papers/{id}
POST /api/paper-menu/paper/save       → POST /api/v2/papers
GET  /api/exam-menu/exam/{id}         → GET  /api/v2/exams/{id}
PUT  /api/exam-menu/exam/{id}         → PUT  /api/v2/exams/{id}
POST /api/student/exam/mcq            → POST /api/v2/student/exams/mcq
GET  /api/document-menu/subject/{id}/no1 → GET /api/v2/documents/{subjectId}/no1
```

### 신규 엔드포인트

```
# 팀 관리
GET    /api/v2/courses/{courseId}/teams
POST   /api/v2/courses/{courseId}/teams
PUT    /api/v2/teams/{teamId}

# 관찰일지
GET    /api/v2/courses/{courseId}/observations
POST   /api/v2/observations
GET    /api/v2/observations/{id}
GET    /api/v2/observations/{id}/pdf

# 프로젝트 평가표
GET    /api/v2/courses/{courseId}/project-evals
POST   /api/v2/project-evals
GET    /api/v2/project-evals/{id}
GET    /api/v2/project-evals/{id}/pdf

# 멘토링일지
GET    /api/v2/courses/{courseId}/mentoring-logs
POST   /api/v2/mentoring-logs
GET    /api/v2/mentoring-logs/{id}
GET    /api/v2/mentoring-logs/{id}/pdf

# 대시보드 (단일 진입점)
GET    /api/v2/courses/{courseId}/dashboard         # 팀 목록 + 서류 진행률
GET    /api/v2/dashboard/teams/{teamId}             # 팀별 15종 서류 상세
PUT    /api/v2/dashboard/teams/{teamId}/status       # 서류 상태 업데이트
GET    /api/v2/dashboard/teams/{teamId}/download-all # 강사 서류 PDF 일괄 다운로드
```

---

## 전환 일정 (제안)

| Phase | 내용 | 기간 |
|-------|------|------|
| **Phase 1** | API 레이어 분리 + JWT 인증 | 1주 |
| **Phase 2-1** | React 기본 세팅 + 인증/과정/교과목 | 2주 |
| **Phase 2-2** | 시험지/시험응시/채점 화면 | 2주 |
| **Phase 2-3** | 보고서 화면 + PDF | 1주 |
| **Phase 3** | 신규 기능 (팀/관찰일지/평가표/멘토링) | 2주 |
| **Phase 4** | Mustache 제거 + 통합 테스트 | 1주 |

---

## 주의사항

1. **Service 로직은 건드리지 않는다** - Controller만 교체
2. **기존 운영과 병행** - Phase 1~3 동안 Mustache 버전도 유지
3. **DB 스키마는 확장만** - 기존 테이블 변경 최소화, 신규 테이블 추가
4. **JWT와 세션 병행** - 전환 기간 동안 두 인증 방식 모두 지원
5. **PDF 생성** - 서버사이드(기존) + 클라이언트사이드(신규) 모두 지원
