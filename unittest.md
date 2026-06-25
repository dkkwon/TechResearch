# C++ Unit Test Generation Guideline for LLM (Codex/Copilot/Claude)

## 목적

LLM은 C++ 모듈에 대한 Unit Test를 작성할 때 단순 코드 커버리지 확보가 아닌, 요구사항 검증 및 회귀 버그 방지를 목표로 한다.

생성되는 테스트는 GoogleTest(GTest) 기반으로 작성한다.

---

# 기본 원칙

## 1. 구현이 아닌 요구사항을 테스트하라

테스트는 내부 구현 방식이 변경되어도 실패하지 않아야 한다.

잘못된 예

* private 함수 호출 순서 검증
* 내부 자료구조(vector/map) 사용 여부 검증
* 내부 변수 값 직접 검증

올바른 예

* 입력에 대한 출력 검증
* 상태 변화 검증
* 인터페이스 계약 검증

---

## 2. 하나의 테스트는 하나의 동작만 검증한다

각 TEST()는 하나의 시나리오만 검증한다.

좋은 예

TEST(RouteTable, AddRoute_ValidRoute_ReturnSuccess)

TEST(RouteTable, AddRoute_DuplicateRoute_ReturnFail)

나쁜 예

TEST(RouteTable, AddDeleteUpdateRoute)

---

## 3. 테스트 이름 규칙

다음 형식을 사용한다.

TEST(ClassName, Function_Scenario_ExpectedResult)

예시

TEST(UeManager, RegisterUe_ValidUe_ReturnSuccess)

TEST(UeManager, RegisterUe_DuplicateUe_ReturnFail)

TEST(UeManager, RegisterUe_MaxUeReached_ReturnFail)

---

# 테스트 생성 우선순위

각 Public API에 대해 아래 순서로 테스트를 생성한다.

## Priority 1 : Happy Path

정상 입력

정상 상태

정상 결과

반드시 생성

---

## Priority 2 : Boundary Condition

다음 경계 조건을 검토한다.

* 0
* 1
* MAX-1
* MAX
* MAX+1
* Empty Container
* Single Element
* Full Capacity

모든 경계 조건을 분석하고 필요한 테스트를 생성한다.

---

## Priority 3 : Error Path

다음 조건을 검토한다.

* nullptr
* Invalid enum
* Invalid state
* Invalid length
* Empty string
* Corrupted message
* Unsupported value

예외 처리 또는 오류 반환값을 검증한다.

---

## Priority 4 : State Transition

상태 머신을 가지는 클래스는 상태 전이를 검증한다.

예시

INIT
-> CONNECTED
-> ACTIVE
-> RELEASED

각 상태 변화 후 상태값을 검증한다.

---

# Assertion 규칙

## 반드시 의미 있는 Assertion 작성

금지

Func();

ASSERT_TRUE(true);

EXPECT_TRUE(true);

허용

EXPECT_EQ(expected, actual);

EXPECT_FALSE(result);

EXPECT_THROW(...);

---

## Assertion 수 제한

하나의 테스트는 가능하면 하나의 핵심 결과를 검증한다.

Assertion이 많아질 경우 테스트를 분리한다.

---

# Mock 사용 규칙

Mock은 외부 의존성에만 사용한다.

대상

* Database
* Socket
* Timer
* File System
* Message Queue

Mock을 사용한 경우 다음을 검증한다.

* 호출 여부
* 호출 횟수
* 입력 파라미터

Mock이 5개 이상 필요하면 클래스 책임 분리를 검토한다.

---

# C++ 특화 테스트

다음 항목을 검토한다.

## Constructor

초기 상태 검증

## Destructor

자원 해제 검증

## RAII

스코프 종료 시 정리 동작 검증

## Copy Constructor

깊은 복사 여부 검증

## Move Constructor

이동 후 객체 상태 검증

## Exception Safety

예외 발생 시 리소스 누수 여부 검증

---

# Thread Safety

멀티스레드 코드인 경우

* 동시 읽기
* 동시 쓰기
* Read/Write 혼합

시나리오를 생성한다.

Race Condition 가능성을 검증한다.

---

# Networking / Telecom Module

통신 모듈인 경우 다음을 반드시 검토한다.

## Message Validation

정상 메시지

비정상 메시지

길이 오류

버전 오류

필수 필드 누락

---

## Sequence Handling

정상 순서

중복 수신

순서 역전

메시지 손실

---

## Timer Expiry

Timeout 발생

Retry 발생

Retry 초과

---

# Coverage 목표

Coverage 확보를 위해 테스트를 생성하지 않는다.

다음을 우선한다.

1. 요구사항 검증
2. 경계값 검증
3. 오류 처리 검증
4. 상태 전이 검증

Coverage는 결과 지표일 뿐 목표가 아니다.

---

# 생성 결과 형식

각 테스트 앞에 테스트 목적을 주석으로 작성한다.

예시

// Verify registration succeeds with valid UE information.
TEST(UeManager, RegisterUe_ValidUe_ReturnSuccess)
{
...
}

---

# 생성 금지 항목

다음 테스트는 생성하지 않는다.

* 의미 없는 ASSERT
* 구현 세부사항 검증
* private 함수 직접 테스트
* 테스트 간 의존성
* Sleep 기반 테스트
* Random 값에 의존하는 테스트
* 환경 의존 테스트
* 네트워크 실제 연결 테스트
* 실제 파일 시스템 접근 테스트

모든 테스트는 독립적이고 반복 가능해야 한다.
