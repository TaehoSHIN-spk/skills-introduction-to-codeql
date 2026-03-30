# 🛡️ Introduction to CodeQL — 단계별 실습 학습 가이드

> **GitHub Skills "Introduction to CodeQL" 실습 과정**을 단계별로 정리한 가이드입니다.
> 이 저장소의 `.github/steps/` 폴더에 있는 Step 1~4 + Review 내용을 기반으로,
> **CodeQL의 활성화 → 취약점 탐지 → 알림 관리 → 취약점 수정**까지의 전체 흐름을 한눈에 파악할 수 있습니다.

---

## 📋 목차

1. [전체 실습 흐름 요약](#-전체-실습-흐름-요약)
2. [Step 1: Code Scanning 활성화](#step-1--code-scanning-활성화)
3. [Step 2: Pull Request에서 취약점 탐지](#step-2--pull-request에서-취약점-탐지)
4. [Step 3: CodeQL 알림 검토 및 분류](#step-3--codeql-알림-검토-및-분류)
5. [Step 4: 보안 취약점 수정](#step-4--보안-취약점-수정)
6. [Review: 학습 완료 및 다음 단계](#-review-학습-완료-및-다음-단계)
7. [핵심 개념 정리](#-핵심-개념-정리)
8. [참고 자료](#-참고-자료)

---

## 🔄 전체 실습 흐름 요약

```
Step 1: CodeQL 활성화
  → Repository Settings에서 Code Scanning 켜기
    → Step 2: 취약한 코드 도입
      → PR에서 SQL Injection 취약점이 포함된 코드 작성
        → Step 3: 알림 검토 및 분류
          → Security 탭에서 CodeQL 알림 확인 & 분석
            → Step 4: 취약점 수정
              → 안전한 코드로 수정 후 커밋
                → Alert 자동 해소 확인 ✅
```

| 단계 | 핵심 키워드 | 학습 목표 |
|------|-----------|----------|
| **Step 1** | 활성화 | CodeQL & Code Scanning 개념 이해, 설정 방법 |
| **Step 2** | 탐지 | 취약한 코드 작성 → PR에서 자동 탐지 확인 |
| **Step 3** | 분류 | Security 탭에서 알림 관리, CWE 이해 |
| **Step 4** | 수정 | 취약점 수정 → Alert 해소 → 감사 추적 확인 |

---

## Step 1 — Code Scanning 활성화

### 📖 배경 지식

#### GitHub Code Scanning이란?

[Code Scanning](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/about-code-scanning)은 **GitHub Advanced Security (GHAS)** 제품의 일부입니다.

- 개발 팀이 **기존 코드 배포 프로세스에 보안 테스트 도구를 직접 통합**할 수 있게 해줍니다
- SAST, 컨테이너, IaC(Infrastructure as Code) 등 다양한 유형을 지원합니다
- 결과가 **GitHub 내에서 코드 바로 옆에 표시**되므로, 컨텍스트 전환이 필요 없습니다 🎉

> 💡 **비용 안내:** GitHub Advanced Security의 모든 기능은 **공개(public) 저장소에서 무료**입니다.
> 비공개(private) 저장소에서는 [유료 플랜](https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-advanced-security/about-billing-for-github-advanced-security)이 필요합니다.

#### CodeQL이란?

[CodeQL](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/about-code-scanning-with-codeql)은 **SQL Injection, XSS, Code Injection** 등의 보안 취약점을 식별하는 정적 분석 도구입니다.

- **쿼리(Query):** 보안 패턴을 탐지하는 개별 규칙
- **쿼리 스위트(Query Suite):** 여러 쿼리를 묶은 패키지
- 보안 전문가 팀이 **다양한 시나리오와 프로그래밍 언어별로 미리 구성한 스위트**를 제공합니다

#### 기본 설정(Default Configuration) 옵션

| 옵션 | 설명 |
|------|------|
| **Languages** | 저장소의 언어를 자동 감지하여 스캐닝 활성화 |
| **Query suites** | 사용할 쿼리 패턴 스위트 선택 (**Default** 또는 **Extended**) |
| **Runner type** | CodeQL 분석을 실행할 GitHub Actions 러너 (기본: GitHub Hosted) |
| **Events** | 스캔 실행 트리거 (머지 전 / 스케줄 등) |

### ⌨️ 실습 활동: CodeQL로 Code Scanning 활성화하기

```
1. 저장소의 Code 탭으로 이동
2. 상단 네비게이션에서 Settings 탭 선택
3. 좌측 Security 섹션 → Advanced Security 선택
4. Code scanning 영역에서 CodeQL → Set up → Default 선택
5. Enable CodeQL 클릭
```

> 💡 이 작업이 완료되면 CodeQL의 첫 번째 실행이 트리거됩니다. **Actions** 탭에서 진행 상황을 확인할 수 있습니다.

---

## Step 2 — Pull Request에서 취약점 탐지

### 📖 배경 지식

Code Scanning이 실제로 어떻게 동작하는지 확인하기 위해, **의도적으로 취약한 코드를 도입**하여 알림을 트리거합니다.

이 저장소의 `server/routes.py` 파일은 Flask 기반의 Python 웹 서버로, 도서 검색 기능을 제공합니다.

### ⌨️ 실습 활동 1: 취약점 생성하기

**안전한 코드 (원본):**
```python
# server/routes.py - line 16
"SELECT * FROM books WHERE name LIKE %s", name
```

**취약한 코드 (수정 후) — SQL Injection:**
```python
# server/routes.py - line 16
"SELECT * FROM books WHERE name LIKE '%" + name + "%'"
```

#### 무엇이 문제인가?

| 구분 | 안전한 코드 | 취약한 코드 |
|------|-----------|-----------|
| **방식** | 파라미터화된 쿼리 (`%s` 플레이스홀더) | 문자열 연결(concatenation) |
| **사용자 입력 처리** | DB 드라이버가 자동으로 이스케이프 | **입력값이 SQL 쿼리에 직접 삽입** |
| **위험성** | ✅ 안전 | ❌ SQL Injection 가능 |
| **공격 예시** | — | `name = "'; DROP TABLE books; --"` |

### ⌨️ 실습 활동 2: PR 생성 및 검토

```
1. Code 탭 → server/routes.py 파일 편집
2. line 16을 취약한 코드로 변경
3. "Create a new branch" 선택 → 브랜치명: learning-codeql
4. Propose changes → Create pull request
5. PR 하단에서 CodeQL 체크 실행 확인
6. 분석 완료 후 SQL Injection 취약점 탐지 결과 확인
```

> 💡 **Show paths** 링크를 클릭하면, 사용자 입력(Source)에서 SQL 실행(Sink)까지의 **데이터 흐름 경로**를 확인할 수 있습니다.

### ⌨️ 실습 활동 3: CodeQL 스캐닝 로그 확인

```
1. Actions 탭 → 좌측에서 CodeQL 워크플로우 필터
2. PR #2 이름의 워크플로우 실행 클릭
3. 상세 분석 로그 확인
```

---

## Step 3 — CodeQL 알림 검토 및 분류

### 📖 배경 지식

GitHub는 보안 관련 이슈를 안전하게 관리하기 위한 전용 **Security** 탭을 제공합니다.
CodeQL 결과는 다른 분석 도구와 동일한 표준(SARIF)으로 저장되며, **Code scanning** 영역에 표시됩니다.

#### 알림(Alert)이 제공하는 정보

| 정보 | 설명 |
|------|------|
| **해결 상태** | Open / Dismissed / Fixed |
| **영향 받는 브랜치** | 취약점이 존재하는 브랜치 |
| **코드 위치** | 파일명, 라인 번호 |
| **심각도(Severity)** | Critical / High / Medium / Low |
| **CVE 식별 번호** | 해당하는 경우 공개 취약점 ID |
| **상세 설명** | 문제점, 권장 해결 방법, 코드 수정 제안 |

#### CWE (Common Weakness Enumeration)란?

- 하드웨어 및 소프트웨어 취약점에 대한 **카테고리 시스템**
- 보안 이슈를 **표준화된 방식으로 분류하고 설명**하는 체계
- CodeQL이 탐지하는 많은 패턴이 기존 CWE 데이터베이스에서 비롯됨
- 예: CWE-89 (SQL Injection), CWE-79 (XSS), CWE-78 (OS Command Injection)

### ⌨️ 실습 활동 1: 기존 알림 확인하기

```
1. Security 탭 → 좌측 Vulnerability alerts → Code scanning
2. 현재는 알림 없음 (취약한 코드가 main에 머지되지 않았으므로)
3. PR로 돌아가서 Merge pull request 클릭
4. Delete branch 클릭
5. CodeQL이 main 브랜치의 새 변경사항을 분석할 때까지 대기
6. Security 탭으로 복귀 → Code scanning에 1개 알림 확인
```

### ⌨️ 실습 활동 2: 알림 상세 검토

```
1. Code scanning 클릭 → 열린 알림 선택
2. 확인 사항:
   ├── 취약점 설명 (Description)
   ├── 취약점 상세 (Vulnerability details)
   ├── 권장 해결 방법 (Recommendation)
   └── 코드 수정 제안 (Suggested fix)
```

---

## Step 4 — 보안 취약점 수정

### 📖 배경 지식

CodeQL이 제공하는 정보를 활용하여 취약점을 이해하고 수정합니다.
알림의 **권장 해결 방법**과 **코드 수정 제안**을 참고하면 효과적으로 수정할 수 있습니다.

### ⌨️ 실습 활동: 열린 알림 해결하기

**수정 방법:** 문자열 연결 → 파라미터화된 쿼리로 변경

```python
# ❌ 취약한 코드 (Before)
"SELECT * FROM books WHERE name LIKE '%" + name + "%'"

# ✅ 안전한 코드 (After)
"SELECT * FROM books WHERE name LIKE %s", name
```

#### 왜 이 수정이 안전한가?

| 항목 | 설명 |
|------|------|
| **파라미터화된 쿼리** | `%s` 플레이스홀더를 사용하여 DB 드라이버가 입력값을 안전하게 처리 |
| **자동 이스케이프** | 특수 문자(`'`, `"`, `;` 등)가 자동으로 이스케이핑됨 |
| **SQL 구조 보호** | 사용자 입력이 SQL 구문을 변경할 수 없음 |

### 수정 후 확인 절차

```
1. Code 탭 → main 브랜치 → server/routes.py 편집
2. line 16을 안전한 코드로 수정
3. Commit changes (main 브랜치에 직접 커밋)
4. CodeQL이 재스캔 실행될 때까지 대기
5. Security 탭 → Code scanning 확인
   ├── 열린 알림: 0개
   └── 닫힌 알림: 1개 → 수정으로 자동 해소됨 🎉
6. Closed 버튼 클릭 → 닫힌 알림 확인
7. 알림의 감사 추적(Audit Trail)에서 수정 이력 확인
```

---

## ✅ Review: 학습 완료 및 다음 단계

### 이 실습에서 배운 것

| 번호 | 학습 내용 | 관련 Step |
|------|----------|----------|
| 1 | CodeQL로 **Code Scanning을 활성화**하는 방법 | Step 1 |
| 2 | Pull Request를 통해 **취약점을 도입하고 탐지**하는 과정 | Step 2 |
| 3 | CodeQL 알림을 **검토하고 분류(triage)** 하는 방법 | Step 3 |
| 4 | 보안 취약점을 **수정하고 알림 해소를 검증**하는 방법 | Step 4 |

### 핵심 교훈

```
✅ CodeQL은 PR 단계에서 보안 취약점을 자동으로 탐지합니다
✅ Security 탭에서 모든 보안 알림을 중앙 관리할 수 있습니다
✅ 취약점을 수정하면 Alert이 자동으로 해소됩니다
✅ 감사 추적(Audit Trail)으로 처리 이력을 확인할 수 있습니다
✅ 정기적인 보안 알림 검토는 건강한 프로젝트 유지의 핵심입니다
```

---

## 📚 핵심 개념 정리

### Code Scanning 전체 아키텍처

```
개발자가 코드 작성 & PR 생성
  → GitHub Actions에서 CodeQL 워크플로우 실행
    → 소스 코드를 CodeQL 데이터베이스로 변환
      → QL 쿼리 실행 (Default / Extended / Custom)
        → 결과를 SARIF 형식으로 생성
          → GitHub Security 탭에 업로드
            → PR에 인라인 코멘트 표시
              → 개발자가 검토 & 수정
                → 재스캔 후 Alert 자동 해소
```

### 이 실습에서 다룬 취약점: SQL Injection

| 항목 | 내용 |
|------|------|
| **CWE** | CWE-89: Improper Neutralization of Special Elements in SQL Command |
| **위험도** | Critical (9.8/10) |
| **공격 방식** | 사용자 입력이 SQL 쿼리에 직접 삽입되어 악의적 SQL 실행 가능 |
| **영향** | 데이터 유출, 데이터 삭제, 인증 우회, 서버 장악 |
| **방어 방법** | 파라미터화된 쿼리, ORM 사용, 입력값 검증 |

### 알림 상태 관리

| 상태 | 의미 | 액션 |
|------|------|------|
| **Open** | 해결되지 않은 활성 알림 | 검토 필요 |
| **Dismissed** | 수동으로 닫음 (사유 기록) | False Positive / Won't Fix / Used in Tests |
| **Fixed** | 코드 수정으로 자동 해소 | 재스캔 시 자동 전환 |

---

## 📎 참고 자료

| 자료 | 링크 |
|------|------|
| **GitHub Skills** | https://github.com/skills |
| **CodeQL 공식 문서** | https://codeql.github.com/docs/ |
| **Code Scanning 문서** | https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/about-code-scanning |
| **CodeQL CLI & VS Code 확장** | https://codeql.github.com/docs/codeql-cli/ |
| **알림 분류(Triage) 가이드** | https://docs.github.com/en/code-security/code-scanning/managing-code-scanning-alerts/triaging-code-scanning-alerts-in-pull-requests |
| **고급 CodeQL 쿼리 기능** | https://docs.github.com/en/code-security/codeql-for-vs-code/using-the-advanced-functionality-of-the-codeql-for-vs-code-extension/creating-a-custom-query |
| **CWE 데이터베이스** | https://cwe.mitre.org/ |
| **SARIF 표준 스펙** | https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html |