# effectiveModernC_lambdaexpressions
 Lambda Expressions Notes and Code Snippets from Effective Modern C++ by Scott Meyers

```cpp

#include <iostream>
#include <vector>
#include <functional>
#include <algorithm>
#include <memory>
#include <chrono>

/*
    By reference capture mode to local references can be dangerous, when closure that's created by lambda will be used in another functions.
    The problem here is that divisor will be tend to cease to exist when AddDivisorFilter goes out of the scope. And the reference in the closure
    will dangle, when the closure thats passed into the filter container is being used in another functions.
*/

class DangleLambda
{
    public:
        DangleLambda(){};
        void AddDivisorFilter();

        using FilterContainer = std::vector<std::function<bool(int)>>;
        FilterContainer filters;

    private:
        auto GetDivisor() -> int { return 5; }
};

void DangleLambda::AddDivisorFilter()
{
    auto divisor = GetDivisor(); // it might be computed with another values too. (GetDivisor(value1, value2))
    filters.emplace_back(
        [&](int value){return value % divisor == 0;} // [&divisor] works too. But DANGER!!! ref to divisor will dangle!
    );
}

/*
    But local variable will be immediately used in the closure. then there is no risk at dangling.
    Note: std::all_of returns whether all elements in a range satisfy a condition.
 */ 
template<typename C>
void workWithContainer(const C& container)
{
    auto divisor = 5; // local variable
    using ContElementType = typename C::value_type;
    if (std::all_of(std::begin(container), std::end(container),
            [&](const ContElementType& value){ return value % divisor == 0; }
        )
    ) {
        // all elements can be divided by divisor(5).
        std::cout << "workWithContainer: all elements can be divided by 5!\n";
    } else{
        std::cout << "workWithContainer: all elements can NOT be divided by 5!\n";
        // do something else...

    }
}

template<typename T>
void forwardWithContainer(T&& n)
{
    workWithContainer(std::forward<T>(n));
}

/*
    By-value-capture mode, only local variables will be captured.
    !!! Here the value "divisor" won't be copied. because it is not a local variable or a parameter for the function addFilter!!!!!!
    Whats being captured when using [=] is the Widget's this pointer, not divisor.
*/
class Widget
{
    public:
        Widget() : divisor(5) {}

        void addFilter();
        void AddDivisorFilter();
        void addFilterFixed();

    private:
        int divisor;
        std::vector<std::function<bool(int)>> filters;
};

void Widget::addFilter()
{
    filters.emplace_back(
        [=](int value){ return value % divisor == 0;} // '=' stands for the this pointer. it is being copied. not divisor...
    );

    /*
    // These wont be compiled.
    filters.emplace_back(
        [](int value){ return value % divisor == 0;}
    );

    filters.emplace_back(
        [divisor](int value){ return value % divisor == 0;}
    ); */
}

// [=] by value capture mode wont copy static variables.
void Widget::AddDivisorFilter(){
    static auto divisor = 5;
    filters.emplace_back(
        [=](int value){return value % divisor == 0;} // here divisor wont be copied. it refers to above static.
    );

    ++divisor;

    // then the lambda that have been added to filters will exhibit the new behavior(corresponding to the new value of divisor)
}

// To solve this problem, create a local variable or use generalized lambda capture.
void Widget::addFilterFixed(){
    filters.emplace_back(
        [divisor = divisor](int value){return value % divisor == 0;}
    );

    // or 

    auto divisorCopy = divisor;
    filters.emplace_back(
        [=](int value){return value % divisorCopy == 0;} // or [divisorCopy](int value){return value % divisorCopy == 0;}
    );
}

/* Things to remember for Item 31: 
    Default by-reference capture can lead to dangling references.
    Default bu-value capture is susceptible to dangling pointers (especially this), and it misleadingly suggests that lambdas are self-contained.
*/ 

/* 
    Item32: Use Init Capture to move objects into closures(c++14 or newer). But in C++11, emulate init capture via hand-written classes or std::bind
*/

// in C++14 or newer
void initcapture() 
{
    auto wp = std::make_unique<Widget>(); // Widget is just an arbitrary class. zufällig.
    auto func = [wp = std::move(wp)]{ /* do something with wp... */}; // NOTE: left side of the init capture is the scope in closure, right side is the scope where lambdas defined.
    // or
    auto func_alternativ = [wp = std::make_unique<Widget>()]{ /* do something... */};
}

// in C++11 via std::bind
void initcapturec11(){
    std::vector<double> data = {2,3,3,4};
    auto func = std::bind([](const std::vector<double>& data){
        /* do something with data... */ 
    }, std::move(data));

    // moving std::unique_ptr via std::bind
    auto wp = std::make_unique<Widget>();
    auto func2 = std::bind([](const std::unique_ptr<Widget>& wp){
        /* do someting with wp... */
    }, std::move(wp));
    // or
    auto func2_alternativ = std::bind([](const std::unique_ptr<Widget>& wp){/* do something with wp...*/}, std::make_unique<Widget>()); 
}

/* 
    So things to remember: 
        Its not possible to move-construct an object into a C++11 closure(possible for only c++14 or newer), but it is possible to move-construct
            and object into a C++11 bind object.

        Emulating move-capture in c++11 consist of move-constructing an object into a bind object, then passing the move-constructed object to the lambda
            by reference.
        
        Because the lifetime of the bind object is the same as that of the closure. its possible to treat objects in the bind as if they were in the closure.
*/

/* 
    Item33: Use decltype on auto&& parameters to std::forward them. (for auto&& x, std::forward<decltype(x)>(x))
*/ 

/* Suppose, the function "normalize" treats lvalues differently from rvalues. For this case the lambda thats written below is wrong...
    auto f = [](auto x){return func(normalize(x));};
    Because this lambda passes always a lvalue x to normalize, even if the argument that was passed to the lambda was an rvalue...

    // a class's function call operator looks like this
    class SomeClass{
        public:
            template<typename T>
            auto operator()(T x) const {
                return func(normalize(x));
            }
    };
*/

/* To solve this problem, First x has to become a universal reference and second, it has to be passed to normalize via std::forward 
    auto f = [](auto&& x){return func(normalize(std::forward<???>(x)));};
    // The question here is what type should be passed for ???. Answer is use decltype(x). It works even though decltype(x) returns rvalue references for rvalue 
        and lvalue references for lvalue(in the instantiation of std::forward T should be normally declared as a non-reference for rvalue and T& for lvalue. But nevertheless it works in the case above too.)
    auto f = [](auto&& x){return func(normalize(std::forward<decltype(x)>(x)));};
*/

/*
    Note: Lambdas allow us to use any number of parameters. So c++14 lambdas can also be varidadic:
    auto f = [](auto&&... params){ 
        return func(normalize(std::forward<decltype(params)>(params)...));
    };
 */

/*
    Item34: Prefer lambdas to std::bind
*/

// Simple and easily understandable example of lambda (setSoundL)
using Time = std::chrono::steady_clock::time_point;
enum class Sound {Beep, Siren, Whistle};
using Duration = std::chrono::steady_clock::duration;
void setAlarm(Time t, Sound s, Duration d){}

using namespace std::chrono;

auto setSoundL = [](Sound s){
    setAlarm(steady_clock::now() + hours(1), s, seconds(30));

    // for c++14 suffixes
    using namespace std::literals;
    setAlarm(steady_clock::now() + 1h, s, 30s); // c++14 but same meaning as above
};

/* first std::bind version but INCORRECT!!! (setSoundB)
    this usage is wrong because "steady_clock::now() + hours(1)" is passed to as an argument to std::bind, not to setAlarm. that means that
    the expression will be evaluated when std::bind is called, and the time resulting from the expression will be stored inside the resulting
    bind object. As a consequence the alarm will be set to go off an hour after the call to std::bind, not an hour after the call to setAlarm!
*/
using namespace std::placeholders; // needed for use of _1
auto setSoundB = std::bind(setAlarm, steady_clock::now() + hours(1), _1, seconds(30)); 

/* Fixing this problem requires telling std::bind to defer/postpone/suspend/verschieben evaluation of the expression until setAlarm is called.
   And the way to do that is to nest a second call to std::bind (with call to std::plus )inside the first one.
*/
auto setSoundB_Correct = std::bind(setAlarm, std::bind(std::plus<>(), steady_clock::now(), hours(1)), _1, seconds(30));
    // note: no need to type speficiation in c++14 but for c++11 => std::bind(setAlarm, std::bind(std::plus<steady_clock::time_point>(), steady_clock::now(), hours(1), _1, seconds(30)));

// Example for a function that returns whether its argument is between a minimum value(minVal) and maximum value(maxVal), where minVal and maxVal are local variables
void betweenExample(){
    auto minVal = 15, maxVal = 100;
    auto betweenL = [minVal, maxVal](const auto& val){ return minVal <= val && val <= maxVal; };

    const auto val = 30;
    auto betweenB = std::bind(std::logical_and<>(),
                              std::bind(std::less_equal<>(), minVal, val),
                              std::bind(std::less_equal<>(), val, maxVal)
    );

    std::cout << "lambda: " << betweenL(30) << " bind: " << betweenB() << "\n"; 
}

/* NOTE: std::bind always copies its arguments. but callers can achieve the effect of having an argument stored by reference by
            applying std::ref to it. auto compressRateB = std::bind(compress, std::ref(w), _1);
*/

int main()
{
    DangleLambda dl;
    dl.AddDivisorFilter();
    auto& x = dl.filters.at(0);
    std::cout << "result: " << x(9) << "\n";

    auto v = {5, 10, 15, 20};
    workWithContainer(v);
    forwardWithContainer(v);

    Widget w;
    w.addFilter();

    initcapture();
    initcapturec11();

    betweenExample();

    return 0;


}

```
