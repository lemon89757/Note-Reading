# Note for Effective STL
## 第 1 条-慎重选择容器类型
- todo
  - 迭代器、指针和引用变为无效具体是指什么？
  - `istream_iterator` 用法，`istream_iterator<T>()` 可以用来表示一个数据流迭代器的末尾？

## 第 2 条-不要试图编写独立于容器类型的代码
- todo
  - 容器内部布局和 C 不兼容是指什么？

## 第 21 条-总是让比较函数在等值情况下返回 false
- 严格弱序化 `strict weak ordering`
  - 定义
  严格弱序化是一种二元关系，它满足以下性质：
    1. **反自反性（Irreflexivity）**：对于所有元素 `x`，`x < x` 为假。
    2. **反对称性（Antisymmetry）**：如果 `x < y` 为真，则 `y < x` 为假。
    3. **传递性（Transitivity）**：如果 `x < y` 为真且 `y < z` 为真，则 `x < z` 为真。
    4. **不可比关系的传递性（Transitivity of Incomparability）**：如果 `x` 和 `y` 不可比（即 `!(x < y) && !(y < x)`），且 `y` 和 `z` 不可比，则 `x` 和 `z` 也不可比。
  - 作用
  严格弱序化在 C++ 的标准模板库（STL）中非常重要，特别是在以下场景中：
    1. **排序算法**：如 `std::sort`，需要比较函数满足严格弱序化。
    2. **有序关联容器**：如 `std::set` 和 `std::map`，要求元素的关键字类型满足严格弱序化。
  - 示例
  假设有一个自定义类型 `Foo`，并希望将其作为 `std::set` 的键值（在这个例子中，`operator<` 定义了 `Foo` 的严格弱序化关系）：
  ```cpp
  struct Foo {
      int ID;
      int score;
      std::string name;
  };

  bool operator<(const Foo& lhs, const Foo& rhs) {
      return lhs.ID < rhs.ID;
  }
  ```
  - 为什么需要严格弱序化
  严格弱序化确保了比较函数的一致性和逻辑正确性。如果不满足严格弱序化，可能会导致未定义行为，例如在 `std::set` 或 `std::map` 中插入重复的关键字。
  
以下是一个简单的 C++ 示例，展示如何定义一个满足严格弱序化的比较函数：
```cpp
#include <iostream>
#include <set>

struct Point3D {
    double x, y, z;

    Point3D(double xx, double yy, double zz) : x(xx), y(yy), z(zz) {}

    bool operator<(const Point3D& rhs) const {
        if (x != rhs.x) return x < rhs.x;
        if (y != rhs.y) return y < rhs.y;
        return z < rhs.z;
    }
};

int main() {
    std::set<Point3D> points;
    points.insert(Point3D(1.0, 2.0, 3.0));
    points.insert(Point3D(1.0, 2.0, 4.0));
    points.insert(Point3D(2.0, 2.0, 3.0));

    for (const auto& p : points) {
        std::cout << "(" << p.x << ", " << p.y << ", " << p.z << ")\n";
    }

    return 0;
}
```
在这个例子中，`Point3D` 的比较函数满足严格弱序化，因此可以正确地用于 `std::set`。