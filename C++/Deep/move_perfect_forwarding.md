```cpp
template <typename T>
void my_swap(T &a, T &b) {
  T tmp(a);
  a = b;
  b = tmp;
}
```

move sementic이 없다면, swap 함수도 이렇게 작성합니다.
이 경우 temp 객체를 생성 + 복사하고, 복사를 2번 수행해야 합니다.
이는 메모리도, 시간도 모두 비효율적입니다

T를 String으로 생각하면, string의 내부 char*만 swap해주면 됩니다. (길이 등은 없다고 생각하겠습니다.)
이를 위해 `my_swap`을 수정할 수 있지만 템플릿 함수이기 때문에 구현이 어렵습니다.

```cpp
template <typename T>
void my_swap(T &a, T &b) {
  T tmp(std::move(a));
  a = std::move(b);
  b = std::move(tmp);
}
```

move 함수는 단순히 타입을 우측값으로 변경하는 함수입니다.
위와 같이 변경하고, T(T&& t)와 같은 이동생성자를 구현하면 효율적인 swap이 구현됩니다.

```cpp
class A {
 public:
  A() { std::cout << "ctor\n"; }
  A(const A& a) { std::cout << "copy ctor\n"; }
  A(A&& a) { std::cout << "move ctor\n"; }
};

class B {
 public:
  A a_;
};

int main() {
  A a;

  std::cout << "create B-- \n";
  B b(/* ?? */);
}
```

위와 같이, A를 멤버 변수로 가지는 B 클래스가 존재할 경우를 생각해봅시다.
B 객체를 생성하며, 기존의 a를 이동하고 싶습니다.

```cpp
B(const A& a) : a_(a) {} // A의 복사 생성자 호출(const A& 이므로)
B(const A& a) : a_(std::move(a)) {} // a를 const A&&로 변경하지만, 해당하는 A의 생성자가 없음
B(A&& a) : a_(a) {} // 매개변수를 std::move(a)로 전달, 그러나 이름이 있는 우측값은 좌측값이다.
```

위 경우 모두 복사 생성자가 호출된다.

결과적으로

```cpp
class A {
 public:
  A() { std::cout << "ctor\n"; }
  A(const A& a) { std::cout << "copy ctor\n"; }
  A(A&& a) { std::cout << "move ctor\n"; }
};

class B {
 public:
  B(A&& a) : a_(std::move(a)) {}

  A a_;
};

int main() {
  A a;
  std::cout << "create B-- \n";
  B b(std::move(a));
}
```

위와 같이, move를 통해 rvalue로 형변환 한 파라미터를 전달하고, b 생성자와 a 생성자는 모두 rvalue 매개변수가 호출되도록, 한번 더 move한 파라미터를  a 생성자에 전달

```cpp
vec.push_back(A(1, 2, 3));
vec.emplace_back(1, 2, 3);  // 위와 동일한 작업을 수행한다.
```

emplace_back은 내부에서 생성자를 이용해 원소를 만들고, 추가한다.
push_back은 실제 객체를 생성해서 만들어주고, 복사 생성자로 원소를 추가하게 된다.
(그러나 컴파일러 최적화로 emplace_back 과 동일한 어셈블리로 동작하게 됐다)

그래도 아예 내부에 원소를 생성하는 emplace_back이 아주 근소하게 효율적이다?!

### 완벽한 전달

```cpp
template <typename T>
void wrapper(T u) {
  g(u);
}

int main() {
  A a;
  const A ca;

  std::cout << "원본 --------" << std::endl;
  g(a);
  g(ca);
  g(A());

  std::cout << "Wrapper -----" << std::endl;
  wrapper(a);
  wrapper(ca);
  wrapper(A());
}
```

위와 같은 경우, `wrapper(A())`에서 우리가 기대하는건 이동 생성자를 이용한 전달이다.

```cpp
template <typename T>
void wrapper(T& u) {
  g(u);
}
```

이렇게 할 경우, `warpper(A())` 는 컴파일 오류가 발생합니다. 레퍼런스로 변경할 수 없기 때문이죠.

```cpp
template <typename T>
void wrapper(T& u) {
  std::cout << "T& 로 추론됨" << std::endl;
  g(u);
}

template <typename T>
void wrapper(const T& u) {
  std::cout << "const T& 로 추론됨" << std::endl;
  g(u);
}
```

이렇게 둘로 쪼갤 수 있지만, rvalue 값으로 전달은 안되고 또 파라미터 개수에 따라 모든 조합을 구현해야합니다.

```cpp
template <typename T>
void wrapper(T&& u) {
  g(std::forward<T>(u));
}
```

보편적 레퍼런스 라는 C++11의 기능을 이용하면 해결할 수 있습니다.

이는 일반적인 함수 시그니쳐와 다르게, 컴파일러가 추론하여 타입을 결정합니다.
즉, T&& 타입만 받는 함수가 아닌, T, T&도 가능합니다.

여기서 `std::forward` 는 무엇일까요?
`wrapper(A())` 의 경우, 타입은 A로 추론됩니다.

```cpp
template <class S>
S&& forward(typename std::remove_reference<S>::type& a) noexcept {
  return static_cast<S&&>(a);
}
```

forward 함수는 위와 같고 결국 좌측값은 좌측값으로, 우측값은 우측값으로 변경해줍니다.