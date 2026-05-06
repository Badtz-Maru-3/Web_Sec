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
| 03 | 실습 환경 | XAMPP, PHP, MySQL 기반 로컬 실습 환경 |

---

## 폴더 구조

```text
SQLi/
├── README.md
├── 01_concept.md
├── 02_vulnerable_code.md
└── 03_setup/
