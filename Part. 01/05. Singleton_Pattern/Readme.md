# 싱글턴 패턴

싱글턴 패턴은 어떤 특정 컴포넌트의 인스턴스가 애플리케이션 전체에서 **단 하나만** 존재해야 하는 상황을 처리하기 위해 고안되었다.
예를 들어 메모리에 DB를 로딩하고 읽기 전용 인터페이스를 제공하는 경우는 싱글턴 패턴을 활용하기 딱 좋은 상황이다. 왜냐하면 동일한 데이터를 여러번 로딩하여 메모리를 낭비할 필요가 없기 때문이다.

</br>

## 5.1 전역 객체로서의 싱글턴

위에서 제시한 문제에 대한 손쉬운 접근 방법은 객체를 단 한 번만 인스턴스화하도록 **개발자 간에 약속**하는 것이다.

```C++
struct Database
{
    // 이 객체를 두 개 이상 인스턴스화하지 마시오.
    Database() {}
};
```

위의 방법은 동료 개발자가 주석을 무시하고 약속을 지키지 않는 경우, 생성자가 호출되어 버릴 수도 있다.
복사 생성자, 복사 대입 연산자에 의한 것일 수도 있고, make_unique()의 호출, 또는 제어 역전(IoC) 컨테이너의 사용에 따른 것일 수도 있다.
이를 극복할 수 있는 뻔한 아이디어는 **static 전역 객체**를 두는 것이다.

```C++
    static Database database{};
```

static 전역 객체의 문제점은 **각각의 컴파일 단위 바이너리들에서 초기화 순서가 정의되어 있지 않다**는 점이다.
static 전역 객체가 여러 개 사용된다면 어느 한 모듈에서 전역 객체를 참조할 때 그 전역 객체가 참조하는 또 다른 전역 객체는 아직 초기화된 상태가 아닐 수도 있다.
그리고 **사용자가 전역 객체가 있다는 사실을 어떻게 알 수 있느냐 하는 문제**도 있다. ::만 입력하여 전역 변수의 존재를 알 수 있긴 하지만 자연스러운 방법은 아니다.
사용자가 조금 더 알기 쉽도록, 필요한 객체를 리턴하는 전역 함수를 제공하는 방법이 있다.

```C++
Database& GetDatabase()
{
    static Database database;
    return database;
}
```

이 함수를 호출하면 DB에 접근할 수 있는 참조를 얻을 수 있다. 위 함수의 스레드 안정성이 C++11 이상 버전에서만 보증된다는 점을 알아야 한다.
static 객체를 초기화하는 코드 앞뒤로 컴파일러가 락을 삽입하여 초기화 와중에 동시에 다른 스레드에서 접근하는 것을 방지해 주는지 확인할 필요가 있다.
이 방식은 문제를 일으킬 가능성이 다분하다.

</br>

## 전통적인 구현

위의 구현은 매우 중요한 문제를 해결하지 못한다. **객체가 추가로 생성되는 것을 방지하는 장치가 없다**는 것이다. 
static 전역 변수로 관리한다고 해서 인스턴스의 추가 생성을 막을 수는 없다.
이에 대한 쉬운 대책은 **생성자 내부에 static 카운터 변수를 두고 값이 증가될 경우 예외를 발생**시키는 것이다.

```C++
struct Database
{
    Database()
    {
        static int instance_count{0};
        if(++instance_count > 1)
            throw std::exception("Cannot make >1 database!");
    }
}
```

위 방법은 인스턴스가 한 개보다 더 많이 생성되는 것을 방지할 수는 있지만 사용자와의 소통 측면에서는 완전한 실패다. 
사용자 관점에서 **인터페이스상으로는 Database의 생성자가 단 한 번만 호출되어야 한다는 것을 알 수 없다**.
Database를 사용자가 명시적으로 생성하는 것을 막는 방법은 **생성자를 private으로 선언하고 인스턴스를 리턴받기 위한 멤버 함수를 만드는 것**이다.

```C++
struct Database
{
protected:
    Database() { /* ... */ }

public:
    static Database& get()
    {
        // C++11 이상 버전에서는 스레드 세이프
        static Database database;
        return database;
    }

    Database(Database const&) = delete;
    Database(Database&&) = delete;
    Database& operator=(Database const&) = delete;
    Database& operator=(Database&&) = delete;
};
```

C++11 이전에는 복제/이동 생성자/연산자를 삭제하는 방법 대신 private으로 선언하는 방법이 있었다.
위의 코드와 같은 수작업이 싫다면 **boost::noncopyable** 클래스를 상속받아 구현하자. 이는 **이동 생성자/연산자를 제외하고 모두 숨길 수 있다**.
Database가 **다른 static 또는 전역 객체에 종속적이고, 소멸자에서 그러한 객체를 참조하고 있다면 위험**하다.
static 객체, 전역 객체의 소멸 순서는 정해져 있지 않기 때문에 **접근하는 시점에서 이미 소멸되어 있을 수도 있다**.

get() 함수에서 힙 메모리 할당으로 객체를 생성하게 할 수 있다. 이렇게 함으로써 전체 객체가 아니라 포인터만 static으로 존재할 수 있다.

```C++
static Database& get()
{
    static Database* database = new Database();
    return *database;
};
```

위 구현은 Database가 프로그램이 종료될 때까지 살아있어 소멸자의 호출이 필요 없는 것으로 가정한다. 따라서 참조 대신 포인터를 사용해 public 생성자가 호출될 일이 없게 한다.
또한 **static 변수의 초기화는 전체 런타임 중에 단 한 번만 호출되므로 메모리 누수를 일으키지 않는다**.

</br>

## 싱글턴의 문제

Database가 도시의 이름과 그 인구수의 목록을 담고 있다고 하자. 그러면 다음과 같이 도시의 이름에서 인구수를 얻는 인터페이스를 가질 수 있다.

```C++
class Database
{
public:
    virtual int get_population(const std::string& name) = 0;
};
```

Database는 도시의 인구수를 알려주는 멤버 함수 하나를 가진다. 이제 Database를 상속받는 싱글턴 구현 클래스 SingletonDatabase가 있다고 하자.

```C++
class SingletonDatabase: public Database
{
private:
    std::map<std::string, int> m_capitals;

    SingletonDatabase() { /* 데이터베이스에서 데이터 읽어 들이기 */ }

public:
    SingletonDatabase(SingletonDatabase const&) = delete;
    void operator=(SingletonDatabase const&) = delete;

    static SingletonDatabase& get()
    {
        static SingletonDatabase db;
        return db;
    }

    auto get_population(const std::string& name) override
    {
        return m_capitals[name];
    }
};
```

위 코드와 같은 **싱글턴 구현의 문제는 다른 싱글턴 컴포넌트에서 또 다른 싱글턴을 사용할 때** 나타난다.
예를 들어 위 코드를 기반으로 하여, 서로 다른 여러 도시의 인구수의 합을 계산하는 싱글턴 컴포넌트를 만들었다고 하자.

```C++
struct SingletonRecordFinder
{
    auto total_population(std::vector<std::string> names)
    {
        auto result = 0;

        for(const auto& name : names)
            result += SingletonDatabase::get().get_population(name);

        return result;
    };
}
```

여기에서 문제는 SingletonRecordFinder가 SingletonDatabase에 밀접하게 의존한다는 데서 발생한다.
SingletonRecordFinder에 대한 단위 테스트를 한다고 생각해보자.

```C++
TEST(RecordFinderTests, SingletonTotalPopulationTest)
{
    SingletonRecordFinder rf;
    auto names = std::vector<std::string> { "Seoul", "Mexico City" };
    auto tp = rf.total_population(names);
    EXPECT_EQ(17500000 + 17400000, tp);
}
```

그런데 실제 데이터는 언제든 바뀔 수 있다. 그때마다 테스트 코드를 수정하는 것은 매우 소모적이다.
실제 데이터 대신 더미 데이터를 이용하여 테스트할 수 있도록 더미 Database 컴포넌트를 사용하고 싶다면 어떻게 할까?
이러한 설계에서는 더미 컴포넌트를 사용하는 것이 불가능하다. 이러한 **유연성 부족**이 싱글턴의 전형적인 단점이다.

한 가지 방법은 **싱글턴 Database에 대한 명시적 의존을 제거하는 것**이다.
꼭 싱글턴 객체가 아니어도 Database 인터페이스를 구현한 객체만 있으면 RecordFinder를 구동할 수 있다.
따라서 데이터를 어디서 얻을지 지정할 수 있게 하는 ConfigurableRecordFinder를 만든다.

```C++
struct ConfigurableRecordFinder
{
    Database& db;

    explicit ConfigurableRecordFinder(Database& db)
        : db{db}
    {}

    auto total_population(std::vector<std::string> names)
    {
        auto result = 0;

        for(const auto& name : names)
            result += db.get_population(name);

        return result;
    }
};
```

이제 **명시적으로 싱글턴에 의존하는 대신 참조 변수 db를 사용**한다.
이렇게 함으로써 ConfigurableRecordFinder를 테스트할 때는 아래와 같은 더미 데이터를 지정할 수 있다.

```C++
class DummyDatabase: Database
{
private:
    std::map<std::string, int> m_capitals;

public:
    DummyDatabase()
    {
        m_capitals["alpha"] = 1;
        m_capitals["beta"] = 2;
        m_capitals["gamma"] = 3;
    }

    auto get_population(const std::string& name) override
    {
        return m_capitals[name];
    }
};
```

이제 DummyDatabase를 사용하도록 단위 테스트 코드를 수정할 수 있다.

```C++
Test(RecordFinderTests, DummyTotalPopulationTest)
{
    DummyDatbase db {};
    ConfigurableRecordFinder rf { db };
    EXPECT_EQ(4, rf.total_population(std::vector<std::string>{"alpha", "beta"}));
}
```

이제 실제 데이터베이스를 사용하지 않기 때문에 실 데이터가 변경될 때마다 단위 테스트 코드를 수정해야 할 일이 없어진다.

</br>

## 싱글턴과 제어 역전(IoC)

어떤 **컴포넌트를 명시적으로 싱글턴으로 만드는 것은 과도하게 깊은 의존성을 유발**한다.
이 때문에 싱글턴 클래스를 다시 일반 클래스로 만들 때 많은 수정 비용이 든다.
또 다른 방법은 **클래스의 생성 소멸 시점을 직접적으로 강제하는 대신 IoC 컨테이너에 간접적으로 위임**하는 것이다.

의존성 주입 프레임워크 Boost.DI를 이용하면 아래와 같이 싱글턴 컴포넌트를 IoC 관례에 맞추어 정의할 수 있다.

```C++
auto injector = di::make_injector(di::bind<IFoo>.to<Foo>.in(di::singleton), /* 기타 설정 작업들 */);
```

위 코드에서는 **타입 이름에 I를 붙여 인터페이스 목적의 타입임을 나타내고 있다**.
여기서 **di::bind**가 의미하는 바는, **IFoo 타입 변수를 멤버로 가지는 컴포넌트가 생성될 때마다 IFoo 타입 멤버 변수를 Foo의 싱글턴 인스턴스로 초기화한다**는 것을 뜻한다.

이렇게 **의존성 주입 컨테이너를 활용하는 방식만이 바람직한 싱글턴 패턴의 구현 방법**이다.
이렇게 하면 싱글턴 객체를 뭔가 다른 것으로 바꾸어야 할 때 코드 한 군데만 수정하면 된다. 즉, 컨테이너 설정 코드만 수정하면 원하는 대로 객체를 바꿀 수 있다.
추가적인 장점으로 **싱글턴을 직접 구현할 필요 없이** Boost.DI 프레임워크에서 자동으로 처리해 주기 때문에 싱글턴 로직을 잘못 구현할 오류의 여지를 없애준다.

</br>

## 모노스테이트(Monostate)

모노스테이트는 **싱글턴 패턴의 변형**이다. 모노스테이트는 겉보기에는 **일반 클래스와 동일하나 동작은 싱글턴처럼** 한다.

```C++
class Printer
{
private:
    static int m_id;

public:
    auto get_id() const { return m_id; }
    void set_id() (int value) { m_id = value; }
};
```

언뜻 보기에는 보통 클래스처럼 보이지만 static 데이터를 이용하여 get/set 멤버 함수가 구현되어 있다.
따라서 사용자는 일반 클래스인 것으로 알고 Printer 인스턴스를 만들지만 실제로는 모든 인스턴스가 같은 데이터를 바라본다.

모노스테이트 방식은 **상속받기가 쉬워 다형성을 활용**할 수 있고 **생존 주기도 적절히 잘 정의**된다.
모노스테이트의 가장 큰 장점은 **시스템에서 사용 중인 이미 존재하는 객체를 이용할 수 있다**는 점이다.
복수의 객체가 존재하는 것이 문제가 되지 않는다면, 대규모 구조 변경 없이 **기존 코드를 조금만 수정하여 잘 동작하는 기능에 변화를 주지 않고 싱글턴 특성을 추가**할 수 있다.

모노스테이트 방식의 단점은 명확하다. 이 방식은 **코드 깊숙이 손을 댄다**.
그리고 **static 멤버를 사용**하기 때문에 **실제 객체가 인스턴스화되어 사용되는지와 관계없이 항상 메모리를 차지**한다.

</br>

## 요약

> 💡 싱글턴을 사용하면 테스트와 리팩터링 용이성을 해칠 수 있다.

> 💡 싱글턴을 사용해야만 한다면 직접적인 사용법 대신 의존성을 주입하는 방식(IoC 컨테이너 활용)으로 활용하자.
