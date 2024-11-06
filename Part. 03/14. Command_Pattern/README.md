# 커맨드 패턴

</br>

커맨드 패턴은 어떤 객체를 활용할 때 직접 그 객체의 API를 호출하여 조작하는 대신에, 작업 명령을 보내는 방식을 제안한다.

여기에서 명령은 어떻게 하라는 지시가 담긴 데이터 클래스 이상도 이하도 아니다.

</br>
</br>

## 14.1 시나리오

</br>

은행의 마이너스 통장을 생각하자. 마이너스 통장은 현재 잔고와 인출 한도가 있다.

</br>

```C++
class BankAccount
{
private:
    int m_balance;
    int m_overdraft_limit;

public:
    void BankAccount(int balance, int limit)
        : m_balance(balance)
        , m_overdraft_limit(limit)
    {}

    void Deposit(int amount)
    {
        m_balance += amount;
        std::cout << "deposited " << amount << ", balance is now " << m_balance << std::endl;
    }

    void Withdraw(int amount)
    {
        if(m_balance - amount >= m_overdraft_limit)
        {
            m_balance -= amount;

            std::cout << "Withdrew " << amount << ", balance is now " << m_balance << std::endl;
        }
    }
};
```
</br>

Deposit()과 Withdraw()를 직접 호출하여 입금할 수도 있고 출금을 할 수도 있다.

하지만 금융 감사가 가능하도록 입출금 **내역을 기록**해야 한다고 하자. 하지만 입출금 클래스는 이미 기존에 검증되어 수정이 불가능한 상태다.

</br>
</br>

## 14.2 커맨드 패턴의 구현

</br>

먼저 커맨드의 인터페이스를 정의한다.

</br>

```C++
class Command
{
public:
    virtual void Call() const = 0;
};
```
</br>

이 인터페이스를 이용해 **은행 계좌를 대상으로 한 작업 정보를 캡슐화**하는 **BankAccountCommand**를 구체화할 수 있다.

</br>

```C++
class BankAccountCommand : public Command
{
private:
    BankAccount& m_account;
    int m_amount;

public:
    enum Action { DEPOSIT, WITHDRAW} m_action;

    BackAccountCommand(BackAccount& account, const Action action, const int amount)
        : m_account(account)        // 작업이 적용될 대상 계좌 초기화
        , m_action(action)          // 실행될 작업 종류
        , m_amount(amount)          // 입금 또는 출금할 금액의 크기 초기화
    {}
};
```
</br>

클라이언트가 이러한 정보를 제공하면 **커맨드를 이용해 입금 또는 인출 작업을 실행**할 수 있다.

</br>

```C++
void Call() const override
{
    switch(action)
    {
    case DEPOSIT:
        account.deposit(amount);
        break;

    case WITHDRAW:
        account.withdraw(amount);
        break;
    }
}
```
</br>

이러한 접근 방법으로 커맨드를 만들고 커맨드가 지정하고 있는 계좌에 변경 작업을 가할 수 있다.

</br>

```C++
BankAccount ba;
Command cmd{ba, BackAccountCommand::DEPOSIT, 100};
cmd.Call();
```

위의 코드는 주어진 계좌에 100달러를 입금한다.

Deposit(), Withdraw()가 클라이언트에 노출하고 싶지 않다면 private으로 선언하고 BackAccountCommand를 friend 선언하면 된다.

</br>
</br>

## 14.3 Undo 작업

</br>

커맨드는 일어난 작업들에 대한 모든 정보를 담고 있다.

따라서 어떤 작업에 의한 변경 내용을 되돌려서 그 작업의 이전 상태의 작업으로 리턴할 수 있다.

</br>

먼저 되돌리기 작업을 **커맨드 인터페이스에 추가 기능으로** 넣을지, 또 **하나의 커맨드로서 처리**할지 결정해야 한다.

여기서는 커맨드 인터페이스의 추가 기능으로서 구현한다. 이 부분은 **의사 결정으로 ISP를 따르는 것이 바람직**하다.

예를 들어 커맨드들 중에 비가역적으로 적용되는 작업들이 있다고 하자.

이 경우 커맨드 인터페이스 자체에 되돌리기 인터페이스가 포함될 경우 클라이언트에게 혼란을 준다.

따라서 **되돌리기가 가능한 커맨드와 일반 커맨드를 분리**하는 것이 좋을 것이다.

</br>

```C++
class Command
{
public:
    virtual void Call() = 0;
    virtual void Undo() = 0;
};
```
</br>

그리고 이를 구체화하는 BankAccountCommand를 살펴보자.

</br>

```C++
class BankAccountCommand : public Command
{
private:
    BankAccount& m_account;
    int m_amount;
    bool m_withdrawal_succeeded;

public:
    enum Action { DEPOSIT, WITHDRAW, ...} m_action;

    BackAccountCommand(BackAccount& account, const Action action, const int amount)
        : m_account(account)
        , m_action(action)
        , m_amount(amount)
        , m_withdrawal_succeeded(false)
    {}

    void Call() override
    {
        switch(action)
        {
        case DEPOSIT:
            m_account.Deposit(m_amount);
            break;
        
        case WITHDRAW:
            m_withdrawal_succeeded = m_account.Withdraw(m_amount);
            break;
        }
    }

    void Undo() override
    {
        switch(action)
        {
        case DEPOSIT:
            m_account.withdraw(m_amount);
            break;

        case WITHDRAW:
            if(m_withdrawal_succeeded)
                m_account.deposit(m_amount);

            break;
        
        ...
        }
    }
};
```
</br>

함수 내부에서 멤버 변수 withdrawal_succeeded에 값을 대입하여 변경하기 때문에 Call()은 더이상 const가 아니다.

Undo()는 const를 유지할 수 있지만 굳이 제한을 두어야 할 이유가 없다.

</br>

이와 같은 수정으로 출금 커맨드의 되돌리기가 부작용을 일으키지 않는다.

이 예제에서는 커맨드에 작업 관련 정보뿐만 아니라 임시 정보까지 저장할 수 있다는 것을 보여준다.

</br>
</br>

## 14.4 컴포지트 커맨드

</br>

계좌 A에서 계좌 B로의 이체는 다음의 두 커맨드로 수행될 수 있다.

1. 계좌 A에서 $X만큼 출금

2. 계좌 B에 $X만큼 입금

두 커맨드를 각각 호출하는 대신 **하나의 "계좌 이체" 커맨드로 감싸서 처리**할 수 있다면 편리할 것이다.

이것은 컴포지트 패턴에서 지향하는 것과 동일한 부분이다.

</br>

컴포지트 커맨드의 골격을 만들어보자. std::vector<BankAccountCommand>를 상속받기로 한다.

**std::vector에는 가상 소멸자가 없기 때문에 절대 상속받으면 안되지만** 예제로서는 충분하다.

</br>

```C++
class CompisiteBankAccountCommand : public std::vector<BankAccountCommand>, public Command
{
public:
    CompositeBankAccountCommand(const initializer_list<value_type>& items)
        : std::vector<BankAccountCommand>(items)
    {}

    void Call() override
    {
        for(auto& cmd: *this)
            cmd.Call();
    }

    void Undo() override
    {
        for(auto it = rbegin(); it != rend(); ++it)
            it->Undo();
    }
}
```
</br>

위 코드에서 볼 수 있듯이 CompositeBankAccountCommand는 std::vector면서 Command다. 즉, **컴포지트 패턴**을 따르고 있다.

생성자는 편리한 initializer_list를 이용해서 인자를 받고 있고 Call(), Undo() 두 개의 작업이 구현되어 있다.

Undo()의 경우는 **원래 커맨드의 역순으로 수행**된다는 점을 주목해서 봐야 한다.

</br>

이제 계좌 이체 작업을 컴포지트 커맨드로 구현할 차례다. 

</br>

```C++
class MoneyTransferCommand : public CompositeBankAccountCommand
{
    MoneyTransferCommand(BankAccount& from, BankAccount& to, int amount)
        : CompositeBankAccountCommand
            (
                BankAccountCommand(from, BankAccountCommand::WITHDRAW, amount),
                BackAccountCommand(to, BankAccountCommand::DEPOSIT, amount)
            )
    {}
};
```
</br>

기본 클래스의 생성자를 재사용해 두 개의 커맨드로 "계좌 이체 커맨드" 객체를 초기화하고 있다.

그리고 기본 클래스의 Call()/Undo() 구현도 재사용한다.

</br>

하지만 앞서 이야기되었듯이 문제가 있다. 기본 클래스의 구현은 명령이 실패할 수도 있다는 것을 고려하지 않고 있다.

만약 **계좌 A에서의 출금이 실패했다면 계좌 B로의 입금이 실행되어서는 안 된다**. 이때는 **전체 명령 사슬이 취소**되어야 한다.

</br>

예외 처리를 위해서는 다음과 같은 작업들이 필요하다.

* Command에 success 플래그를 추가한다.

* 각 작업의 수행마다 성공/실패 여부를 기록한다.

* 성공한 명령에 대해서만 Undo가 실행될 수 있도록 한다.

* Undo를 주의 깊게 수행하는 DependentCompositeCommand를 커맨드 클래스 사이에 둔다.

</br>

각 커맨드가 호출될 때, **직전의 커맨드가 성공적으로 수행되었을 때만 실제 수행되게** 한다.

그렇지 않은 경우 success 플래그를 false로 세팅하고 커맨드를 실행하지 않는다.

</br>

```C++
void Call() override
{
    auto ok = true;

    for(auto& cmd : *this)
    {
        if(ok)
        {
            cmd.Call();
            ok = cmd.succeeded;
        }
        else
            cmd.succeeded = false;
    }
}
```
</br>

Undo()는 오버라이딩 할 필요가 없다. 

개별 커맨드가 자체적으로 자신의 success 플래그를 확인하고 이 플래그가 true일 때만 Undo를 실행 때문이다.

</br>

구성하고 있는 모든 커맨드가 성공적으로 실행되어야만 성공하는 더 강한 조건의 컴포지트 커맨드를 생각해볼 수도 있다.

예를 들어 계좌 이체를 할 때 출금은 성공했더라도 입금이 실패했다면 전체를 취소하는 것이 올바르다.

이러한 강한 조건은 구현하기가 더 까다롭다.

</br>
</br>

## 14.5 명령과 조회의 분리

</br>

명령과 조회 분리(Command Quert Separation: CQS)는 시스템에서의 작업이 다음 두 종류 중 한 가지로 분류된다는 점에서 제안되었다.

* 명령: 어떤 시스템의 상태 변화를 야기하는 작업 지시, 어떤 결과값의 생성이 없는 것

* 조회: 어떤 결과값을 생성하는 정보 요청, 그 요청을 처리하는 시스템의 상태 변화를 일으키지 않는 것

</br>

직접적으로 상태에 대한 읽기, 쓰기 작업을 노출하고 있는 객체라면 그것을 private으로 바꾸고 각 상태에 대한 get/set 멤버 함수들 대신 단일 인터페이스로 변경할 수 있다.

예를 들어 체력과 빠르기 두 개의 속성을 가지는 크리처 클래스가 있다고 할 때, 각 속성에 대한 get/set 메소드 대신 아래처럼 단일한 커맨드 인터페이스를 제공할 수 있다.

</br>

```C++
class Creature
{
private:
    int m_strength;
    int m_agility;

public:
    Creature(int strength, int agility)
        : m_strength(strength)
        , m_agility(agility)
    {}

    void ProcessCommand(const CreatureCommands& cc);
    int ProcessQuery(const CreatureQuery& cq) const;
};
```
</br>

위 코드에는 get/set 메소드가 없다. 대신 두 개의 API 멤버 함수 ProcessCommands(), ProcessQuery()가 있다.

Creature가 **제공해야 하는 속성과 기능이 아무리 늘어나더라도 이 API 두 개만으로 처리**된다.

즉, 커맨드만으로 크리처와의 상호작용을 수행할 수 있다.

이 **API들은 각각 전용 클래스로 정의**되고 **enum 타입 클래스 CreatureAbility와 연관**된다.

</br>

```C++
enum class CreatureAbility { STRENGTH, AGILITY };

class CreatureCommand
{
public:
    CreatureAbility m_ability;
    int m_amount;
    enum Action { SET, INCREASEBY, DECREASEBY } action;
}

class CreatureQuery
{
public:
    CreatureAbility m_ability;
}
```
</br>

위 코드에서 볼 수 있듯이 명령 객체는 어떤 멤버 변수를 어떻게 얼마만큼 바꿀 것인가를 지시한다.

조회 객체는 조회할 대상만 지정한다. 여기서는 조회 결과가 함수 리턴 값으로 전달되는 것을 가정하고 조회 객체 자체에 따로 저장하지 않는다.

하지만 앞서 보았듯이 만약 다른 객체가 이 객체에 영향을 마친다면 조회 객체에 값을 저장할 필요가 생길 가능성도 있다.

</br>

```C++
void Creature::ProcessCommand(const CreatureCommand& cc)
{
    int* ability;
    switch(cc.m_ability)
    {
    case CreatureAbility::STRENGTH:
        m_ability = &m_strength;
        break;

    case CreatureAbility::AGILITY:
        m_ability = &m_agility;
        break;
    }

    switch(cc.action)
    {
    case CreatureCommand::SET:
        *ability += cc.m_amount;
        break;

    case CreatureCommand::INCREASEBY:
        *ability += cc.m_amount;
        break;

    case CreatureCommand::DECREASEBY:
        *ability -= cc.m_amount;
        break;
    }
}
```
</br>

ProcessQuery()의 정의는 더 단순하다.

</br>

```C++
int Creature::ProcessQuery(const CreatureQuery& cq) const
{
    switch(cq.ability)
    {
        case CreatureAbility::STRENGTH:
            return m_strength;
        case CreatureAbility::AGILITY:
            return m_agility;
    }

    return 0;
}
```
</br>

명령과 조회에 로깅이 필요하거나 객체를 유지해야 한다면 위 두 멤버 함수만 수정하면 된다.

유일한 문제는 get/set 방식의 API를 고집하는 클라이언트를 어떻게 지원하느냐다.

</br>

아래와 같이 Process..() 멤버 함수의 호출을 감싸고 적절히 인자를 전달함으로써 get/set을 만들 수 있다.

</br>

```C++
void Creature::SetStrength(int value)
{
    ProcessCommand(CreatureCommand{CreatureCommand::SET, CreatureAbility::STRENGTH, value});
}

void Creature::GetStrength() const
{
    return ProcessQuery(CreatureQuery{CreatureAbility::STRENGTH});
}
```
</br>