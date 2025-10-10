# C++20 设计模式——可复用的面向对象设计方法

## 第三章——工厂方法和抽象工厂模式

- 工厂方法中：
  1. 按值传递返回对象时导致对象切割（只保留基类部分），因此一般返回普通指针或智能指针；
  2. 调用任何没有使用关键字 `virtual` 限定的方法都将只会得到基类中改方法所定义的行为；

- 关于 `make_shared`
    > 即使已经将 `WallFactory` 声明为 `SolidWall` 的友元类，我们仍旧无法使用 make_shared。

    1. `std::make_shared` 会调用 `SolidWall` 的构造函数，而工厂模式通常把该构造函数设为 `private/protected`，即使 `WallFactory` 是 `SolidWall` 的友元，也只能在普通代码里直接 `new SolidWall(...)`；
    2. **`make_shared` 是一个独立函数模板**，它内部并不在 `WallFactory` 的作用域里，因此不享有该友元关系，编译器依旧拒绝访问私有的构造函数，导致无法实现实例化；
    3. 友元关系不会传递给 `make_shared`，因此在需要隐藏构造函数的典型工厂实现中，只能手动 `new`，再交给 `shared_ptr` 接管；
    ```C++
    class SolidWall : public Wall {
        friend class WallFactory;
    private:
        SolidWall() {};
    };

    class WallFactory{
    public:
        static std::shared_ptr<Wall> create(){
            // 合法，WallFactory 是友元
            return std::shared_ptr<SolidWall>(new SolidWall());

            // 非法，std::make_shared 不是友元
            // return std::make_shared<SolidWall>();
        }
    }
    ```

`static BasicWallFactory factory;` 是一个 **类内声明**，必须在 **类外** 给出 **定义（初始化）**，否则会在链接时产生 “undefined reference” 错误。  
由于 `BasicWallFactory` 的构造函数是 **private**，而 `Wall` 是它的 **外围类**，因此可以在 `Wall` 的源文件中直接写定义：

```cpp
// Wall.cpp  （或任何 .cpp/.cc 文件）
#include "Wall.h"

// 在类外定义并初始化
Wall::BasicWallFactory Wall::factory{};
```

要点  
1. 必须放在 **类外** 实现；  
2. 用 **列表初始化 `{}`** 即可，因为 `BasicWallFactory` 有默认构造函数；  
3. 由于 `BasicWallFactory` 的构造函数是 `private`，**只有 `Wall` 能访问**，因此上面一句可以合法通过编译。


因为 **在类外** 的那一行

```cpp
Wall::BasicWallFactory Wall::factory = Wall::BasicWallFactory();
```

并不在 `Wall` 的成员函数体里，也不是友元，**编译器认为它是普通的“外部代码”**。  
`BasicWallFactory` 的构造函数是 `private`，普通外部代码当然无权调用，于是报错。

合法写法只有两种：

1. **列表初始化 `{}` 直接构造**（仍在 `Wall` 作用域内）  
   ```cpp
   Wall::BasicWallFactory Wall::factory{};   // OK
   ```

2. **在 `Wall` 的某个成员函数里初始化**（运行时）  
   ```cpp
   void Wall::initFactory() {
       factory = BasicWallFactory();   // OK，在 Wall 作用域里
   }
   ```

所以：  
- **类外定义时**只能用 `{}` 或 `()` 隐式构造，不能显式调用私有构造函数。  
- **赋值形式的拷贝构造 `= BasicWallFactory()`** 会被当成外部调用，故禁止。


因为 **嵌套类（inner class）对外围类（enclosing class）天然拥有“特权”**：

C++ 标准规定  
> “A nested class has access to all names (including private ones) to which its enclosing class has access, and the enclosing class has access to all names (including private ones) of its nested class.”  
> —— §11.7/1 (C++20)

换句话说：

- `BasicWallFactory` 是 `Wall` 的 **成员类**（nested class）。  
- 因此 `Wall` 的 **任何成员**（包括静态成员、成员函数、友元）都可以直接使用 `BasicWallFactory` 的 **私有成员**（构造函数、成员变量等）。  
- 反过来，`BasicWallFactory` 也能使用 `Wall` 的私有成员。

示例验证

```cpp
class Wall {
private:
    class BasicWallFactory {
        BasicWallFactory() = default;   // private
    };

public:
    static BasicWallFactory factory;    // 合法：Wall 可直接构造
};

Wall::BasicWallFactory Wall::factory{};  // OK，因为 Wall 是 enclosing class
```

而类外的普通代码就不享有这种特权，因此：

```cpp
Wall::BasicWallFactory x = Wall::BasicWallFactory(); // 错误：外部无权访问 private ctor
```

本质就是 **语言规则赋予嵌套类与外围类之间的“互访”权限**。