# 팩토리 패턴

</br>
</br>

## 3.1 시나리오

</br>

직표좌표계의 좌표점 정보를 저장하려고 한다.

</br>

```C++
class Point
{
private:
    float m_x;
    float m_y;

public:
    Point(const float x, const float y)
        : m_x(x)
        , m_y(y)
    {}
};
```
</br>

여기까지는 전혀 문제될 부분이 없다. 하지만 극좌표계로 좌표값을 저장해야 한다면 어떨까?

</br>

```C++
Point(const float r, const float theta)
{
    m_x = r * cos(theta);
    m_y = r * sin(theta);
}
```
</br>

직표좌표계 생성자도 두 개의 float 값을 매개변수로 가지기 때문에 **극좌표계의 생성자와 구분할 수가 없다**.

한 가지 단순한 구분 방법은 **좌표계 종류를 구분하는 enum 값을 매개변수에 추가**하는 것이다.

</br>

```C++
enum class PointType
{
    CARTESIAN,
    POLAR
};
```
</br>

Point의 생성자에 좌표계 종류를 지정하는 매개변수를 추가한다.

</br>

```C++
Point(float a, float b, PointType type = PointType::CARTESIAN)
{
    if(type == PointType::CARTESIAN)
    {
        m_x = a;
        m_y = b;
    }
    else
    {
        m_x = a * cos(b);
        m_y = a * sin(b);
    }
}
```
</br>

생성자에서 좌표값을 지정하는 변수의 이름이 x, y에서 a, b로 변경되었다.

x, y는 직교좌표계를 의미하기 때문에 극좌표계 값도 지정할 수 있도록 중립적인 변수명을 나타내기 위함이다.

이 부분은 **생성자의 사용법을 직관적으로 표현하는 데 있어서 분명 손해**보는 부분이다.

</br>

이러한 생성자는 기능적으로는 문제가 없지만 훌륭함과는 거리가 멀다. 문제점을 개선해보자.

</br>
</br>

## 3.2 팩토리 메소드

</br>

시나리오에서 생성자의 문제는 항상 타입과 같은 이름을 가진다는 것이다.

즉, 일반적인 함수명과 달리 **생성자명은 추가적인 정보를 표시할 수 없다**는 점이다.

그리고 생성자 오버로딩으로는 같은 float 타입인 x, y와 r, theta를 구분할 수 없다.

</br>

이를 위해 **생성자를 protected로** 하여 숨기고, 대신 Point **객체를 만들어 리턴하는 static 멤버 함수**를 제공할 수 있다.

</br>

```C++
class Point
{
private:
    float m_x;
    float m_y;

protected:
    Point(const float x, const float y)
        : m_x(x)
        , m_y(y)
    {}

public:
    static Point NewCartesian(float x, float y) { return { x, y }; }
    static Point NewPolar(float r, float theta) { return { r * cos(theta), r * sin(theta) }; }
};
```
</br>

여기서 각각의 static 멤버 함수들을 **팩토리 메소드**라고 한다. 팩토리 메소드가 하는 일은 Point 객체를 생성하여 리턴하는 것이다.

이는 멤버 함수의 이름과 좌표 매개변수의 이름 모두 그 의미가 무엇인지, 어떤 값이 인자로 주어져야 하는지 명확하게 표현하고 있다.

</br>

```C++
auto p1 = Point::NewPolar(5, M_PI_4);
```
</br>
</br>

## 3.3 팩토리 클래스

</br>

Point를 생성하는 함수들을 별도의 클래스에 몰아넣을 수 있다. 그런 클래스를 **팩토리 클래스**라고 한다.

</br>

```C++
class Point
{
private:
    float m_x;
    float m_y;

    Point(float x, float y)
        : m_x(x)
        , m_y(y)
    {}

public:
    friend class PointFactory;
};
```
</br>

Point **생성자는 private으로 선언**되어 사용자가 **직접 생성자를 호출하는 경우가 없도록** 한다.

</br>

또한 **Point는 PointFactory를 friend 클래스로 선언**한다. 

이 부분은 **팩토리가 Point의 생성자에 접근**할 수 있게 하려는 의도다.

이 선언이 없으면 팩토리에서 Point의 객체를 생성할 수 없다.

이는 **생성할 클래스와 그 팩토리 클래스가 동시에 만들어져야 한다**는 것을 의미한다.

</br>

```C++
class PointFactory
{
public:
    static Point NewCartesian(float x, float y) { return { x, y }; }
    static Point NewPolar(float r, float theta) { return { r * cos(theta), r * sin(theta) }; }
};
```
</br>

이제 Point 객체 생성을 전담하는 별도의 클래스가 만들어졌다.

</br>

```C++
auto p1 = PointFactory::NewCartesian(3, 4);
```
</br>
</br>

## 3.4 내부 팩토리

</br>

내부 팩토리는 생성할 타입의 **내부 클래스로서 존재하는 간단한 팩토리 클래스**를 말한다.

C#, Java 등 friend 키워드에 해당하는 문법이 없는 프로그래밍 언어들에서는 내부 팩토리를 흔하게 사용한다.

</br>

내부 팩토리의 장점은 생성할 타입의 내부 클래스이기 때문에 **private 멤버들에 자유로운 접근 권한을 가진다**는 점이다.

거꾸로 내부 클래스를 보유한 외부 클래스도 내부 클래스의 private 멤버들에 접근할 수 있다.

</br>

```C++
class Point
{
private:
    float m_x;
    float m_y;

    Point(float x, float y)
        : m_x(x)
        , m_y(y)
    {}

    class PointFactory
    {
    private:
        PointFactory();
    
    public:
        static Point NewCartesian(float x, float y) { return { x, y }; }
        static Point NewPolar(float r, float theta) { return { r * cos(theta), r * sin(theta) }; }
    };

public:
    static PointFactory Factory;
};
```
</br>

팩토리가 생성할 클래스 내부에 팩토리 클래스가 들어가 있다.

이러한 방법은 **팩토리가 생성해야 할 클래스가 단 한 종류일 때 유용**하다.

하지만 **팩토리가 여러 타입을 활용하여 객체를 생성해야 한다면 내부 팩토리는 적합하지 않다**.

</br>

내부 팩토리를 static 멤버 변수로 선언하여 다음과 같이 접근할 수 있게 한다.

</br>

```C++
auto p1 = Point::Factory.NewCartesian(3, 4);
```
</br>

만약 ::과 .을 섞어 사용하는 것이 마음에 들지 않는다면 **일관되게 ::을 사용하도록** 할 수도 있다.

**팩터리를 public으로 선언**하면 다음과 같이 사용할 수 있다.

</br>

```C++
auto p1 = Point::PointFactory::NewCartesian(3, 4);
```
</br>

또한 Point가 중복해서 나오는게 거슬린다면 **typedef**를 이용해서 없앨 수도 있다.

</br>

```C++
typedef PointFactory Factory;

Point::Factory::NewCartesian(3, 4);
```
</br>

이 방법은 가장 자연스러운 표현을 가능하게 한다.

</br>

아니면 가장 원천적인 방법으로 **내부 팩터리 Factory를 바로 호출**할 수 있게 할 수도 있다.

하지만 이는 **나중에 확장할 일이 없다는 전제**하에서만 가능하다.

</br>
</br>

## 3.5 추상 팩토리

</br>

드물지만 **여러 종류의 연관된 객체들을 생성해야 하는 경우**도 있다. **추상 팩토리**는 이런 경우에 유용하다.

앞서 살펴본 팩토리들과는 달리 추상 팩토리는 **복잡한 시스템에서만 필요성이 떠오른다**.

흔하게 사용되지는 않지만 알아두는 것이 좋다.

</br>

뜨거운 차와 커피를 판매하는 카페를 운영해야 한다고 하자.

이 두 음료는 완전히 다른 장비를 이용해 만들어진다. 이 부분을 팩토리로 모델링할 수 있다.

차와 커피는 아이스와 핫 모두 가능하지만 우선은 핫만 가능하다고 하자.

</br>

먼저, 핫 음료를 추상화하는 HotDrink를 정의하자.

</br>

```C++
class HotDrink
{
public:
    virtual void Prepare(int volume) = 0;
};
```
</br>

Prepare()는 지정된 용량의 뜨거운 음료를 준비할 때 호출한다.

HotDrink를 상속받아 Tea와 Coffe를 구현할 수 있다.

</br>

```C++
class Tea : public HotDrink
{
public:
    virtual void Prepare(int volume) override
    {
        cout << "Take tea bag, boil water, pour " << volume << "ml, add some lemon" << endl;
    }
};

class Coffee : public HotDrink
{
public:
    virtual void Prepare(int volume) override
    {
        cout << "boil water, pour " << volume << "ml, add one shot" << endl;
    }
};
```
</br>

이렇게 Tea와 Coffee가 준비되어 있으면 MakeDrink()를 만들 수 있다.

이 함수는 음료의 이름을 받아 그 해당 음료를 생성하여 리턴한다.

</br>

```C++
std::unique_ptr<HotDrink> MakeDrink(std::string type)
{
    std::unique_ptr<HotDrink> drink;

    if(type == "Tea")
    {
        drink = std::make_unique<Tea>();
        drink->Prepare(200);
    }
    else
    {
        drink = std::make_unique<Coffee>();
        drink->Prepare(50);
    }

    return drink;
}
```
</br>

그런데 앞서 언급했듯이 차를 만드는 장비와 커피를 만드는 장비가 다르다. 따라서 팩토리를 만들기로 한다.

먼저 HotDrink로 추상화되어 있으므로 HotDrinkFactory를 기반으로 모델링하기로 한다.

</br>

```C++
class HotDrinkFactory
{
public:
    virtual std::unique_ptr<HotDrink> Make() const = 0;
};
```
</br>

위의 팩토리가 바로 **추상 팩토리**다. **어떤 특정 인터페이스를 규정하고 있지만 구현 클래스가 아닌 추상 클래스**다.

즉, 이 타입이 함수 인자로서 사용될 수는 있지만 실제 음료 객체를 만들어 내려면 구체화된 구현 클래스가 필요하다.

</br>

```C++
class TeaFactory : public HotDrinkFactory
{
public:
    std::unique_ptr<HotDrink> Make() const override { return std::make_unique<Tea>(); }
};

class CoffeeFactory : public HotDrinkFactory
{
public:
    std::unique_ptr<HotDrink> Make() const override { return std::make_unique<Coffee>(); }
}; 
```
</br>

이제 좀 더 상위 수준에서 다른 종류의 음료를 만들어야 된다고 하자.

뜨거운 음료뿐만 아니라 차가운 음료도 만들 수 있어야 한다.

이를 위해 DrinkFactory를 두어 **사용 가능한 다양한 팩토리들에 대한 참조를 내부에 가지게** 할 수 있다.

</br>

```C++
class DrinkFactory
{
private:
    std::map<std::string, std::unique_ptr<HotDrinkFactory>> m_hot_factories;

public:
    DrinkFactory()
    {
        m_hot_factories["Coffee"] = std::make_unique<CoffeeFactory>();
        m_hot_factories["Tea"] = std::make_unique<TeaFactory>();
    }

    std::unique_ptr<HotDrink> make_drink(const string& name)
    {
        auto drink = m_hot_factories[name]->Make();
        drink->Prepare();

        return drink;
    }
};
```
</br>