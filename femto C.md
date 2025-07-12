# 1. femto C 기본 목적

C 수준의 유연한 저수준 제어에 Rust의 소유권 개념을 부분적으로 결합시킨 언어. **단순함**과 **통일성**을 지향합니다.

---

# 2. 타입 시스템

## 2.1. 원시 타입

* **고정 크기 정수**: `u8`, `u16`, `u32`, `u64`, `i8`, `i16`, `i32`, `i64`
* **플랫폼 의존 크기 정수**: `uint`, `int`
* **실수**: `f32`, `f64`
* **불리언**: `bool` (`true`, `false` 리터럴 사용)

## 2.2. 포인터

* **소유권 포인터**: `box T` (항상 유효한 힙 인스턴스 `T`를 참조)
* **참조 포인터**: `&T` (유효한 인스턴스 `T`에 대한 참조, 읽기쓰기 모두 가능)
* **원시 포인터**: `*T` (C 스타일의 포인터)
* **원형 포인터**: `*void`
* **함수 포인터**: `(func (...): T)`

## 2.3. 타입 전치사

* **Nullable**: `?box T`, `?&T`는 `void`일 수 있습니다. (`NULL` 대신 `void` 사용)
* **배열**: `[]T`는 `*T`와 동일하며 여러 `T`가 연속된 메모리 구조를 나타냅니다.

## 2.4. 캐스팅

* `(id as T)`: 명시적 타입 캐스팅

---

# 3. 소유권 시스템

* `box T`는 항상 유효한 힙 인스턴스 `T`를 참조합니다.
* **Scope**: 중괄호 `{}`로 표현하며, scope가 끝나면 자동으로 소멸자가 호출됩니다.
* **소유권 이동**: 변수 대입과 함수 인자 전달 시 소유권이 이동하며, 유효성을 컴파일러가 검증합니다.
* `&T`는 유효한 인스턴스 `T`에 대한 참조이며 읽기쓰기 모두 가능합니다.
* **안전성**: 배열 인덱싱과 멀티스레딩 안전성은 C와 동일하게 동작합니다. 별도의 컨테이너 객체나 Lock을 사용해야 합니다.

---

# 4. 원시 포인터 기능

* `malloc`, `free`, `sizeof`를 사용하여 직접 `*T`를 다룹니다.
* **원시 포인터와 관리 포인터 변환 규칙**:
    * `box T`, `&T` -> `*T`: **명시적인 캐스팅** 필요. 소유권은 여전히 `T`에 존재하며, `free(*T)`는 오류입니다.
    * `*T` -> `&T`: 원시 포인터 값에 대한 **일시적 빌림**. 빌리는 중 `free` 사용은 오류입니다.
    * `*T` -> `box T`: **명시적인 소유권 이동**. 이후 `*T`에 대한 `free` 사용은 오류입니다.

## 4.1. Unsafe Function

* **함수명**: `_`로 끝납니다 (예: `id_`).
* **unsafe function 내부에만 가능한 동작**:
    * `malloc`, `free`
    * `*T` 산술 연산
    * `*T` 역참조
    * `*T`에서 `box T`, `&T`로의 캐스팅

---

# 5. 변수 선언

* `var id: type;`
* `var id: type = expr;`
* `var id = expr;` (**타입 추론**)
* `const id: type = expr;`
* **초기화**: **상수**, `box T`, `&T`는 선언과 함께 초기화되어야 합니다.
* `box T`에 새 값을 할당하면 기존 값은 자동으로 `delete`됩니다.
* **상수 취급**: 리터럴로 초기화된 상수는 컴파일 타임 리터럴처럼 취급됩니다.

---

# 6. 배열 선언

* 배열의 이름도 타입은 포인터와 완전히 동일합니다.
* `var id: [cons expr]T;` (고정 크기 배열)
* `var id: [constexpr][constexpr]T = {...};` (다차원 배열 초기화)

---

# 7. 동적 메모리 선언

* `new`는 힙 메모리를 확보하고 항상 `box` 타입만을 반환합니다.
* `var id: box T = new T;` (단일 객체)
* `var id: box T = new T(...);` (생성자를 통한 초기화)
* `var id: box []T = new [expr]T;` (동적 배열, `var id: box *T = new [expr]T;`과 동일)

---

# 8. 배열과 동적 메모리 인덱싱

* **인덱싱**: `id[expr]` (항목에 대한 직접 접근)
* **내부 동작**: 내부적으로는 `*(id + expr)`과 동일합니다.

---

# 9. 연산자

* **산술**: `+`, `-`, `*`, `/`, `%`
* **관계**: `<`, `<=`, `>`, `>=`, `==`, `!=`
* **논리**: `&&`, `||`, `!`
* **비트**: `&`, `|`, `~`, `^`, `<<`, `>>`
* **메모리**:
    * **역참조**: `*` (`*T`, `&T`에 적용)
    * **참조**: `&` (`T`, `box T`, `&T`에 적용)
* **캐스팅**: `as`

---

# 10. 제어문

* `if (bool) scope`
* `if (bool) scope else scope`
* `if (bool) scope else if (bool) scope`
* `switch (integer expr) { case integer constexpr: ... case a, b: ... default: ... }`
* `while (bool) scope`
* `for (st; bool expr; st) scope`
* **흐름 제어**: `break`, `continue`

---

# 11. 함수

* **접근 제어**: `id`가 언더바 (`_`)로 시작하는 함수만 `private`합니다.
* `func id() {}` (매개변수 없음, 반환 값 없음)
* `func id(a: type, b: type): type {}` (매개변수, 반환 값 있음)
* `func id(parms): T scope`

---

# 12. 구조체

* **접근 제어**: `id`가 언더바 (`_`)로 시작하는 구조체 혹은 필드만 `private`합니다.
* **필드/메서드 접근**: `.` 사용. 구조체 포인터는 자동 역참조됩니다.
* `struct T { id: type; id: type; }`
* **특수 메서드 (생성자, 소멸자, 복사)**:
    * `func (self: box T) __new(...): box T {...}`
    * `func (self: box T) __delete(...) {...}`
    * `func (self: &T) __copy(...): box T {...}`
* **구조체 메서드**:
    * `func (self: *T) id(...) {...}`
    * `func (self: &T) id(...) {...}`

---

# 13. 예외 처리

* **일반 예외**: 오류 코드 반환으로 처리합니다.
* **심각한 오류**: `try`, `catch`, `raise`로 처리합니다. (`box T`만 던지기 가능)
* `try` 블록 안에서 명시적으로 `raise`되는 모든 예외의 타입을 `catch`에서 받지 못하면 컴파일 에러가 발생합니다.
* `try { raise id; } catch (var e: box T) {...}`
* **표준 예외 객체**: `err.e {code: int; msg: box []u8}`는 표준 라이브러리에서 제공 예정입니다.
* **기본 예외**: `raise` 없는 기본 예외 (예: `new` 메모리 확보 실패)는 `box int` 타입의 에러를 발생시킵니다.

---

# 14. 템플릿

* 함수나 구조체 뒤에 `<T>`를 붙여 동일한 알고리즘을 가진 코드를 여러 타입에 대해 생성합니다.
* 내부적으로는 코드를 복사하여 각 타입마다 적용시켜 구현됩니다.
* `struct id<T> {...}`
* `func id<T, U>(...): T {...}`

---

# 15. 모듈

* **파일명**: 모듈 `name`이 됩니다.
* `include name;`
* `include name as alias;`
* **표준 라이브러리**: `std`
* **C 표준 라이브러리**: `c.xxx`

---

# 16. 컴파일러 직접 지시

* **주석**: `//`, `/* */`
* `#order` 사용:
    * `#raw_asm {...}`
    * `#raw_llvm {...}`
    * `#raw_c {...}`
    * `#define {...}`
    * `#enum {...}`

---

# 17. 키워드 목록 (총 40개)

## 17.1. 타입 (17개)

* `u8`, `u16`, `u32`, `u64`, `i8`, `i16`, `i32`, `i64`
* `uint`, `int`
* `f32`, `f64`
* `bool`, `true`, `false`, `void`, `box`

## 17.2. 변수, 메모리 키워드 (6개)

* `var`, `const`, `new`, `malloc`, `free`, `sizeof`

## 17.3. 제어 키워드 (9개)

* `if`, `else`, `switch`, `case`, `default`, `while`, `for`, `break`, `continue`

## 17.4. 함수, 구조체 키워드 (3개)

* `func`, `return`, `struct`

## 17.5. 예외 처리, 모듈 키워드 (5개)

* `try`, `catch`, `raise`, `include`, `as`

---

# 18. 예시 코드

## 18.1. `box T`를 이용한 힙 메모리 관리 (안전한 방식)

```rust
// 간단한 구조체 정의
struct Point {
    x: i32;
    y: i32;
}

func (self: box Point) __new(x: i32, y: i32): box Point {
    self.x = x;
    self.y = y;
    return self;
}

func main(): void {
    // new를 사용하여 힙에 Point 인스턴스 생성 및 소유권 획득
    var p1: box Point = new Point(10, 20); 
    
    // p1에 접근하여 값 변경 (자동 역참조)
    p1.x = 30;

    // p1의 소유권을 p2로 이동
    var p2: box Point = p1; 

    // 이제 p1은 유효하지 않으므로 접근 불가 (컴파일 에러 예상)
    // p1.y = 40; 

    // p2를 통해 값 접근
    std.print_int(p2.x); // 출력: 30

    // 스코프를 벗어나면 p2가 가리키는 메모리는 자동으로 해제됩니다.

    test_delete();
    // __delete 메서드가 있다면 호출됩니다.
}

// 사용자 정의 소멸자 예시
struct MyData {
    value: box []u8; // 내부적으로 힙 메모리를 소유
}

func (self: box MyData) __new(v: box []u8): box MyData {
    self.value = v;
    return self;
}

func (self: box MyData) __delete(): void {
    std.print_str("MyData 인스턴스가 소멸됩니다.");
    // value 필드의 box []u8도 자동으로 소멸됩니다.
}

func test_delete(): void {
    var data_instance: box MyData = new MyData(new [10]u8);
    // data_instance는 스코프를 벗어나면 __delete가 호출됩니다.
}
```

## 18.2. `&T`를 이용한 참조 (빌림)

```rust
struct Wallet {
    balance: i32;
}

func (self: box Wallet) __new(b: int): box Wallet {
    self.balance = b;
    return self;
}

// Wallet의 잔액을 증가시키는 함수 (참조를 빌려옴)
func add_balance(w: &Wallet, amount: i32): void {
    // &Wallet은 읽기쓰기 모두 가능하므로 w.balance 직접 수정
    w.balance = w.balance + amount; 
}

func main(): void {
    var my_wallet: box Wallet = new Wallet(100);

    // add_balance 함수에 my_wallet의 참조를 전달
    // 소유권은 이동하지 않으므로 my_wallet은 여전히 유효
    add_balance(&my_wallet, 50); 
    
    std.print_int(my_wallet.balance); // 출력: 150

    // 함수 호출 후에도 my_wallet은 여전히 사용 가능
    my_wallet.balance = 200;
}
```

## 18.3. `*T` 원시 포인터와 `unsafe` 활용 (저수준 제어)

```rust
// _allocate_int는 안전하지 않은 함수로, raw malloc을 사용합니다.
func _allocate_int_unsafe_(): *int {
    // malloc으로 int 크기의 메모리 할당
    var raw_ptr: *int = (malloc(sizeof(int)) as *int); 
    if (raw_ptr == (void as *int)) { // void를 *int로 캐스팅하여 NULL 비교
        std.print_str("메모리 할당 실패!");
        return (void as *int);
    }
    *(raw_ptr) = 123; // 역참조하여 값 쓰기
    return raw_ptr;
}

// _deallocate_int는 안전하지 않은 함수로, raw free를 사용합니다.
func _deallocate_int_unsafe_(ptr: *int): void {
    if (ptr != (void as *int)) {
        free(ptr as *void); // free는 *void를 받으므로 캐스팅
    }
}

func main_(): void {
    var raw_int_ptr: *int;

    // _allocate_int_unsafe_는 unsafe 함수이므로 직접 호출 가능
    raw_int_ptr = _allocate_int_unsafe_(); 

    if (raw_int_ptr != (void as *int)) {
        // raw_int_ptr의 값을 출력
        std.print_int(*(raw_int_ptr)); // 출력: 123

        // raw_int_ptr을 &int로 일시적으로 빌림
        var borrowed_ref: &int = (raw_int_ptr as &int);
        std.print_int(borrowed_ref); // 출력: 123 (자동 역참조)

        // 원시 포인터를 box int로 소유권 이동 (unsafe 필요)
        // 이 예시에서는 raw_int_ptr이 함수 스코프 내에서 malloc되었으므로 가능하지만, 
        // 실제로는 이처럼 소유권을 이동하는 경우가 흔치 않을 수 있습니다.
        // 다음 라인은 위험하며, 원래 메모리 할당 방식에 따라 사용에 주의해야 합니다.
        var owned_int_box: box int = std.from_int(raw_int_ptr);
        std.print_int(owned_int_box); // 출력: 123 (자동 역참조)

        // 이제 owned_int_box가 소유권을 가졌으므로, raw_int_ptr에 대한 free는 오류
        // _deallocate_int_unsafe_(raw_int_ptr); // 컴파일 오류 혹은 런타임 오류 유발 가능

        // owned_int_box는 스코프를 벗어나면 자동으로 해제됩니다.
    }

    // 만약 소유권을 box로 이동하지 않았다면, 여기서 free 호출
    // _deallocate_int_unsafe_(raw_int_ptr); 
}
```

## 18.4. 배열과 포인터 연산 (`unsafe` 예시)

```rust
func _process_array_unsafe_(data_ptr: *u8, length: uint): void {
    for (var i: uint = 0; i < length; i = i + 1) {
        // 포인터 산술 연산과 역참조
        *(data_ptr + i) = *(data_ptr + i) + 1; // 각 요소에 1 더하기
    }
}

func main(): void {
    var arr_size: uint = 5;
    // box []u8을 사용하여 동적으로 u8 배열 생성
    var my_array: box []u8 = new [arr_size]u8;

    // 배열 초기화
    my_array[0] = 1;
    my_array[1] = 2;
    my_array[2] = 3;
    my_array[3] = 4;
    my_array[4] = 5;

    std.print_str("원본 배열:");
    for (var i: uint = 0; i < arr_size; i = i + 1) {
        std.print_int(my_array[i]);
    }

    // my_array (box []u8)를 *u8로 캐스팅하여 unsafe 함수에 전달
    // 소유권은 여전히 my_array가 가집니다.
    _process_array_unsafe_((my_array as *u8), arr_size);

    std.print_str("처리 후 배열:");
    for (var i: uint = 0; i < arr_size; i = i + 1) {
        std.print_int(my_array[i]); // 출력: 2, 3, 4, 5, 6
    }
    // my_array는 스코프를 벗어나면 자동으로 해제됩니다.
}
```
