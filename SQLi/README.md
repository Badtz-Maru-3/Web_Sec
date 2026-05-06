# SQL Injection

SQL Injection(SQLi)에 대해 공부한 내용을 정리하는 폴더입니다.

이 폴더에서는 SQL Injection의 기본 개념, 발생 원리, 주요 유형, 실습 환경, 취약한 코드 분석, 공격 흐름, 대응 방법을 정리합니다.

---

## 학습 목표

- SQL Injection이 발생하는 원리 이해
- 사용자 입력값이 SQL Query에 영향을 주는 방식 분석
- Error-based SQL Injection 이해
- Union-based SQL Injection 이해
- Blind SQL Injection의 기본 개념 이해
- 취약한 PHP/MySQL 코드 분석
- 안전한 쿼리 작성 방식 이해
- 로컬 실습 환경에서 공격 흐름 재현

---

## 정리할 내용

| 번호 | 주제 | 설명 |
|---|---|---|
| 01 | SQL Injection 개념 | SQLi의 정의와 발생 조건 |
| 02 | 취약한 코드 구조 | 사용자 입력값이 쿼리에 직접 삽입되는 코드 분석 |
| 03 | Error-based SQLi | DB 에러 메시지를 이용한 정보 확인 방식 |
| 04 | Union-based SQLi | `UNION SELECT`를 이용한 결과 병합 방식 |
| 05 | Blind SQLi | 참/거짓 응답 차이를 이용한 추론 방식 |
| 06 | 실습 환경 | XAMPP, PHP, MySQL 기반 로컬 실습 환경 |
| 07 | 대응 방법 | Prepared Statement, 입력 검증, 에러 메시지 제한 등 |

---

## 폴더 구조

```text
SQLi/
├── README.md
├── 01_concept.md
├── 02_vulnerable_code.md
├── 03_error_based_sqli.md
├── 04_union_based_sqli.md
├── 05_blind_sqli.md
├── 06_lab_setup.md
└── 07_mitigation.md
