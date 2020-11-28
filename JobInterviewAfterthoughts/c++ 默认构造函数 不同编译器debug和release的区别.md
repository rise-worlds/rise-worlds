# c++ 默认构造函数 不同编译器debug和release的差别

这几年一直在linux上开发，用的gcc版本比较高，最近把gcc降到4.8.5（centos 7默认版本）后，出现了一些成员变量初始化的问题。

看示例：
```c++
#include <iostream>

class Test
{
public:
    Test() = default;
    // Test(const Test& value) = default;
    // Test& operator=(const Test &value) = default;
    ~Test() = default;
    int GetA() { return m_a; }
    void SetA(int value) { m_a = value; }
private:
    int m_a;
};

int main(int argc, char **args)
{
    Test test;
    Test test2 = test;
    std::cout << test.GetA() << std::endl;
    std::cout << test2.GetA() << std::endl;
    test.SetA(1000);
    Test test3 = {};
    std::cout << test.GetA() << std::endl;
    std::cout << test3.GetA() << std::endl;
    Test test4 = {test};
    std::cout << test.GetA() << std::endl;
    std::cout << test4.GetA() << std::endl;
    return 0;
}
```

|编译器|debug|release|
|---|--|--|
|gcc 4.8 |`2147483647`或者`-2147483648`|0|
|gcc 8|0|0|
|gcc 9.3|0|0|
|vs2019 msvc 142|随机数|0|
|clang 7|随机数|随机数|
|clang 10 x86|1|随机数|
|clang 10 x64|0|随机数|

***gcc 4.8 好像不同硬件上会不一样，在另一 服务器上测试都为0***

看来还是使用旧式显式初始化靠谱一些，或者这样`int m_a = 0;`进行显式初始化（需要编译器支持c++11）
如果哪天如果有人问我这种问题，我应该怎么回答呢？是不是要把高版本给过滤掉，像上学时回答考试问题一样。。。
