## 어댑터 패턴

</br>
</br>

### 6.1 시나리오

</br>

픽셀을 그리는데 적합한 어떤 그리기 라이브러리가 있고 이 라이브러리를 사용해야만 그림을 그릴 수 있다고 하자.

그런데 우리는 선분이나 도형과 같은 기하학적인 모양을 그려야만 하는 상황이다.

하나하나에 점을 찍는 방식으로 무엇이든 그릴 수는 있지만, 이것으로 기하학적 도형을 그리기엔 너무 저수준의 방식이다.

따라서 기하학적 도형을 픽셀 기반 표현으로 바꾸어주는 어댑터가 필요하다.

</br>

먼저 기본적인 그리기 객체를 정의해보자.

</br>

```C++
struct Point
{
    int x;
    int y;
};

struct Line
{
    Point start;
    Point end;
};
```
</br>

이제 기하학적 도형을 모두 담을 수 있도록 일반화하자. 가장 일반적인 방법은 선분의 집합으로 표현하는 것이다.

아래와 같이 순수 가상 반복자 메소드의 쌍으로 정의한다고 하자.

</br>

```C++
struct VectorObject
{
    virtual std::vector<Line>::iterator begin() = 0;
    virtual std::vector<Line>::iterator end() = 0;
};
```
</br>

이렇게 하면 사각형 Rectangle을 다음과 같이 정의할 수 있다.

</br>

```C++
struct VectorRectangle : VectorObject
{
private:
    std::vector<Line> lines;

public:
    VectorRectangle(int x, int y, int width, int height)
    {
        lines.emplace_back(Line{Point{x, y}, Point{x + width, y}});
        lines.emplace_back(Line{Point{x + width, y}, Point{x + width, y + height}});
        lines.emplace_back(Line{Point{x, y}, Point{x, y + height}});
        lines.emplace_back(Line{Point{x, y + height}, Point{x + width, y + height}});
    }

    std::vector<Line>::iterator begin() override
    {
        return lines.begin();
    }

    std::vector<Line>::iterator end() override
    {
        return lines.end();
    }
};
```
</br>

이제 문제 상황을 만들어 보자. 화면에 선분을 비롯해 사각형을 그리고 싶은 것이다. 하지만 그럴 수 없다.

그림을 그리기 위한 인터페이스는 아래와 같은 픽셀을 찍는 함수 하나 뿐이다.

</br>

```C++
void DrawPoints(CPaintDC& dc, std::vector<Point>::iterator start, std::vector<Point>::iterator end)
{
    for(auto i = start; i != end; ++i)
        dc.SetPixel(i->x, i->y, 0);
}
```
</br>

그리기 인터페이스는 픽셀을 찍는 것밖에 없지만 우리는 선분을 그려야 한다. 즉, 어댑터가 필요하다.

</br>
</br>

### 6.2 어댑터

</br>

사각형 몇 개를 그려야 한다고 가정하자.

</br>

```C++
std::vector<shared_ptr<VectorObject>> vector_objects
{
    make_shared<VectorRectangle>(10, 10, 100, 100),
    make_shared<VectorRectangle>(30, 30, 60, 60)
}
```
</br>

이 객체들을 그리기 위해서는 사각형을 이루는 선분의 집합에서 각각의 선분이 점들의 집합으로 변환되어야 한다.

이를 위해 별도의 클래스를 만들어 선분 하나를 점의 집합으로 저장하고 각 점을 순회하도록 한다.

</br>

```C++
struct LineToPointAdapter
{
private:
    std::vector<Point> points;

public:
    LineToPointAdapter(Line& line)
    {
        int left = std::min(line.start.x, line.end.x);
        int right = std::max(line.start.x, line.end.x);
        int top = std::min(line.start.y, line.end.y);
        int bottom = std::max(line.start.y, line.end.y);

        int dx = right - left;
        int dy = line.end.y - line.start.y;

        if(dx == 0)
        {
            for(int y = top; y <= bottom; ++y)
            {
                points.emplace_back(Point{left, y});
            }
        }
        else if(dy == 0)
        {
            for(int x = left; x <= right; ++x)
            {
                points.emplace_back(Point{x, top});
            }
        }
    }

    virtual std::vector<Point>::iterator begin() { return points.begin(); }
    virtual std::vector<Point>::iterator end() { return points.end(); }
};
```
</br>

위의 어댑터를 이용하면 몇몇 기하 도형들을 그릴 수 있다.

앞서 예제 코드에서 정의한 사각형 두 개를 아래와 같이 단순한 코드로 그릴 수 있다.

</br>

```C++
for(auto& obj : vector_objects)
{
    for(auto& line : obj)
    {
        LineToPointAdapter lpo {line};
        DrawPoints(dc, lpo.begin(), lpo.end());
    }
}
```
</br>

기하 도형에서 선분 집합을 정의하면 그 선분들로 점들의 집합으로 변환한다.

그리고 그 점들을 순회할 수 있는 반복자들을 DrawPoints로 넘겨 그림을 그린다.

</br>

위 코드는 화면이 업데이트될 때마다 DrawPoints()가 호출된다. 이는 비효율적인 중복이 생긴다.

이를 해결할 수 있는 방법으로 일시적 어댑터를 생각할 필요도 있다.