가능합니다. C++에서는 이런 요구사항을 **접근 제어(Access Control)** 로 해결하는 방법이 여러 가지 있습니다.

질문하신 내용을 예를 들면,

```cpp
class B
{
public:
    void Func1();   // 누구나 호출 가능

    void Func2();   // A를 통해서만 호출 가능

    void Func3();   // A를 통해서만 호출 가능
};

class A
{
public:
    ...
};
```

처럼 `Func2()`, `Func3()`는 오직 `A`만 호출하도록 만들고 싶은 것입니다.

가장 많이 사용하는 방법들을 장단점과 함께 설명드리겠습니다.

---

# 방법 1. friend 사용 (가장 일반적, 권장)

가장 간단한 방법입니다.

```cpp
class A;

class B
{
public:
    void Func1();

private:
    void Func2();
    void Func3();

    friend class A;
};

class A
{
public:
    void DoSomething(B& b)
    {
        b.Func2();
        b.Func3();
    }
};
```

사용

```cpp
B b;

b.Func1();      // 가능
b.Func2();      // 컴파일 에러

A a;
a.DoSomething(b);   // 가능
```

### 장점

* 가장 직관적
* 컴파일 타임에 강제됨
* 구현이 간단

### 단점

* A가 B의 private 구현에 접근 가능
* Friend 관계가 많아지면 결합도가 증가

대부분의 프로젝트에서는 이 방법을 가장 많이 사용합니다.

---

# 방법 2. Proxy(Handle) 객체 사용 (대규모 프로젝트 권장)

B는 일반 인터페이스만 공개합니다.

```cpp
class B
{
public:
    void Func1();

private:
    void Func2();

    friend class A;
};
```

A가 래퍼를 제공합니다.

```cpp
class A
{
public:
    void Func2(B& b)
    {
        b.Func2();
    }
};
```

사용자는

```cpp
B b;

b.Func1();

A a;
a.Func2(b);
```

만 사용할 수 있습니다.

장점

* B의 API가 깔끔
* 권한 관리가 쉬움

---

# 방법 3. PassKey Idiom (대규모 라이브러리에서 많이 사용, 가장 권장)

Google, LLVM 등에서도 사용하는 패턴입니다.

```cpp
class A;

class B
{
public:
    void Func1();

    class Key
    {
    private:
        Key() {}
        friend class A;
    };

    void Func2(Key);
};
```

A만 Key를 생성할 수 있습니다.

```cpp
class A
{
public:
    void Call(B& b)
    {
        b.Func2(B::Key{});
    }
};
```

사용자는

```cpp
B b;

b.Func2(B::Key{});   // 컴파일 에러
```

### 장점

* friend로 private 전체를 공개하지 않음
* 특정 함수만 제한 가능
* 매우 강력한 컴파일 타임 제약

### 단점

* 코드가 조금 복잡

개인적으로 가장 추천하는 방법입니다.

---

# 방법 4. Interface 분리

Public Interface

```cpp
class IB
{
public:
    virtual void Func1() = 0;
};
```

Internal Interface

```cpp
class IBInternal : public IB
{
public:
    virtual void Func2() = 0;
};
```

A는

```cpp
IBInternal*
```

을 알고 있고,

일반 사용자는

```cpp
IB*
```

만 사용합니다.

장점

* 의존성이 명확
* 테스트하기 쉬움

단점

* 클래스 수 증가

---

# 방법 5. Nested Class

```cpp
class B
{
public:
    class Internal;
};
```

Internal API를 별도로 관리하는 방법입니다.

Boost나 Qt에서 종종 사용합니다.

---

# 방법 6. Token 객체 사용 (PassKey의 확장)

```cpp
class AccessToken
{
private:
    AccessToken() {}
    friend class A;
};
```

```cpp
class B
{
public:
    void Func2(AccessToken);
};
```

A만

```cpp
AccessToken{}
```

을 생성할 수 있습니다.

---

# 어떤 방법을 추천할까?

프로젝트 규모에 따라 추천이 달라집니다.

| 방법                  | 구현 난이도 | 캡슐화   | 권장도   |
| ------------------- | ------ | ----- | ----- |
| friend class        | ★      | 보통    | ⭐⭐⭐⭐☆ |
| private + friend 함수 | ★      | 좋음    | ⭐⭐⭐⭐⭐ |
| PassKey Idiom       | ★★★    | 매우 좋음 | ⭐⭐⭐⭐⭐ |
| Interface 분리        | ★★★★   | 매우 좋음 | ⭐⭐⭐⭐☆ |
| Token 객체            | ★★★    | 매우 좋음 | ⭐⭐⭐⭐⭐ |
| Proxy               | ★★     | 좋음    | ⭐⭐⭐⭐☆ |

## DPDK/RAN 같은 대규모 C++ 프로젝트라면

사용자께서 이전에 DPDK와 RAN SW 구조를 자주 질문하신 점을 고려하면, **`friend class`보다는 `PassKey Idiom`을 가장 추천**합니다.

이유는 다음과 같습니다.

* `friend class A`는 `A`가 `B`의 **모든 private 멤버**에 접근할 수 있습니다.
* 반면 PassKey는 **특정 함수에 대해서만 접근 권한을 부여**할 수 있어 캡슐화가 훨씬 강합니다.
* API를 읽는 사람도 "이 함수는 특별한 권한이 있어야 호출 가능하다"는 의도를 바로 이해할 수 있습니다.
* 향후 `A` 외에 `Controller`, `Manager` 등 다른 승인된 클래스가 추가되어도 권한 모델을 확장하기 쉽습니다.

대규모 프레임워크나 라이브러리에서도 이러한 **권한 토큰(PassKey/Token) 패턴**이 캡슐화를 유지하면서 접근을 제한하는 대표적인 방법으로 사용됩니다.
