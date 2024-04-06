---
title: JVM Class Loader
date: 2023-12-20 10:00 +0900
aliases: null
tags:
  - JVM
  - 클래스 로딩
  - Class Loader
  - Delegation Model
  - Loading Linking Initialization
  - Permanent Generation
  - Metaspace
  - static final 클래스 로딩
  - 컴파일 타임 상수
  - 런타임 상수
image: /assets/img/thumbnails/Java.png
categories:
  - Skill
  - Java
---

## Class Loader

자바 소스코드가 실행되는 과정은 다음과 같다

1. 자바 컴파일러가 `소스 코드(.java) -> 바이트 코드(.class)`로 컴파일 
   - 생성된 바이트 코드는 완전한 기계어가 아니고 JVM이 이해할 수 있는 레벨의 코드
2. 컴파일된 바이트 코드를 `필요한 시점`에 Class Loader가 JVM Runtime Data Area에 동적 로드
   - 클래스 로딩은 <span style="color:red">Lazy Loading</span> 방식으로 실제 해당 클래스가 사용될 때까지 로딩을 미룬다
3. 로드된 바이트 코드를 인터프리터 & JIT 컴파일러에 의해서 기계어로 번역한 후 실행

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img1.png" alt="img"/>
</div>

> ClassLoader의 주요 역할은 <span style="color:red">컴파일된 바이트 코드를 필요한 시점에 동적으로 JVM에 로드</span>하고 이 과정에서 여러 검증 및 초기화를 진행한다
{: .prompt-info }

### Class Loader 원칙

#### 1. Delegation Principal

> 클래스 로딩을 진행하는 경우 <span style="color:red">저 본인의 Parent Class Loader에게 로딩을 위임</span>한다

- Bootstrap Class Loader (최상위)
- Extension/Platform Class Loader
- Application/System Class Loader (최하위)

#### 2. Visibility Principal

> Child Class Loader는 Parent Class Loader가 로드한 클래스에 대한 가시성을 보장받을수 있다<br>
> 하지만 역방향의 가시성(Parent → Child)는 보장할 수 없다

- 간단한 예시로써 Bootstrap Class Loader가 로드한 Object Class는 그 하위 Class Loader에서 사용할 수 있다

#### 3. Uniqueness Principal

> 한번 로딩된 클래스는 중복 로드될 수 없고 유일해야 한다

- 동적으로 Class가 로드될 때 반드시 유일하게 로드되어야 하고 JVM은 이를 보장한다 (synchronized)

### Class Loader 종류

#### 1. Bootstrap Class Loader

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img2.png" alt="img"/>
</div>

> 최상위 클래스 로더이자 JVM-Built-In 클래스 로더

- JVM 시작 시 가장 먼저 구동되는 클래스 로더
- 자바 자체의 클래스 로더 + 최소한의 자바 클래스(java.lang.Object, Class, ClassLoader, ...)를 로드하는 역할을 수행한다
- `Native Code`로 구현되어 있고 getClassLoader()로 조회하면 null이 나온다
  - 최상위 클래스 로더이므로 Parent가 존재하지 않고 그에 따라서 null이 도출된다

#### 2. Extension/Platform Class Loader

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img3.png" alt="img"/>
</div>

> Bootstrap Class Loader를 부모로 갖는 클래스 로더

- JDK 확장 라이브러리들을 로드하는 역할

#### 3. Application/System Class Loader

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img4.png" alt="img"/>
</div>

> Extension/Platform Class Loader를 부모로 갖는 클래스 로더

- ClassPath/Jar에 속한 모든 클래스들을 로드하는 역할

### Class Loading 과정

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img5.png" alt="img"/>
</div>

- Class Loader가 특정 클래스에 대한 로드를 시도할때는 위와 같은 `Binary Name`으로 가져오게 된다

<br>

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img6.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img7.png" alt="img"/>
</div>

#### Step 1

> 로드하려는 클래스가 이미 로드되었는지 확인

Class Loader가 지켜야 하는 3가지 원칙중에서 <span style="color:red">Uniqueness Principal</span>을 지킨다고 볼 수 있다

1. synchronized를 통해서 Critical Section에 대한 Mutual Exclusion 보장 
   - Unique하게 가져오기 위해서는 로드하는 영역을 Critical Section이라고 생각하고 해당 Section에는 순차적으로 진입해야 한다
   - 이를 위해서 Java의 synchronized를 활용해서 Mutual Exclusion을 보장하고 있다
2. 그 후 findLoadedClass를 통해서 이미 해당 클래스가 로드되었는지 확인

Class Loader가 특정 클래스를 로딩하는 시점에 이미 로딩되었는지 확인하고 만약 로드되었다면 c != null이므로 로드하는 로직으로 진입하지 않는다

#### Step 2

> 로드되지 않은 클래스라면 로딩을 진행하게 된다<br>
> &rarr; Parent Class Loader가 존재한다면 로딩을 위임한다

Class Loader가 지켜야 하는 3가지 원칙중에서 <span style="color:red">Delegation Principal</span>을 지킨다고 볼 수 있다

- `parent != null` : Parent가 존재한다는 의미이고 그에 따라서 상위 클래스 로더에게 클래스 로딩을 위임한다
- `parent == null` : 로드를 시도하려는 Class Loader가 Bootstrap Class Loader라는 의미

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img8.png" alt="img"/>
</div>

#### Step 3

> 자신의 Parent Class Loader에 의해서 로드되지 않았다면 자신이 해당 클래스를 찾아본다

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img9.png" alt="img"/>
</div>

- Binary Name에 해당하는 클래스를 찾아본다
- xxxClassLoader별로 반드시 Override해야하는 메소드로써 클래스를 찾지 못하면 ClassNotFoundException을 던진다

> Class Loader에 의해서 성공적으로 특정 클래스가 로딩되면 <span style="color:red">해당 클래스 타입의 Class 객체가 Heap Area에 생성</span>된다<br>
> &rarr; `Class<T>`
{: .prompt-tip }

<br>

## Class Loader가 수행하는 3가지 작업

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img10.png" alt="img"/>
</div>

### 1. Loading

> 실행 도중 특정 클래스를 참조할 때 해당 클래스의 `.class 파일`을 찾아서 동적으로 JVM Runtime Data Area에 로드한다

1. 해당 클래스의 바이트 코드를 검색한다
2. 찾은 바이트 코드를 `JVM 메모리 영역인 Method Area`에 로드한다
   - 클래스 이름, 부모 클래스 이름, 메소드 & 변수 정보, ..등을 관리
3. 로드된 클래스에 대한 `Class<T> 객체`를 Heap Area에 생성한다
   - `Class<T> 객체`를 활용해서 런타임에 해당 클래스 정보에 접근할 수 있다

### 2. Linking

> 로드된 클래스를 `검증`하고 `Symbolic Reference -> Memory Reference`로 참조를 변환한다

#### 1) Verify

로드된 클래스가 JVM 스펙상에서 유효한지 검증한다

- Symbol Table이 일관되고 올바른 형식인지
- 접근 지정자에 따른 접근 범위를 고려해서 참조하고 있는지
- 매개변수 & 자료형이 올바른지
- ...

#### 2) Prepare

모든 클래스 변수(static)에 대한 메모리 공간을 할당하고 `기본값`으로 초기화시킨다

- int, short, byte, long = 0
- float, double = 0.0
- boolean = false
- 참조형 = null

#### 3) Resolve

클래스, 인터페이스, 필드, 메소드에 대한 `모든 Symbolic Reference -> Memory Reference`로 변환시킨다

- 실제로 메모리 레벨에서 어떤 주소를 가리키는지 결정

```java
public class Main {
    public static void main(String[] args) {
        Sub sub = new Sub();
        sub.hello();
    }
}

class Sub {
    void hello() {}
}
```

```text
$ javap -c Main.class
Compiled from "Main.java"
public class com.sjiwon.javaplayground.Main {
  public com.sjiwon.javaplayground.Main();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #7                  // class com/sjiwon/javaplayground/Sub
       3: dup
       4: invokespecial #9                  // Method com/sjiwon/javaplayground/Sub."<init>":()V
       7: astore_1
       8: aload_1
       9: invokevirtual #10                 // Method com/sjiwon/javaplayground/Sub.hello:()V
      12: return
}
```

- #1: `Object 클래스의 생성자(<init>)`에 대한 심볼릭 참조
- #7: `com/sjiwon/javaplayground/Sub 클래스`에 대한 심볼릭 참조.
- #9: `com/sjiwon/javaplayground/Sub 클래스의 생성자(<init>)`에 대한 심볼릭 참조
- #10: `com/sjiwon/javaplayground/Sub 클래스의 hello 메소드`에 대한 심볼릭 참조

> `javap -c Main.class`를 통해서 바이트 코드 수준의 정보를 볼 수 있다

### 3. Initialization

> Linking(2) 과정에서 기본값으로 초기화시킨 클래스 변수(static)에 대한 `실제 값`을 할당<br>
> - 클래스가 처음으로 참조될 때 한 번만 실행

1. 정적 필드 초기화
2. static 블록 초기화

```java
class A {
    public static B b = new B();

    static {
        System.out.println("A static 블록");
    }

    A() {
        System.out.println("A 생성자");
    }
}

class B {
    B() {
        System.out.println("B 생성자");
    }
}

public class Main {
    public static void main(String[] args) {
        new A();
    }
}
```

```text
B 생성자
A static 블록
A 생성자
```

<br>

## Class Loading 테스트

```java
public class Hello {
    public static String a = "> Hello Class -> static String a";
    public static final String b = "> Hello Class -> static final String b";

    public Hello() {
        System.out.println("> Hello Class -> Constructor");
    }

    class Inner {
        public static String c = "> Hello Class's Inner Class -> static String c";
        public static final String d = "> Hello Class's Inner Class -> static final String d";

        public Inner() {
            System.out.println("> Hello Class's Inner Class -> Constructor");
        }
    }

    static class StaticInner {
        public static String e = "> Hello Class's Static Inner Class -> static String e";
        public static final String f = "> Hello Class's Static Inner Class -> static final String f";

        public StaticInner() {
            System.out.println("> Hello Class's Static Inner Class -> Constructor");
        }
    }
}

public class Main {
    public static void main(String[] args) {
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img11.png" alt="img"/>
</div>

> `-verbose:class` 옵션을 통해서 클래스 로딩 과정을 확인할 수 있다

### 1. Only Main

```java
public class Main {
    public static void main(String[] args) {
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img12.png" alt="img"/>
</div>

### 2. Hello()

#### 1) 생성자

```java
public class Main {
    public static void main(String[] args) {
        new Hello();
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img13.png" alt="img"/>
</div>

#### 2) static 필드

```java
// Case 1
public class Main {
    public static void main(String[] args) {
        System.out.println(new Hello().a);
    }
}

// Case 2
public class Main {
    public static void main(String[] args) {
        System.out.println(Hello.a);
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img14.png" alt="img"/>
</div>

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img15.png" alt="img"/>
</div>

#### 3) static final 필드

```java
// Case 1
public class Main {
    public static void main(String[] args) {
        System.out.println(new Hello().b);
    }
}

// Case 2
public class Main {
    public static void main(String[] args) {
        System.out.println(Hello.b);
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img16.png" alt="img"/>
</div>

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img17.png" alt="img"/>
</div>

- 위의 결과와는 다르게 `Hello.b (static final)`의 경우 Hello 클래스가 로딩되지 않았다
  - 이에 대한 자세한 설명은 포스팅 하단에 <span style="color:red">JVM상에서 static 필드의 위치</span>에서 설명할 예정

### 3. Hello() + Inner()

#### 1) 생성자

```java
public class Main {
    public static void main(String[] args) {
        new Hello().new Inner();
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img18.png" alt="img"/>
</div>

##### ***✏️ Inner Class의 메모리 누수 문제***

Inner Class에 대한 인스턴스를 생성하기 위해서는 다음과 같은 과정을 거친다

1. Outer Class 초기화
2. Inner Class 초기화

<span style="color:red">Inner Class는 Outer Class에 대한 암묵적 참조</span>를 가지고 있고 그에 따라서 Inner Class가 로딩되기 위해서는 Outer Class도 반드시 로딩되어야 한다<br>
이러한 암묵적 외부 참조로 인해 메모리 누수 문제가 발생할 수 있다

- Outer Class는 필요하지 않는 상황에서 Inner Class가 필요하다면 필연적으로 Outer Class에 대한 로딩이 이루어져야 한다
- 여기서 Outer Class는 Inner Class에 의해서 참조를 가지게 되고 그에 따라서 GC의 대상이 될 수 없다
- 이러한 클래스가 쌓일수록 점차 많은 메모리 누수가 발생할 여지가 존재한다

> 따라서 특정 클래스에 대해서 Inner Class를 선언해야 할 일이 있다면 단순한 Inner Class가 아니라 `Static Inner Class`로 선언해야 위와 같은 메모리 누수 문제를 피할 수 있다

#### 2) static 필드

```java
// Case 1
public class Main {
    public static void main(String[] args) {
        System.out.println(new Hello().new Inner().c);
    }
}

// Case 2
public class Main {
    public static void main(String[] args) {
        System.out.println(Hello.Inner.c);
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img19.png" alt="img"/>
</div>

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img20.png" alt="img"/>
</div>

#### 3) static final 필드

```java
// Case 1
public class Main {
    public static void main(String[] args) {
        System.out.println(new Hello().new Inner().d);
    }
}

// Case 2
public class Main {
    public static void main(String[] args) {
        System.out.println(Hello.Inner.d);
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img21.png" alt="img"/>
</div>

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img22.png" alt="img"/>
</div>

### 4. Hello() + StaticInner()

#### 1) 생성자

```java
public class Main {
    public static void main(String[] args) {
        new Hello.StaticInner();
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img23.png" alt="img"/>
</div>

> Inner Class와는 다르게 Static Inner Class는 Outer Class에 대한 암묵적 외부 참조를 가지고 있지 않고 그에 따라서 Outer Class인 Hello는 로드되지 않음을 확인할 수 있다

#### 2) static 필드

```java
// Case 1
public class Main {
    public static void main(String[] args) {
        System.out.println(new Hello.StaticInner().e);
    }
}

// Case 2
public class Main {
    public static void main(String[] args) {
        System.out.println(Hello.StaticInner.e);
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img24.png" alt="img"/>
</div>

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img25.png" alt="img"/>
</div>

#### 3) static final 필드

```java
// Case 1
public class Main {
    public static void main(String[] args) {
        System.out.println(new Hello.StaticInner().f);
    }
}

// Case 2
public class Main {
    public static void main(String[] args) {
        System.out.println(Hello.StaticInner.f);
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img26.png" alt="img"/>
</div>

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img27.png" alt="img"/>
</div>

<br>

## JVM상에서 static 필드의 위치 (Hotspot JVM)

```java
public class StaticFields {
    public int a = 4;
    public final int b = 4;
    public static int c = 4;
    public static final int d = 4;
    public static final String e = "Hello";
    public static final List<String> f = List.of("Hello", "World");

    public StaticFields() {
    }

    public void methodA() {
    }

    public final void methodB() {
    }

    public static void methodC() {
    }
}
```

StaticFields 클래스의 정적 멤버들은 JVM 내부 어디에서 관리될까?

- `public static int c`
- `public static final int d`
- `public static final String e`
- `public static final List<String> f`
- `public static void methodC()`

```text
// 바이트 코드
$ javap -c StaticFields.class
Compiled from "StaticFields.java"
public class StaticFields {
  public int a;

  public final int b;

  public static int c;

  public static final int d;

  public static final java.lang.String e;

  public static final java.util.List<java.lang.String> f;

  public StaticFields();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: iconst_4
       6: putfield      #7                  // Field a:I
       9: aload_0
      10: iconst_4
      11: putfield      #13                 // Field b:I
      14: return

  public void methodA();
    Code:
       0: return

  public final void methodB();
    Code:
       0: return

  public static void methodC();
    Code:
       0: return

  static {};
    Code:
       0: iconst_4
       1: putstatic     #16                 // Field c:I
       4: ldc           #19                 // String Hello
       6: ldc           #21                 // String World
       8: invokestatic  #23                 // InterfaceMethod java/util/List.of:(Ljava/lang/Object;Ljava/lang/Object;)Ljava/util/List;
      11: putstatic     #29                 // Field f:Ljava/util/List;
      14: return
}
```

### Permanent Generation vs Metaspace

- [Java Memory: PermGen vs MetaSpace - LinkedIn](https://www.linkedin.com/pulse/java-memory-permgen-vs-metaspace-incus-data-pty-ltd-drxbe/)
- [Permgen is part of heap or not? - Stack Overflow](https://stackoverflow.com/questions/41358895/permgen-is-part-of-heap-or-not)

#### 1. Permanent Generation (PermGen)

PermGen은 `자바 8부터 없어진 영역`이고 JVM Non-Heap Area에 위치한다<br>
PermGen은 아래와 같은 데이터를 관리한다

- 클래스 메타데이터 (클래스 이름, 변수, 메소드, ...)
- 클래스 변수 (static)
- 상수 (static final)
- String Constant Pool
- JIT 컴파일러에 의해 최적화된 코드
- ...

<br>
그러면 PermGen은 왜 없어졌을까? -> [Remove the Permanent Generation](https://openjdk.org/jeps/122)

##### 1) 고정된 크기

PermGen 영역의 크기는 `JVM이 시작되는 시점`에 설정되고 변경되지 않는 값이다<br>
따라서 이러한 고정된 크기로 인해 `동적으로 많은 클래스를 로딩`하게 되면 OOM(OutOfMemory)가 자주 발생하게 되었다

- `java.lang.OutOfMemoryError: PermGen space`

##### 2) 메모리 관리

PermGen은 위에서 말했듯이 (클래스 메타데이터, 변수, 상수, ..)등의 데이터들을 관리하고 대부분 애플리케이션의 생명주기 동안 지속적으로 참조되기 때문에, 가비지 컬렉션(GC)에 의해 회수될 가능성이 상대적으로 적다<br>
따라서 대부분의 데이터가 계속해서 사용되는 PermGen 영역의 특성이 GC의 효율성을 저하시키고, 결국 메모리 부족 문제로 이어질 수 있다

- 결국 `고정된 크기`라는 부분이 크리티컬한 문제

#### 2. Metaspace

> PermGen의 고정된 크기 + 메모리 문제를 해결하기 위해 등장한 영역
{: .prompt-info }

Metaspace는 `OS의 Native Memory`를 사용해서 메타 데이터 정보들을 관리하는 영역이다<br>

- Native Memory는 OS가 관리하는 영역이고 `OS Level에서 메모리를 알아서 조절`하기 때문에 이전에 발생하던 PermGen OOM 문제에서 자유로워졌다

#### 데이터 관리 영역의 변화

> Class metadata, interned Strings and class static variables will be moved from the permanent generation to <span style="color:red">either the Java heap or native memory.</span><br>
> The proposed implementation will allocate <span style="color:red">class meta-data in native memory and move interned Strings and class statics to the Java heap.</span>

- 클래스 메타데이터 = Metaspace
- String Constant Pool = Heap Area
- static 멤버 = ~~_Heap Area_~~

> static 멤버의 경우 관리되는 영역에 대해서 아래 2가지 의견이 존재하는 것을 확인하였습니다.<br>
> 1. Heap Area
> 2. Primitive + Method = Metaspace / Reference = 실제 값은 Heap Area, 그에 대한 참조는 Metaspace
> 
> 이에 대한 정확한 정보를 아신다면 댓글 남겨주시면 감사하겠습니다
{: .prompt-warning }

### 컴파일 타임 상수 (static final)

위에서 살펴본 클래스 로딩 테스트에서 `static final 필드`에 대해서는 클래스 로딩없이 참조됨을 확인할 수 있었다

> 이 이유는 바로 참조한 타입의 상수가 <span style="color:red">컴파일 타임 상수</span>로 인식되었기 때문이다
{: .prompt-tip }

<br>

[JVM Docs - Binary Compatibility](https://docs.oracle.com/javase/specs/jls/se14/html/jls-13.html#jls-13.1)

> A reference to a field that is a constant variable (§4.12.4) <span style="color:red">must be resolved at compile time</span> to the value V denoted by the constant variable's initializer.

- 상수 필드는 컴파일 시점에 모든것이 결정된다

<br>
그러면 무슨 필드든 final만 붙이면 상수로 인식되고 컴파일 시점에 모든것이 결정될까? -> 그건 또 아니다

[JVM Docs - final variables](https://docs.oracle.com/javase/specs/jls/se14/html/jls-4.html#jls-4.12.4)

> A constant variable is a <span style="color:red">final variable of primitive type or type String</span> that is initialized with a constant expression.

- final로 선언된 `Primitive Type + String`만 컴파일 타임 상수로 인식된다

```java
public class Main {
    public static final int MAXIMUM = 10;
    public static final String USERNAME = "sjiwon";
    public static final String input = System.console().readLine();
    public static final double random = Math.random();
    public static final List<String> list = List.of("a", "b");

    public static void main(String[] args) {
        System.out.println(MAXIMUM);
        System.out.println(USERNAME);
        System.out.println(input);
        System.out.println(random);
        System.out.println(list);
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-12-20-JVM%20Class%20Loader/img28.png" alt="img"/>
</div>

- 컴파일 타임 상수에 해당되는 필드만 실제 값으로 치환되면서 최적화가 이루어짐을 확인할 수 있다

> 위의 테스트에서도 static final 필드에 대해서 클래스가 로딩되지 않았던 이유 역시 컴파일 타임 상수이고 컴파일 시점에 최적화를 통해서 실제 값으로 대체했기 때문이다
