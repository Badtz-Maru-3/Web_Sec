# 01. SQL Injection 개념

## 1. SQL Injection이란?

SQL Injection(SQLi)은 웹 애플리케이션이 데이터베이스에 전달하는 SQL Query에 사용자의 입력값이 안전하지 않은 방식으로 포함될 때 발생할 수 있는 취약점이다.

사용자 입력값이 단순한 데이터가 아니라 SQL 문법의 일부로 해석되면, 원래 개발자가 의도한 Query와 다른 동작이 수행될 수 있다.

대표적으로 다음과 같은 문제가 발생할 수 있다.

- 권한이 없는 데이터 조회
- 인증 우회
- 데이터 수정 또는 삭제
- 데이터베이스 구조 확인
- 서버 또는 백엔드 인프라에 대한 추가 공격 가능성 증가

SQL Injection은 CWE 기준으로 `CWE-89: Improper Neutralization of Special Elements used in an SQL Command`에 해당한다.

---

## 2. SQL Injection이 발생하는 핵심 원리

SQL Injection의 핵심은 **데이터와 명령어의 경계가 무너지는 것**이다.

웹 애플리케이션은 보통 사용자의 입력값을 받아 SQL Query를 만든다.

예를 들어 게시글, 상품, 사용자 정보를 조회하는 기능은 내부적으로 다음과 같은 Query를 사용할 수 있다.

```sql
SELECT * FROM items WHERE id = 1;
