# 🛡️ GitHub CodeQL 활용 가이드

> **GitHub CodeQL을 활용하여 코드 보안 취약점을 자동으로 탐지하고, 팀 차원에서 체계적으로 보안 분석을 운영하는 방법**을 정리한 가이드입니다.
> 특정 기술 스택에 종속되지 않으며, **Python, Java, JavaScript, C/C++, C#, Go 등 어떤 스택에서든 동일하게 적용**할 수 있습니다.

---

## 📋 목차

1. [CodeQL이란?](#1--codeql이란)
2. [⭐ 지원 언어 및 프레임워크](#2--지원-언어-및-프레임워크)
3. [⭐ GitHub Actions와 CodeQL 통합](#3--github-actions와-codeql-통합)
4. [CodeQL CLI — 로컬 환경에서 사용하기](#4--codeql-cli--로컬-환경에서-사용하기)
5. [VS Code 확장 — CodeQL for Visual Studio Code](#5--vs-code-확장--codeql-for-visual-studio-code)
6. [QL 쿼리 언어 기초](#6--ql-쿼리-언어-기초)
7. [⭐ 커스텀 쿼리 작성하기](#7--커스텀-쿼리-작성하기)
8. [Data Flow & Taint Tracking](#8--data-flow--taint-tracking)
9. [코드 스캐닝 결과 확인 및 관리 (SARIF)](#9--코드-스캐닝-결과-확인-및-관리-sarif)
10. [팀 차원의 운영 전략](#10--팀-차원의-운영-전략)
11. [스택별 적용 가이드](#11--스택별-적용-가이드)
12. [핵심 요약](#12--핵심-요약)
13. [참고 자료](#13--참고-자료)

---

## 1. 🤖 CodeQL이란?

CodeQL은 **GitHub**에서 제공하는 오픈소스 **정적 분석(Static Analysis)** 도구입니다.
코드베이스를 **데이터베이스**로 변환하고, SQL과 유사한 쿼리 언어(**QL**)로 취약점, 버그, 코드 스멜 등을 탐지합니다.

> **핵심 철학:** "코드를 데이터처럼 다룬다" — 소스 코드의 AST, 데이터 흐름, 제어 흐름을 관계형 데이터로 변환하여 쿼리합니다.

### 핵심 개념: 코드 → 데이터베이스 → 쿼리 → 결과

```
소스 코드
  → CodeQL이 코드를 데이터베이스로 변환 (AST, 데이터 흐름, 제어 흐름 등)
    → QL 쿼리를 실행하여 패턴 매칭
      → 보안 취약점, 버그, 코드 스멜 탐지
        → 결과를 SARIF 형식으로 출력
          → GitHub Security 탭에서 확인
```

### 전통적 보안 분석 vs CodeQL

| 구분 | 전통적 보안 분석 | CodeQL |
|------|----------------|--------|
| 분석 방식 | 규칙 기반 패턴 매칭 | **시맨틱 코드 분석** (의미 기반) |
| 정확도 | False Positive 많음 | **데이터 흐름 추적**으로 높은 정확도 |
| 커스터마이징 | 제한적 | **QL 언어로 자유롭게** 쿼리 작성 |
| CI/CD 통합 | 별도 도구 필요 | **GitHub Actions에 네이티브 통합** |
| 비용 | 유료 도구 다수 | **공개 저장소 무료** |
| 결과 확인 | 별도 대시보드 | **GitHub Security 탭**에서 직접 확인 |

### CodeQL이 탐지할 수 있는 것들

| 카테고리 | 탐지 가능한 취약점 예시 |
|----------|----------------------|
| **인젝션** | SQL Injection, XSS, Command Injection, Path Traversal |
| **인증/인가** | 하드코딩된 비밀번호, 불충분한 인증 검사 |
| **암호화** | 취약한 암호화 알고리즘, 안전하지 않은 랜덤 생성 |
| **데이터 흐름** | 신뢰할 수 없는 입력의 위험한 사용 (Taint Tracking) |
| **코드 품질** | 사용되지 않는 변수, 불필요한 코드, 잠재적 NPE |
| **설정** | 안전하지 않은 설정, 디버그 모드 노출 |

---

## 2. ⭐ 지원 언어 및 프레임워크

### 공식 지원 언어 (2025년 기준)

| 언어 | 지원 상태 | 빌드 필요 여부 | 주요 프레임워크 지원 |
|------|----------|--------------|-------------------|
| **Python** | ✅ 정식 지원 | ❌ 불필요 | Django, Flask, FastAPI |
| **JavaScript / TypeScript** | ✅ 정식 지원 | ❌ 불필요 | React, Express, Angular |
| **Java** | ✅ 정식 지원 | ✅ 필요 | Spring Boot, Maven, Gradle |
| **C / C++** | ✅ 정식 지원 | ✅ 필요 | CMake, Make |
| **C#** | ✅ 정식 지원 | ✅ 필요 | .NET, ASP.NET Core |
| **Go** | ✅ 정식 지원 | ❌ 불필요 | Gin, Echo, net/http |
| **Ruby** | ✅ 정식 지원 | ❌ 불필요 | Rails, Sinatra |
| **Swift** | ✅ 정식 지원 | ✅ 필요 | iOS/macOS 앱 |
| **Kotlin** | 🔶 베타 | ✅ 필요 | Android, Spring |

> 💡 **인터프리터 언어** (Python, JavaScript, Ruby, Go)는 빌드 없이 바로 분석 가능합니다.
> **컴파일 언어** (Java, C/C++, C#, Swift, Kotlin)는 빌드 과정이 필요합니다.

---

## 3. ⭐ GitHub Actions와 CodeQL 통합

### 왜 중요한가?

| 문제 상황 | CodeQL 없이 | CodeQL 있으면 |
|----------|-------------|--------------|
| 보안 취약점 | 수동 코드 리뷰에 의존 | **PR마다 자동으로** 취약점 탐지 |
| 배포 사고 | 취약한 코드가 프로덕션에 배포될 수 있음 | **머지 전 차단** 가능 |
| 규정 준수 | 감사 때마다 별도 스캔 | **연속적 모니터링** 자동화 |
| 팀 부담 | 보안 전문가가 모든 코드 검토 | **자동 탐지 후 개발자가 직접 수정** |

### 기본 워크플로우 설정

```yaml
# .github/workflows/codeql-analysis.yml
name: "CodeQL"

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '30 1 * * 1'   # 매주 월요일 01:30 UTC에 자동 실행

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write

    strategy:
      fail-fast: false
      matrix:
        language: [ 'python' ]
        # 지원 언어: 'javascript', 'java', 'csharp', 'cpp', 'go', 'ruby', 'swift'

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Initialize CodeQL
      uses: github/codeql-action/init@v3
      with:
        languages: ${{ matrix.language }}
        # queries: +security-extended   # 확장 보안 쿼리 팩 추가 가능

    # 컴파일 언어의 경우 빌드 스텝 추가
    # - name: Build
    #   run: make build

    - name: Autobuild
      uses: github/codeql-action/autobuild@v3

    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v3
      with:
        category: "/language:${{ matrix.language }}"
```

### 쿼리 팩(Query Suite) 선택

| 쿼리 팩 | 설명 | 사용 시점 |
|---------|------|----------|
| `security-extended` | 기본 보안 + 확장 보안 쿼리 | **권장** — 가장 많이 사용 |
| `security-and-quality` | 보안 + 코드 품질 쿼리 | 코드 품질까지 관리하고 싶을 때 |
| `default` | 기본 보안 쿼리만 | 빠른 실행이 필요할 때 |
| 커스텀 쿼리 팩 | 팀이 직접 작성한 쿼리 | 팀 고유 규칙 적용 |

```yaml
# queries 설정 예시
- name: Initialize CodeQL
  uses: github/codeql-action/init@v3
  with:
    languages: python
    queries: +security-extended,+security-and-quality
```

### 워크플로우 동작 흐름

```
코드 Push / PR 생성
  → GitHub Actions 트리거
    → CodeQL Init: 언어 감지 & DB 생성 준비
      → Autobuild: 프로젝트 빌드 (컴파일 언어인 경우)
        → Analyze: QL 쿼리 실행 & 취약점 탐지
          → SARIF 결과를 GitHub에 업로드
            → Security 탭 → Code scanning alerts에 표시
              → PR에 인라인 코멘트로 취약점 안내
```

---

## 4. 🖥️ CodeQL CLI — 로컬 환경에서 사용하기

### 설치

```bash
# GitHub CLI를 통한 설치 (권장)
gh extension install github/gh-codeql

# 또는 직접 다운로드
# https://github.com/github/codeql-cli-binaries/releases
```

### 데이터베이스 생성

```bash
# Python 프로젝트 (빌드 불필요)
codeql database create my-python-db --language=python --source-root=.

# Java 프로젝트 (빌드 필요)
codeql database create my-java-db --language=java --command="mvn clean install -DskipTests"

# JavaScript 프로젝트 (빌드 불필요)
codeql database create my-js-db --language=javascript --source-root=.
```

### 쿼리 실행

```bash
# 기본 보안 쿼리 실행
codeql database analyze my-python-db --format=sarifv2.1.0 --output=results.sarif

# 특정 쿼리 팩 실행
codeql database analyze my-python-db codeql/python-queries:Security --format=sarifv2.1.0 --output=results.sarif

# 커스텀 쿼리 실행
codeql database analyze my-python-db ./my-custom-queries/ --format=sarifv2.1.0 --output=results.sarif
```

### 결과를 GitHub에 업로드

```bash
# SARIF 파일을 GitHub Code Scanning에 업로드
codeql github upload-results \
  --sarif=results.sarif \
  --ref=refs/heads/main \
  --commit=$(git rev-parse HEAD) \
  --repository=owner/repo
```

---

## 5. 🧩 VS Code 확장 — CodeQL for Visual Studio Code

### 왜 사용하는가?

| 문제 상황 | CLI만 사용 | VS Code 확장 사용 |
|----------|-----------|------------------|
| 쿼리 작성 | 텍스트 에디터에서 수동 작성 | **자동완성, 구문 강조, 에러 표시** |
| 결과 확인 | SARIF 파일을 별도로 파싱 | **GUI로 코드 위치까지 바로 탐색** |
| 디버깅 | 쿼리 오류 시 CLI 로그 분석 | **인라인 에러 메시지 & Quick Evaluation** |
| 학습 | 문서를 참조하며 시행착오 | **코드 네비게이션 & AST 탐색** |

### 설치 및 설정

1. **VS Code** → Extensions(Ctrl+Shift+X) → `CodeQL` 검색 → 설치
2. **CodeQL CLI**가 로컬에 설치되어 있어야 합니다
3. **데이터베이스 로드**: 좌측 CodeQL 패널 → "Add Database" → 로컬 DB 선택

### 주요 기능

```
쿼리 작성 (.ql 파일)
  → 자동완성 & 타입 체크
    → ▶ Run Query 클릭
      → Results 패널에서 결과 확인
        → 결과 클릭 시 해당 소스 코드로 이동
          → Quick Evaluation: 쿼리 일부만 선택 실행
            → SARIF 내보내기
```

### Quick Evaluation (핵심 기능)

쿼리 작성 중 **일부 표현식만 선택**하고 `CodeQL: Quick Evaluation`을 실행하면,
전체 쿼리를 실행하지 않고도 **중간 결과를 즉시 확인**할 수 있습니다.

```ql
// 이 부분만 선택하고 Quick Evaluation 실행 가능
from Function f
where f.getName() = "eval"
select f
```

---

## 6. 📝 QL 쿼리 언어 기초

### 기본 구조

모든 CodeQL 쿼리는 다음 4가지 요소로 구성됩니다:

```ql
import <라이브러리>       // 1. 분석 대상 언어의 라이브러리 로드

from <타입> <변수>        // 2. 분석할 코드 요소 선언
where <조건>             // 3. 필터링 조건
select <출력>            // 4. 결과 출력
```

### 언어별 import 예시

```ql
import python          // Python 분석
import javascript      // JavaScript/TypeScript 분석
import java            // Java 분석
import csharp          // C# 분석
import go              // Go 분석
import cpp             // C/C++ 분석
```

### 기본 쿼리 예시

#### Python: eval() 함수 호출 탐지

```ql
import python

from Call call, Name name
where
  call.getFunc() = name and
  name.getId() = "eval"
select call, "eval() 함수가 사용되었습니다 — 보안 위험!"
```

#### JavaScript: eval() 함수 호출 탐지

```ql
import javascript

from CallExpr call
where call.getCalleeName() = "eval"
select call, "eval() 함수가 사용되었습니다"
```

#### Java: public 메서드 목록 조회

```ql
import java

from Method m
where m.hasModifier("public")
select m, m.getDeclaringType().getName() + "." + m.getName()
```

### 핵심 문법 요소

| 요소 | 설명 | 예시 |
|------|------|------|
| **Predicate** | 재사용 가능한 논리식 정의 | `predicate isPublic(Method m) { m.hasModifier("public") }` |
| **Class** | 커스텀 타입 정의 | `class PublicMethod extends Method { ... }` |
| **exists** | 존재 여부 확인 | `exists(Variable v \
| v.getName() = "password")` |
| **forall** | 모든 조건 충족 확인 | `forall(Parameter p \
| p.getType().getName() = "String")` |
| **not** | 부정 조건 | `not m.hasModifier("private")` |
| **count** | 개수 집계 | `count(Method m \
| m.getDeclaringType() = c)` |

---

## 7. ⭐ 커스텀 쿼리 작성하기

### 왜 커스텀 쿼리가 필요한가?

| 상황 | 기본 쿼리로는 | 커스텀 쿼리로 |
|------|-------------|-------------|
| 사내 보안 규칙 | 일반적인 취약점만 탐지 | **팀/회사 고유 규칙** 적용 |
| 특정 라이브러리 | 범용 패턴만 인식 | **사용 중인 라이브러리**에 특화된 탐지 |
| 코딩 컨벤션 | 보안만 체크 | **코드 품질 + 컨벤션** 동시 검증 |
| 규제 준수 | 범용 룰셋 | **PCI-DSS, HIPAA 등** 특정 규정 맞춤 |

### 커스텀 쿼리 프로젝트 구조

```
.github/codeql/
├── custom-queries/
│   ├── qlpack.yml                    # 쿼리 팩 설정 파일
│   ├── security/
│   │   ├── HardcodedPassword.ql      # 하드코딩된 비밀번호 탐지
│   │   └── InsecureDeserialization.ql
│   └── quality/
│       ├── UnusedImports.ql          # 미사용 import 탐지
│       └── TooManyParameters.ql      # 파라미터 과다 함수 탐지
```

### qlpack.yml 설정

```yaml
name: my-org/custom-queries
version: 1.0.0
libraryPathDependencies:
  - codeql/python-all     # 분석 대상 언어에 맞게 설정
```

### 커스텀 쿼리 예시: Python 하드코딩된 비밀번호 탐지

```ql
/**
 * @name 하드코딩된 비밀번호
 * @description 소스 코드에 비밀번호가 하드코딩되어 있습니다
 * @kind problem
 * @problem.severity error
 * @security-severity 9.0
 * @precision high
 * @id py/hardcoded-password
 * @tags security
 */

import python

from Assignment assign, StrConst str
where
  assign.getTarget().(Name).getId().regexpMatch("(?i).*(password|passwd|secret|api_key|token).*") and
  assign.getValue() = str and
  str.getText().length() > 0
select assign, "비밀번호/시크릿이 하드코딩되어 있습니다: " + assign.getTarget().(Name).getId()
```

### 커스텀 쿼리 예시: 파라미터가 너무 많은 함수 탐지

```ql
/**
 * @name 파라미터가 과다한 함수
 * @description 함수의 파라미터가 5개를 초과합니다
 * @kind problem
 * @problem.severity warning
 * @id py/too-many-parameters
 * @tags quality
 */

import python

from Function f
where count(f.getAnArg()) > 5
select f, "이 함수는 파라미터가 " + count(f.getAnArg()).toString() + "개입니다. 5개 이하로 줄이세요."
```

### GitHub Actions에서 커스텀 쿼리 사용

```yaml
- name: Initialize CodeQL
  uses: github/codeql-action/init@v3
  with:
    languages: python
    queries: +./.github/codeql/custom-queries   # 커스텀 쿼리 경로 지정
```

---

## 8. 🔍 Data Flow & Taint Tracking

### 개념 비교

| 구분 | Data Flow (데이터 흐름) | Taint Tracking (오염 추적) |
|------|----------------------|--------------------------|
| 정의 | 데이터가 프로그램 내에서 **어떻게 이동**하는지 추적 | 신뢰할 수 없는 입력이 **위험한 지점까지 도달**하는지 추적 |
| 범위 | 값 자체의 흐름만 추적 | 값이 **변형**되어도 오염이 전파되는 것까지 추적 |
| 용도 | 변수 추적, 상수 전파 분석 | **보안 취약점 탐지** (XSS, SQLi 등) |
| 관계 | 기본 분석 방식 | Data Flow의 **확장** (더 넓은 범위 추적) |

### Taint Tracking 핵심 3요소

```
Source (소스)          → 신뢰할 수 없는 데이터가 들어오는 지점
  ↓                      (예: 사용자 입력, HTTP 요청 파라미터)
Sanitizer (정화기)     → 데이터를 안전하게 만드는 지점
  ↓                      (예: 이스케이핑, 유효성 검증)
Sink (싱크)            → 데이터가 사용되는 민감한 지점
                         (예: SQL 쿼리 실행, eval(), 파일 쓰기)
```

### Taint Tracking 쿼리 예시 (Python - SQL Injection)

```ql
/**
 * @name SQL Injection
 * @description 사용자 입력이 SQL 쿼리에 직접 사용됩니다
 * @kind path-problem
 * @problem.severity error
 * @security-severity 9.8
 * @id py/sql-injection-custom
 * @tags security
 */

import python
import semmle.python.dataflow.new.DataFlow
import semmle.python.dataflow.new.TaintTracking

module SqlInjectionConfig implements DataFlow::ConfigSig {
  predicate isSource(DataFlow::Node source) {
    exists(source.asExpr().(Attribute).getName() |
      source.asExpr().(Attribute).getName() = "args" or
      source.asExpr().(Attribute).getName() = "form"
    )
  }

  predicate isSink(DataFlow::Node sink) {
    exists(Call c |
      c.getFunc().(Attribute).getName() = "execute" and
      sink.asExpr() = c.getArg(0)
    )
  }
}

module SqlInjectionFlow = TaintTracking::Global<SqlInjectionConfig>;

from SqlInjectionFlow::PathNode source, SqlInjectionFlow::PathNode sink
where SqlInjectionFlow::flowPath(source, sink)
select sink.getNode(), source, sink,
  "이 SQL 쿼리는 $@에서 오는 사용자 입력을 포함하고 있습니다.",
  source.getNode(), "사용자 입력"
```

---

## 9. 📊 코드 스캐닝 결과 확인 및 관리 (SARIF)

### SARIF란?

**SARIF**(Static Analysis Results Interchange Format)는 정적 분석 결과를 **표준화된 JSON 형식**으로 저장·공유하는 포맷입니다.
CodeQL은 모든 분석 결과를 SARIF로 출력하며, GitHub Security 탭과 연동됩니다.

### SARIF 파일 구조

```json
{
  "version": "2.1.0",
  "$schema": "https://raw.githubusercontent.com/oasis-tcs/sarif-spec/master/Schemata/sarif-schema-2.1.0.json",
  "runs": [
    {
      "tool": {
        "driver": {
          "name": "CodeQL",
          "version": "2.15.0",
          "rules": [
            {
              "id": "py/sql-injection",
              "name": "SqlInjection",
              "shortDescription": { "text": "SQL Injection" },
              "defaultConfiguration": { "level": "error" }
            }
          ]
        }
      },
      "results": [
        {
          "ruleId": "py/sql-injection",
          "level": "error",
          "message": { "text": "SQL 쿼리에 사용자 입력이 직접 사용됩니다." },
          "locations": [{
            "physicalLocation": {
              "artifactLocation": { "uri": "server/app.py" },
              "region": { "startLine": 42, "startColumn": 17 }
            }
          }]
        }
      ]
    }
  ]
}
```

### GitHub에서 결과 확인하기

```
Repository → Security 탭 → Code scanning alerts
  → 각 Alert 클릭
    → 취약점 설명, 코드 위치, 데이터 흐름 경로 확인
      → 상태 관리: Open / Dismissed / Fixed
```

### 결과 관리 워크플로우

| 단계 | 작업 | 담당 |
|------|------|------|
| 1. 탐지 | CodeQL이 PR/Push에서 자동 스캔 | 자동 |
| 2. 알림 | Security 탭에 Alert 등록 + PR 인라인 코멘트 | 자동 |
| 3. 분류 | True Positive / False Positive 분류 | 보안 담당자 |
| 4. 수정 | 취약점 코드 수정 및 PR 제출 | 개발자 |
| 5. 검증 | 수정 후 재스캔으로 Alert 자동 해소 확인 | 자동 |
| 6. 기록 | 처리 이력이 GitHub에 자동 보존 | 자동 |

---

## 10. 🏢 팀 차원의 운영 전략

### Branch Protection 연동

```yaml
# GitHub Repository Settings → Branches → Branch protection rules
# "Require status checks to pass before merging" → CodeQL 체크 추가

# 효과:
# - CodeQL에서 error 수준 Alert이 발견되면 머지 차단
# - 팀 전체가 보안 수준을 유지하도록 강제
```

### 운영 체크리스트

| 항목 | 설명 | 권장 설정 |
|------|------|----------|
| **스캔 트리거** | 언제 CodeQL을 실행할지 | Push + PR + 주간 스케줄 |
| **쿼리 팩** | 어떤 수준의 쿼리를 사용할지 | `security-extended` (최소) |
| **Branch Protection** | 취약점 발견 시 머지 차단 | error 수준 이상 차단 |
| **알림 설정** | 새 Alert 발생 시 알림 | Slack/Email 연동 |
| **담당자 지정** | Alert 처리 책임자 | CODEOWNERS 또는 보안 팀 |
| **SLA 설정** | Alert 처리 기한 | Critical: 24h / High: 1주 / Medium: 1개월 |
| **커스텀 쿼리** | 팀 고유 규칙 추가 | 분기별 검토 및 업데이트 |

### 추천 보안 파이프라인

```
개발자 코드 작성
  → PR 생성
    → CodeQL 자동 스캔 (GitHub Actions)
      → 취약점 발견 시
        ├── error 수준 → 머지 차단 + 개발자에게 인라인 피드백
        ├── warning 수준 → 경고 표시 (머지 가능)
        └── note 수준 → 참고용 표시
      → 취약점 없으면
        → 코드 리뷰 진행
          → 머지
            → 주간 스캔으로 지속 모니터링
```

---

## 11. 🔧 스택별 적용 가이드

### Python (Flask/Django/FastAPI)

```yaml
# .github/workflows/codeql.yml
strategy:
  matrix:
    language: [ 'python' ]
# Autobuild가 자동으로 처리 — 별도 빌드 스텝 불필요
```

> **주요 탐지 항목**: SQL Injection, SSRF, XSS (Jinja2 템플릿), Path Traversal, eval()/exec() 사용, pickle 역직렬화

### Java (Spring Boot/Maven/Gradle)

```yaml
strategy:
  matrix:
    language: [ 'java' ]
steps:
  - uses: github/codeql-action/init@v3
  # Maven 빌드
  - run: mvn clean install -DskipTests
  # 또는 Gradle 빌드
  # - run: ./gradlew build -x test
  - uses: github/codeql-action/analyze@v3
```

> **주요 탐지 항목**: SQL Injection, XSS, LDAP Injection, XML Injection, 안전하지 않은 역직렬화, Spring Security 설정 미흡

### JavaScript/TypeScript (React/Express/Angular)

```yaml
strategy:
  matrix:
    language: [ 'javascript' ]
# Autobuild가 자동으로 처리 — 별도 빌드 스텝 불필요
```

> **주요 탐지 항목**: XSS (DOM-based, Reflected), Prototype Pollution, SQL Injection (ORM 미사용), ReDoS, eval() 사용

### C# (.NET/ASP.NET Core)

```yaml
strategy:
  matrix:
    language: [ 'csharp' ]
steps:
  - uses: github/codeql-action/init@v3
  - run: dotnet build --no-restore
  - uses: github/codeql-action/analyze@v3
```

> **주요 탐지 항목**: SQL Injection, XSS, CSRF, XML External Entity (XXE), LDAP Injection, 안전하지 않은 역직렬화

### Go (Gin/Echo)

```yaml
strategy:
  matrix:
    language: [ 'go' ]
# Autobuild가 자동으로 처리
```

> **주요 탐지 항목**: SQL Injection, Command Injection, Path Traversal, 불안전한 TLS 설정

---

## 12. 📌 핵심 요약

| 항목 | 핵심 내용 |
|------|----------|
| **CodeQL이란?** | 코드를 DB로 변환하고 QL 쿼리로 취약점을 탐지하는 **시맨틱 정적 분석** 도구 |
| **동작 원리** | 소스 코드 → 데이터베이스 → 쿼리 실행 → SARIF 결과 → GitHub Security 탭 |
| **GitHub Actions 통합** | `.github/workflows/codeql-analysis.yml` 추가만으로 자동화 |
| **CLI** | `codeql database create` → `codeql database analyze` → 결과 확인 |
| **VS Code 확장** | 쿼리 작성 · 자동완성 · Quick Evaluation · 결과 탐색 |
| **QL 쿼리 언어** | `import` → `from` → `where` → `select` 구조의 선언형 언어 |
| **커스텀 쿼리** | 팀 고유의 보안 규칙, 코딩 컨벤션을 쿼리로 정의하여 자동 탐지 |
| **Taint Tracking** | Source → Sanitizer → Sink 흐름을 추적하여 인젝션 계열 취약점 탐지 |
| **SARIF** | 정적 분석 결과의 표준 JSON 포맷 — GitHub, VS Code 등과 연동 |
| **팀 운영** | Branch Protection 연동, SLA 설정, 주간 스케줄 스캔으로 **지속적 보안 모니터링** |

---

## 13. 📚 참고 자료

| 자료 | 링크 |
|------|------|
| **CodeQL 공식 문서** | https://docs.github.com/en/code-security/code-scanning/introduction-to-code-scanning |
| **CodeQL 쿼리 작성 가이드** | https://codeql.github.com/docs/writing-codeql-queries/ |
| **CodeQL 쿼리 저장소** | https://github.com/github/codeql |
| **CodeQL CLI 다운로드** | https://github.com/github/codeql-cli-binaries/releases |
| **VS Code CodeQL 확장** | https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-codeql |
| **SARIF 스펙** | https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html |
| **GitHub Skills - Introduction to CodeQL** | https://github.com/skills/introduction-to-codeql |
| **CodeQL 데이터 흐름 분석** | https://codeql.github.com/docs/writing-codeql-queries/about-data-flow-analysis/ |
