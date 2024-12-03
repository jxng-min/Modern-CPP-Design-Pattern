## 템플릿 메소드 패턴

</br>

템플릿 메소드 패턴은 전략 패턴과 매우 유사하다. 하지만 상속을 이용한다.

핵심 원리는 어떤 알고리즘의 골격을 한 곳에 정의해두고 상세 구현을 다른 곳에 둔다는 점이다.

이 부분은 시스템을 확장한다는 측면에서 OCP 원칙을 준수한다.

</br>
</br>

### 23.1 시나리오

</br>

보드 게임류는 대부분 게임 방식이 비슷하다.

게임을 시작하고, 플레이어가 차례로 돌아가며 어떤 액션을 하고, 승자가 결정되면 게임이 종료된다.

체스, 바둑 등등 거의 대부분의 보드 게임이 이러한 게임 형태를 따른다.

따라서 아래와 같은 알고리즘을 정의할 수 있다.

</br>

```C++
class Game
{
protected:
    virtual void Start() = 0;
    virtual bool HaveWinner() = 0;
    virtual void TakeTurn() = 0;
    virtual int GetWinner() = 0;

public:
    void Run()
    {
        Start();

        While(!HaveWinner())
        {
            TakeTurn();
        }

        std::cout << "Player " << GetWinner() << "wins." << std::endl;
    }
}
```
</br>

게임을 실행하는 Run()은 단지 다른 메소드들을 호출할 뿐이다. 하지만 순서는 고정되어 있다.

각 순서에서 어떤 방식으로 작동하는지는 순수 가상 함수들을 정의하면 되지만 템플릿 메소드는 순서를 고정한다.

</br>

일반적으로 보자면 꼭 순수 가상 메소드일 필요는 없다. 특히 리턴 값이 없는 void인 경우는 더욱 그렇다.

만약 어떤 게임에 명시적인 Start() 단계가 없다면 순수 가상인 Start()는 ISP에 위반된다.

</br>

본론으로 돌아와, 이제 위 코드에 공통적인 기능을 더 추가해보겠다.

플레이어 수와 현재 플레이어의 인덱스는 모든 게임에서 의미가 있을 것이다.

</br>

```C++
class Game
{
protected:
    int current_player{0};
    int number_of_players;
    ...

public:
    explicit Game(int number_of_players)
        : number_of_players(number_of_players)
    {}
};
```
</br>

이제 Game 클래스를 확장하여 체스 게임을 구현할 수 있다.

</br>

```C++
class Chess : Game
{
private:
    int turns{0};
    int max_turns{10};

protected:
    void Start() override {}
    bool HaveWinner() override { return turns == max_turns; }
    void TakeTurn() override 
    {
        turns++;
        current_player = (current_player + 1) % number_of_players;
    }
    int GetWinner() override { return current_player; }

public:
    explicit Chess()
        : Game { 2 }
    {}
}
```
</br>

체스는 두 명의 플레이어를 필요로 하기 때문에 생성자에서 2명을 매개변수로 전달한다.

필요한 함수들을 오버라이딩하면서 가상으로 10번의 말을 두고 승자를 가리도록 구현한다.