# Dart 完全指南（参考官方文档编写）

> 本教程参考 Dart 官方文档（dart.dev）编写，覆盖 Dart 3.x 最新特性
> 适用读者：零基础或想系统学习 Dart 的开发者（尤其是有 Java 背景的读者）

---

## 目录

1. [概述与开发环境](#1-概述与开发环境)
2. [变量与类型系统](#2-变量与类型系统)
3. [函数](#3-函数)
4. [运算符](#4-运算符)
5. [流程控制](#5-流程控制)
6. [类](#6-类)
7. [枚举与增强枚举](#7-枚举与增强枚举)
8. [泛型](#8-泛型)
9. [空安全（Null Safety）](#9-空安全null-safety)
10. [异步编程](#10-异步编程)
11. [Dart 3 新增：Records（记录类型）](#11-dart-3-新增records记录类型)
12. [Dart 3 新增：Patterns（模式匹配）](#12-dart-3-新增patterns模式匹配)
13. [Dart 3 新增：Class Modifiers（类修饰符）](#13-dart-3-新增class-modifiers类修饰符)
14. [Dart 3 新增：Switch Expressions（Switch 表达式）](#14-dart-3-新增switch-expressionsswitch-表达式)
15. [Mixins](#15-mixins)
16. [扩展方法（Extension Methods）](#16-扩展方法extension-methods)
17. [类型别名（Type Aliases）](#17-类型别名type-aliases)
18. [元数据注解](#18-元数据注解)
19. [集合（Collections）](#19-集合collections)
20. [库与导入](#20-库与导入)
21. [常用工具库](#21-常用工具库)
22. [Dart 与 Java 关键差异速查](#22-dart-与-java-关键差异速查)

---

## 1. 概述与开发环境

### 1.1 Dart 是什么

Dart 是 Google 开发的一种面向对象编程语言，专为客户端开发优化（Flutter 即用 Dart 编写），也支持服务端和 Web 开发。

### 1.2 核心特性

- **sound null safety（空安全）**：编译时消除空指针异常
- **类型推导**：变量类型可由编译器推断
- **面向对象**：一切皆对象（包括函数、null）
- **跨平台**：AOT 编译为原生代码、JIT 编译、或编译为 JavaScript
- **快速分配**：专为 UI 渲染优化的垃圾回收

### 1.3 第一个程序

```dart
void main() {
  print('Hello, Dart!');
}
```

### 1.4 开发环境

```bash
# 安装（macOS/Linux）
brew install dart

# 或下载 SDK：https://dart.dev/get-dart

# 验证
dart --version

# 运行脚本
dart run hello.dart

# 编译为原生可执行文件
dart compile exe hello.dart

# REPL（交互式解释器）
dart
```

---

## 2. 变量与类型系统

### 2.1 变量声明

```dart
// 显式类型声明
String name = 'Alice';
int age = 30;
double height = 1.75;
bool isStudent = true;

// 类型推导（var）
var name = 'Alice';      // 推导为 String
var age = 30;           // 推导为 int
var height = 1.75;      // 推导为 double
var isStudent = true;   // 推导为 bool

// 顶层变量（库级）
const apiBaseUrl = 'https://api.example.com';
final timestamp = DateTime.now();
```

### 2.2 final vs const vs var

| 关键字 | 赋值次数 | 初值时机 | 用途 |
|--------|---------|---------|------|
| `var` | 多次 | 运行时 | 普通可变变量 |
| `final` | 一次 | 首次访问时 | 运行时常量 |
| `const` | 一次 | 编译时 | 编译时常量 |

```dart
var counter = 0;
counter = 1;  // ✅ 可重新赋值

final date = DateTime.now();  // ✅ 运行时确定
// date = DateTime.now();    // ❌ 只能赋值一次

const pi = 3.14159;  // ✅ 编译时确定
// const now = DateTime.now();  // ❌ DateTime.now() 不是编译时常量
```

### 2.3 内置类型

```dart
// 整数（没有 long/short 区分，int 在 Dart VM 上是 64 位）
int count = 42;
int hex = 0xFF;           // 十六进制
int binary = 0b1010;     // 二进制
int bigInt = 9223372036854775807;

// 浮点数
double price = 19.99;
double scientific = 1.5e10;  // 科学计数法
double infinity = double.infinity;
double nan = double.nan;

// 字符串
String greeting = 'Hello';
String name = "Dart";
String multiLine = '''
  这是多行字符串，
  可以包含换行。
''';
String raw = r'原始字符串，\n 不会转义';

// 布尔
bool isActive = true;
bool isEmpty = false;

// 符号（Runes，用于表示 Unicode 字符）
var symbol = #identifier;
var emoji = '\u{1F600}';  // 😀

// 列表（List，即数组）
var list = [1, 2, 3, 4, 5];
List<String> names = ['Alice', 'Bob', 'Charlie'];

// 集合（Set）
var set = {1, 2, 3, 3, 3};  // {1, 2, 3}，自动去重

// 映射（Map，即键值对）
var map = {'a': 1, 'b': 2, 'c': 3};
Map<String, int> scores = {'Alice': 95, 'Bob': 87};

// Runes（Unicode 码点序列）
var runes = 'Dart'.runes;  // (68, 97, 114, 116)

// 符号
var symbol = #mySymbol;
```

### 2.4 类型转换

```dart
// 数值转换
var n = int.parse('42');           // String → int
var d = double.parse('3.14');      // String → double
var s = 42.toString();             // int → String
var s2 = 3.14.toStringAsFixed(2);  // double → String (保留2位小数)
var i = int.tryParse('abc');       // 安全转换，失败返回 null

// 进制转换
var hex = 255.toRadixString(16);   // 10 进制 → 16 进制字符串
var bin = 8.toRadixString(2);      // 10 进制 → 2 进制字符串
var fromHex = int.parse('ff', radix: 16);  // 16 进制字符串 → int

// 显式类型转换
(num as int).toDouble();           // num → int（运行时检查）
(dynamic as String).toUpperCase(); // dynamic → String
```

### 2.5 类型推导

```dart
// 集合字面量的类型推导
var numbers = [1, 2, 3];           // List<int>
var words = ['a', 'b', 'c'];      // List<String>
var mixed = [1, 'a', true];        // List<dynamic>

// Set 和 Map
var numberSet = {1, 2, 3};         // Set<int>
var stringSet = <String>{'a', 'b'}; // Set<String>
var pairMap = {'x': 1, 'y': 2};   // Map<String, int>

// 带泛型的集合
var explicitlyTyped = <int>[1, 2, 3];  // List<int>
var explicitlySet = <String>{'a', 'b'}; // Set<String>
var explicitlyMap = <String, int>{'a': 1};  // Map<String, int>
```

---

## 3. 函数

### 3.1 基本函数

```dart
// 标准函数
int add(int a, int b) {
  return a + b;
}

// 单表达式函数（箭头语法）
int multiply(int a, int b) => a * b;

// 返回 void
void greet(String name) {
  print('Hello, $name!');
}

// 无参函数
String getGreeting() => 'Hello';

// void 函数
void sayNothing() => print('...');
```

### 3.2 参数类型

```dart
// 位置参数（必需）
int add(int a, int b) => a + b;
add(1, 2);  // 3

// 可选位置参数（用 [] 包裹，默认值为 null）
String greet(String name, [String? title]) {
  if (title != null) {
    return 'Hello, $title $name!';
  }
  return 'Hello, $name!';
}
greet('Alice');              // Hello, Alice!
greet('Bob', 'Mr.');         // Hello, Mr. Bob!

// 可选位置参数 + 默认值
String greetWithDefault(String name, [String title = 'Mr.']) {
  return 'Hello, $title $name!';
}

// 命名参数（用 {} 包裹，默认 required 或有默认值）
String greetNamed({required String name, String title = 'Mr.'}) {
  return 'Hello, $title $name!';
}
greetNamed(name: 'Alice');              // 必需参数
greetNamed(name: 'Bob', title: 'Dr.'); // 可选指定

// required 命名参数（强制要求）
void createUser({required String name, required int age}) {
  print('$name is $age years old');
}
createUser(name: 'Alice', age: 30);  // ✅
createUser(name: 'Bob');              // ❌ 编译错误

// 参数默认值
void configure({int timeout = 30, bool verbose = false}) {
  print('timeout=$timeout, verbose=$verbose');
}
```

### 3.3 函数类型

```dart
// 函数作为参数
void execute(void Function() fn) {
  fn();
}
execute(() => print('Executing!'));

// 带返回值的函数参数
int apply(int x, int Function(int) op) {
  return op(x);
}
apply(5, (x) => x * 2);  // 10

// 函数作为返回值
Function(int) makeMultiplier(int factor) {
  return (int x) => x * factor;
}
var triple = makeMultiplier(3);
triple(4);  // 12

// 泛型函数
T first<T>(List<T> list) {
  if (list.isEmpty) {
    throw StateError('Empty list');
  }
  return list.first;
}
first<int>([1, 2, 3]);  // 1
first<String>(['a', 'b']);  // 'a'

// typedef 声明函数类型别名
typedef BinaryOp = int Function(int a, int b);

int add(BinaryOp op) => op(1, 2);
```

### 3.4 闭包

```dart
// 函数可以捕获外部变量（闭包）
Function makeAdder(int addBy) {
  return (int x) => x + addBy;
}

var add10 = makeAdder(10);
var add5 = makeAdder(5);

add10(3);   // 13
add5(3);    // 8
```

### 3.5 级联操作符（..）

```dart
// 不使用级联
var paint = Paint();
paint.color = Colors.black;
paint.strokeCap = StrokeCap.round;
paint.strokeWidth = 5.0;

// 使用级联
var paint = Paint()
  ..color = Colors.black
  ..strokeCap = StrokeCap.round
  ..strokeWidth = 5.0;

// 级联 + 空安全
var result = nullablePaint
  ?..color = Colors.red
  ..strokeWidth = 2.0;

// 赋值表达式的返回值
var value = (paint..color = Colors.blue).color;  // value = Colors.blue
```

---

## 4. 运算符

### 4.1 算术运算符

```dart
var a = 10, b = 3;

a + b;    // 13  加法
a - b;    // 7   减法
a * b;    // 30  乘法
a / b;    // 3.333...  除法（返回 double）
a ~/ b;   // 3   整除
a % b;    // 1   取余
-a;       // -10 取反
++a;      // 11  前置递增（返回新值）
a++;      // 10  后置递增（返回旧值，然后递增）
```

### 4.2 比较运算符

```dart
var a = 10, b = 3;

a == b;   // false  相等
a != b;   // true   不等
a > b;    // true   大于
a < b;    // false  小于
a >= b;   // true   大于等于
a <= b;   // false  小于等于

// 恒等运算符（比较对象身份，非值）
identical(a, b);  // false，identity 比较
a === b;          // Dart 中没有 ===，用 identical()
```

### 4.3 类型测试运算符

```dart
// is: 运行时检查类型
if (obj is String) {
  var length = obj.length;  // 安全，obj 在这个分支中是 String
}

// is!: 否定
if (obj is! String) {
  // obj 不是 String
}

// as: 强制类型转换
var str = obj as String;

// as? 安全类型转换（失败返回 null）
var str = obj as? String;  // obj 是 String 时转换，否则 null

// 结合空安全使用
if (obj is String?) {
  // obj 可能是 String 或 null
}
```

### 4.4 赋值运算符

```dart
a = 10;       // 普通赋值
a ??= 5;      // null 赋值，只有 a 为 null 时才赋值
a += 5;       // a = a + 5
a -= 5;       // a = a - 5
a *= 2;       // a = a * 2
a /= 2;       // a = a / 2（Dart 3 后是 double 运算）
a ~/= 2;      // a = a ~/ 2
a %= 3;       // a = a % 3
a <<= 1;      // a = a << 1
a >>= 1;      // a = a >> 1
a >>>= 1;     // a = a >>> 1（零填充右移）
a &= 1;       // a = a & 1
a |= 1;       // a = a | 1
a ^= 1;       // a = a ^ 1
```

### 4.5 逻辑运算符

```dart
true && false;   // false  逻辑与
true || false;   // true   逻辑或
!true;           // false  逻辑非
```

### 4.6 位运算符

```dart
var x = 5;  // 二进制: 0101

x & 3;      // 1  按位与（0101 & 0011 = 0001）
x | 3;      // 7  按位或（0101 | 0011 = 0111）
x ^ 3;      // 6  按位异或（0101 ^ 0011 = 0110）
~x;         // -6  按位取反

x << 1;     // 10  左移（0101 << 1 = 1010）
x >> 1;     // 2   右移，算术移位（保留符号位）
x >>> 1;    // 2   零填充右移（JavaScript/Dart 特有）
```

### 4.7 三元与级联

```dart
// 三元运算符
var label = score >= 60 ? 'Pass' : 'Fail';

// ?? 运算符（null 合并）
var name = nullableName ?? 'Anonymous';

// ??= 运算符（null 赋值）
name ??= 'Anonymous';

// ?. 运算符（null 安全属性访问）
var length = nullableStr?.length;  // null 安全

// !. 运算符（非空断言（危险））
var length = nullableStr!.length;  // 如果为 null 抛异常

// ?.[ ] 安全索引访问
var item = nullableList?[0];  // null 安全索引

// 级联操作符
obj
  ..method1()
  ..method2();
```

---

## 5. 流程控制

### 5.1 if / else

```dart
var score = 85;

if (score >= 90) {
  print('A');
} else if (score >= 80) {
  print('B');
} else if (score >= 70) {
  print('C');
} else if (score >= 60) {
  print('D');
} else {
  print('F');
}

// 单行写法
var grade = score >= 90 ? 'A' : score >= 80 ? 'B' : score >= 70 ? 'C' : 'F';
```

### 5.2 switch

```dart
var command = 'PING';

switch (command) {
  case 'PING':
    print('Pinging...');
    break;
  case 'PONG':
    print('Ponging...');
    break;
  case 'QUIT':
    print('Goodbye!');
    break;
  default:
    print('Unknown command');
}

// Dart 3.10+: 多值匹配
switch (command) {
  case 'PING' || 'ping':
    print('Pinging...');
  case 'QUIT' || 'EXIT' || 'q':
    print('Goodbye!');
  default:
    print('Unknown');
}

// continue 跳转到标签（少用）
switch (command) {
  case 'skip':
    continue skipLabel;
  skipLabel:
  case 'next':
    print('Skipping to next...');
}

// switch 作为表达式（Dart 3）
var result = switch (command) {
  'PING' => 'Pinging...',
  'PONG' => 'Ponging...',
  _ => 'Unknown',
};

// switch 表达式 + 模式匹配（Dart 3）
var describe = switch ((x, y)) {
  (0, 0) => 'Origin',
  (_, 0) => 'On X-axis',
  (0, _) => 'On Y-axis',
  _ when x == y => 'Diagonal ($x, $y)',
  _ => 'Point ($x, $y)',
};
```

### 5.3 循环

```dart
// for 循环
for (var i = 0; i < 5; i++) {
  print(i);
}

// for-in 循环（遍历可迭代对象）
var items = ['apple', 'banana', 'orange'];
for (var item in items) {
  print(item);
}

// for-each（方法引用）
items.forEach(print);

// 带索引的遍历
for (var i = 0; i < items.length; i++) {
  print('$i: ${items[i]}');
}

// 或用 asMap()
items.asMap().forEach((i, item) => print('$i: $item'));

// while 循环
var n = 0;
while (n < 3) {
  print(n);
  n++;
}

// do-while 循环（至少执行一次）
do {
  print(n);
  n--;
} while (n > 0);

// break 和 continue
for (var i = 0; i < 10; i++) {
  if (i == 3) continue;   // 跳过 3
  if (i == 7) break;      // 退出循环
  print(i);
}

// 标签 + break/continue（跳出多层循环）
outer:
for (var i = 0; i < 3; i++) {
  for (var j = 0; j < 3; j++) {
    if (i == 1 && j == 1) break outer;  // 直接跳出外层循环
    print('($i, $j)');
  }
}
```

### 5.4 异常处理

```dart
// try-catch-finally
try {
  var result = 10 ~/ 0;  // 整数除以 0 抛异常
} on IntegerDivisionByZeroException {
  // 捕获特定类型的异常
  print('Cannot divide by zero');
} on FormatException catch (e) {
  // FormatException 异常
  print('Format error: $e');
  print('Stack: $e.stackTrace');
} catch (e, stack) {
  // 捕获所有其他异常
  print('Error: $e');
  print('Stack trace: $stack');
} finally {
  // 无论成功失败都执行
  print('Cleanup');
}

// rethrow 重新抛出异常
try {
  // ...
} catch (e) {
  print('Logging error: $e');
  rethrow;  // 重新抛出，保留原始堆栈
}

// throw 抛出异常
void validateAge(int age) {
  if (age < 0) {
    throw ArgumentError('Age cannot be negative');
  }
  if (age > 150) {
    throw RangeError('Age is too large');
  }
}

// 异常作为表达式（Dart 3+）
var value = switch (input) {
  'one' => 1,
  'two' => 2,
  _ => throw FormatException('Invalid input: $input'),
};
```

---

## 6. 类

### 6.1 基本类

```dart
class Point {
  // 实例变量（字段）
  double x;
  double y;

  // 构造函数
  Point(this.x, this.y);  // 简写：直接赋值给同名字段

  // 命名构造函数
  Point.origin()
      : x = 0,
        y = 0;

  Point.fromJson(Map<String, dynamic> json)
      : x = json['x'] as double,
        y = json['y'] as double;

  // 常量构造函数（所有字段 final）
  const Point.constPoint(this.x, this.y);

  // 工厂构造函数
  factory Point.fromPolar(double angle, double distance) {
    return Point(
      distance * cos(angle),
      distance * sin(angle),
    );
  }

  // Getter
  double get magnitude => sqrt(x * x + y * y);

  // Setter
  set magnitude(double value) {
    var scale = value / magnitude;
    x *= scale;
    y *= scale;
  }

  // 实例方法
  double distanceTo(Point other) {
    var dx = x - other.x;
    var dy = y - other.y;
    return sqrt(dx * dx + dy * dy);
  }

  // 运算符重载
  Point operator +(Point other) => Point(x + other.x, y + other.y);
  Point operator -(Point other) => Point(x - other.x, y - other.y);

  // @override 覆写
  @override
  String toString() => 'Point($x, $y)';

  // 静态成员
  static final origin = Point(0, 0);
  static double distanceBetween(Point a, Point b) => a.distanceTo(b);
}
```

### 6.2 构造函数详解

```dart
class Person {
  String name;
  int age;

  // 1. 默认构造函数
  Person(this.name, this.age);

  // 2. 命名构造函数
  Person.anonymous() : name = 'Anonymous', age = 0;

  // 3. 工厂构造函数（控制实例创建逻辑）
  factory Person.fromJson(Map<String, dynamic> json) {
    return Person(
      json['name'] as String,
      json['age'] as int,
    );
  }

  // 4. 重定向构造函数
  Person.guest(String name) : this(name, 0);

  // 5. 常量构造函数（用于 const 上下文）
  const Person.constPerson(this.name, this.age);
}

// 调用
var p1 = Person('Alice', 30);
var p2 = Person.fromJson({'name': 'Bob', 'age': 25});
var p3 = Person.guest('Guest');
const p4 = Person.constPerson('Const', 20);
```

### 6.3 继承

```dart
class Animal {
  String name;

  Animal(this.name);

  void speak() {
    print('$name makes a sound');
  }
}

class Dog extends Animal {
  String breed;

  // 子类构造函数必须调用父类构造函数
  Dog(String name, this.breed) : super(name);

  // 覆写方法
  @override
  void speak() {
    print('$name barks: Woof!');
  }

  // 子类独有方法
  void fetch() {
    print('$name fetches the ball');
  }
}

var dog = Dog('Buddy', 'Golden Retriever');
dog.speak();  // Buddy barks: Woof!
dog.fetch();   // Buddy fetches the ball
```

### 6.4 抽象类

```dart
// 抽象类（不能实例化）
abstract class Shape {
  // 抽象方法（子类必须实现）
  double get area;
  double get perimeter;

  // 具体方法（子类可选覆写）
  void describe() {
    print('This is a $runtimeType with area: ${area.toStringAsFixed(2)}');
  }
}

class Rectangle extends Shape {
  double width;
  double height;

  Rectangle(this.width, this.height);

  @override
  double get area => width * height;

  @override
  double get perimeter => 2 * (width + height);
}

class Circle extends Shape {
  double radius;

  Circle(this.radius);

  @override
  double get area => 3.14159 * radius * radius;

  @override
  double get perimeter => 2 * 3.14159 * radius;
}

var shape = Rectangle(3, 4);
shape.describe();  // This is a Rectangle with area: 12.00
```

### 6.5 接口

```dart
// Dart 中每个类都隐式定义了一个接口
// 用 abstract class 定义纯接口
abstract class Printable {
  void print();
  String toReadableString();
}

// 实现接口（implements）
class Report implements Printable {
  String title;
  List<String> content;

  Report(this.title, this.content);

  @override
  void print() {
    print('=== $title ===');
    for (var line in content) {
      print(line);
    }
  }

  @override
  String toReadableString() {
    return '$title: ${content.join(', ')}';
  }
}

// 一个类可以实现多个接口
abstract class Serializable {
  Map<String, dynamic> toJson();
}

class DataItem implements Printable, Serializable {
  String name;
  int value;

  DataItem(this.name, this.value);

  @override
  void print() => print('DataItem: $name = $value');

  @override
  String toReadableString() => '$name: $value';

  @override
  Map<String, dynamic> toJson() => {'name': name, 'value': value};
}
```

### 6.6 静态成员

```dart
class HttpClient {
  static const String version = '1.0';
  static int _connectionCount = 0;

  static void reset() {
    _connectionCount = 0;
    print('Connections reset');
  }

  static int get activeConnections => _connectionCount;

  void connect() {
    _connectionCount++;
    print('Connected! Active: $_connectionCount');
  }
}

HttpClient.reset();
HttpClient.connect();
print(HttpClient.version);  // 1.0
```

---

## 7. 枚举与增强枚举

### 7.1 基本枚举

```dart
enum Status {
  pending,
  active,
  completed,
  failed,
}

var status = Status.active;

// switch 中使用
switch (status) {
  case Status.pending:
    print('Waiting...');
  case Status.active:
    print('In progress');
  case Status.completed:
    print('Done');
  case Status.failed:
    print('Failed');
}

// 属性
status.name;          // 'active'
status.index;         // 1

// 遍历
for (var s in Status.values) {
  print(s);
}
```

### 7.2 增强枚举（Dart 2.17+）

```dart
enum Planet {
  mercury(0.38, 0.056),
  venus(0.91, 0.815),
  earth(1.0, 1.0),
  mars(0.38, 0.107),
  jupiter(2.34, 317.8),
  saturn(0.93, 95.2),
  uranus(0.92, 14.6),
  neptune(1.12, 17.2);

  // 枚举可以有字段
  final double surfaceGravity;
  final double mass;  // 地球质量倍数

  // 构造函数必须是 const
  const Planet(this.surfaceGravity, this.mass);

  // 枚举可以有方法
  double weightOnPlanet(double weight) {
    return weight * surfaceGravity;
  }

  // 静态方法
  static Planet find(String name) {
    return Planet.values.firstWhere(
      (p) => p.name == name,
      orElse: () => throw ArgumentError('Unknown planet: $name'),
    );
  }
}

var earth = Planet.earth;
earth.surfaceGravity;      // 1.0
earth.mass;                // 1.0
earth.weightOnPlanet(100); // 100.0（地球上的重量）
earth.name;                // 'earth'
earth.index;                // 2

// 遍历并调用方法
for (var planet in Planet.values) {
  print('${planet.name}: weight of 100kg = ${planet.weightOnPlanet(100).toStringAsFixed(2)}kg');
}
```

---

## 8. 泛型

### 8.1 泛型类

```dart
class Box<T> {
  T? _value;

  T? get value => _value;

  void put(T item) {
    _value = item;
  }

  bool get isEmpty => _value == null;
}

var intBox = Box<int>();
intBox.put(42);
print(intBox.value);  // 42

var stringBox = Box<String>();
stringBox.put('Hello');
print(stringBox.value);  // Hello
```

### 8.2 泛型接口

```dart
abstract class Repository<T> {
  Future<List<T>> getAll();
  Future<T?> getById(String id);
  Future<void> save(T item);
  Future<void> delete(String id);
}

class InMemoryRepository<T> implements Repository<T> {
  final Map<String, T> _store = {};

  @override
  Future<List<T>> getAll() async => _store.values.toList();

  @override
  Future<T?> getById(String id) async => _store[id];

  @override
  Future<void> save(T item) async {
    // 需要 item 有 id 属性，用泛型约束
  }

  @override
  Future<void> delete(String id) async {
    _store.remove(id);
  }
}
```

### 8.3 泛型约束

```dart
// T 必须实现 Comparable<T>
class SortedList<T extends Comparable<T>> {
  final List<T> _items = [];

  void add(T item) {
    _items.add(item);
    _items.sort();
  }

  List<T> get items => List.unmodifiable(_items);
}

class Person implements Comparable<Person> {
  String name;
  int age;

  Person(this.name, this.age);

  @override
  int compareTo(Person other) {
    var cmp = age.compareTo(other.age);
    if (cmp != 0) return cmp;
    return name.compareTo(other.name);
  }

  @override
  String toString() => 'Person($name, $age)';
}

var list = SortedList<Person>();
list.add(Person('Alice', 30));
list.add(Person('Bob', 25));
list.add(Person('Charlie', 30));
print(list.items);  // [Person(Bob, 25), Person(Alice, 30), Person(Charlie, 30)]
```

### 8.4 泛型方法

```dart
// 泛型函数
T first<T>(List<T> list) {
  if (list.isEmpty) throw StateError('Empty list');
  return list.first;
}

// 泛型扩展方法
extension ListExtension<T> on List<T> {
  T? firstWhereOrNull(bool Function(T) test) {
    for (var element in this) {
      if (test(element)) return element;
    }
    return null;
  }

  List<T> shuffledCopy() => List<T>.from(this)..shuffle();
}

var list = [1, 2, 3, 4, 5];
var first = list.firstWhereOrNull((n) => n > 3);  // 4
```

---

## 9. 空安全（Null Safety）

### 9.1 核心概念

Dart 的空安全是**sound**（可靠）的，意味着：
- 所有类型默认**不可空**
- 空值必须显式声明为**可空**
- 编译器强制检查，在编译时消除空指针异常

### 9.2 声明可空类型

```dart
// 不可空（默认）
String name = 'Alice';
// name = null;  // ❌ 编译错误

// 可空（用 ?）
String? nullableName;
nullableName = null;  // ✅
nullableName = 'Bob'; // ✅

// 不可空的 final（必须在声明时初始化）
final String greeting = 'Hello';  // ✅
// greeting = 'Hi';  // ❌ final 只能赋值一次

// 可空的 final（延迟初始化）
final String? nullableGreeting;  // ✅ 可以稍后赋值
nullableGreeting = 'Hi';  // ✅
```

### 9.3 空安全运算符

```dart
// ?? 空值合并运算符
String displayName = name ?? 'Anonymous';
// 等价于：name != null ? name : 'Anonymous'

// ??= 空值赋值运算符
name ??= 'Anonymous';  // 只有 name 为 null 时才赋值

// ?. 安全方法调用
int? length = nullableName?.length;
// 如果 nullableName 是 null，表达式返回 null

// ! 非空断言（慎用！运行时不安全）
int length = nullableName!.length;  // 如果是 null 抛异常

// ?.[] 安全索引访问
String? first = nullableList?[0];

// ?.() 安全函数调用
void Function()? callback;
callback?.call();

// ?.. 安全级联（当对象为 null 时不执行级联）
nullablePaint?..color = Colors.red..strokeWidth = 2.0;
```

### 9.4 late 延迟初始化

```dart
class DataLoader {
  // 声明时不知道值，但保证使用前一定会赋值
  late String data;

  Future<void> load() async {
    // 模拟异步加载
    await Future.delayed(Duration(seconds: 1));
    data = 'Loaded data';
  }

  void printData() {
    // 如果未调用 load() 就调用 printData，会抛异常
    print(data);
  }
}

var loader = DataLoader();
loader.load().then((_) => loader.printData());  // 正常
// loader.printData();  // ❌ 运行时异常：LateInitializationError
```

### 9.5 空安全结合集合

```dart
// List<String>? — 列表本身可能是 null
List<String>? nullableList;
nullableList = null;  // ✅
nullableList = ['a', 'b'];  // ✅
var first = nullableList?[0];  // null 安全

// List<String> — 列表不可 null，但元素可以是 null
List<String?> nullableElements = ['a', null, 'c'];
nullableElements[1] = null;  // ✅

// List<String>? — 列表和元素都可能是 null
List<String?>? fullyNullable = null;

// 过滤非空元素
var nonNulls = nullableElements.whereType<String>().toList();
// 或者
var nonNulls2 = nullableElements.where((e) => e != null).cast<String>().toList();
```

### 9.6 required 必传参数

```dart
// 使用 required 关键字标记命名参数必须传值
class User {
  final String name;
  final int age;
  final String? email;  // 可选参数

  User({
    required this.name,  // 必需
    required this.age,   // 必需
    this.email,          // 可选
  });
}

User(name: 'Alice', age: 30);           // ✅
User(name: 'Bob', age: 25, email: 'b'); // ✅
// User(age: 30);                       // ❌ 编译错误
```

---

## 10. 异步编程

### 10.1 Future

```dart
// Future 表示一个异步操作的结果（类似 Java 的 CompletableFuture）
Future<String> fetchUser() async {
  // 模拟网络请求
  await Future.delayed(Duration(seconds: 1));
  return 'Alice';
}

// async 函数自动返回 Future
Future<void> main() async {
  print('Fetching user...');

  // await 等待异步完成
  var user = await fetchUser();
  print('Got: $user');

  // 错误处理
  try {
    var result = await riskyOperation();
  } catch (e, stack) {
    print('Error: $e');
    print('Stack: $stack');
  }

  // finally
  try {
    await operation();
  } finally {
    cleanup();
  }
}

Future<int> riskyOperation() async {
  await Future.delayed(Duration(seconds: 1));
  throw Exception('Something went wrong');
}
```

### 10.2 Future 组合

```dart
// Future.wait — 等待多个 Future 全部完成
Future<void> loadAll() async {
  var results = await Future.wait([
    fetchUser(),
    fetchConfig(),
    fetchPreferences(),
  ]);
  print('All loaded: ${results.length} items');
}

// Future.then — 链式调用
fetchUser()
    .then((user) => fetchPosts(user))
    .then((posts) => print('Posts: $posts'))
    .catchError((e) => print('Error: $e'));

// Future.whenComplete — 无论成功失败都执行
downloadFile()
    .then((_) => print('Downloaded!'))
    .catchError((e) => print('Failed: $e'))
    .whenComplete(() => print('Done, cleanup!'));

// Future.timeout — 超时控制
await fetchData().timeout(
  Duration(seconds: 5),
  onTimeout: () => 'default value',
);

// Future.any — 第一个完成
var first = await Future.any([
  Future.delayed(Duration(seconds: 2), () => 'slow'),
  Future.delayed(Duration(seconds: 1), () => 'fast'),
]);  // 'fast'
```

### 10.3 Stream

```dart
// Stream 表示异步事件序列
Stream<int> countStream(int max) async* {
  for (var i = 1; i <= max; i++) {
    await Future.delayed(Duration(seconds: 1));
    yield i;
  }
}

// 监听 Stream
Future<void> main() async {
  // 方式1：await for
  await for (var value in countStream(5)) {
    print('Got: $value');
  }

  // 方式2：listen
  countStream(3).listen(
    (value) => print('Event: $value'),
    onError: (e) => print('Error: $e'),
    onDone: () => print('Done!'),
  );

  // 方式3：单次订阅
  var stream = countStream(3);
  var first = await stream.first;
  var last = await stream.last;
}

// Stream 转换方法
await for (var value in countStream(10)
    .where((v) => v.isEven)      // 过滤偶数
    .map((v) => v * 2)           // 翻倍
    .take(3)) {                  // 取前3个
  print(value);  // 4, 8, 12
}

// Stream 方法
await stream.where((e) => e > 0).first;
await stream.map((e) => e * 2).toList();
await stream.take(5).drain();
await stream.skip(2).first;
await stream.any((e) => e > 10);  // 是否有任意满足
await stream.every((e) => e > 0);  // 是否全部满足
await stream.reduce((a, b) => a + b);  // 累加
await stream.fold<int>(0, (sum, e) => sum + e);  // 带初始值的累加
```

### 10.4 StreamController

```dart
import 'dart:async';

class EventBus {
  final _controller = StreamController<Event>.broadcast();

  Stream<Event> get stream => _controller.stream;

  void publish(Event event) {
    _controller.add(event);
  }

  void close() {
    _controller.close();
  }
}

// 使用
var bus = EventBus();
bus.stream.listen((event) => print('Received: $event'));
bus.publish(Event('test'));

// 带广播的 Stream
var controller = StreamController<int>.broadcast();
var stream = controller.stream.asBroadcastStream();
stream.listen((v) => print('Listener 1: $v'));
stream.listen((v) => print('Listener 2: $v'));
controller.add(1);  // 两个监听者都收到
```

### 10.5 Completer

```dart
import 'dart:async';

// Completer 手动控制 Future 的完成
Future<String> delayedGreeting() {
  final completer = Completer<String>();

  Timer(Duration(seconds: 2), () {
    completer.complete('Hello after delay!');
  });

  return completer.future;
}

// 错误完成
Future<int> parseWithTimeout(String input) {
  final completer = Completer<int>();

  Timer(Duration(seconds: 5), () {
    if (!completer.isCompleted) {
      completer.completeError(TimeoutException('Parse timed out'));
    }
  });

  try {
    completer.complete(int.parse(input));
  } catch (e) {
    completer.completeError(e);
  }

  return completer.future;
}
```

### 10.6 Isolates（轻量级线程）

```dart
import 'dart:isolate';
import 'dart:math';

// 在独立 isolate 中执行计算密集型任务
Future<int> computeSum(int limit) async {
  var receivePort = ReceivePort();

  await Isolate.spawn(
    _computeIsolate,
    [receivePort.sendPort, limit],
  );

  var result = await receivePort.first as int;
  return result;
}

void _computeIsolate(List<dynamic> args) {
  SendPort sendPort = args[0];
  int limit = args[1];

  var sum = 0;
  for (var i = 0; i < limit; i++) {
    sum += i;
  }
  // 也可以用 Isolate.exit
  sendPort.send(sum);
}

// Flutter 中使用 compute 函数
// import 'package:flutter/foundation.dart';
// var result = await compute(_heavyComputation, inputData);
```

---

## 11. Dart 3 新增：Records（记录类型）

### 11.1 基本使用

Records 是**匿名的、异构的、固定的**复合类型，类似元组但支持命名字段。

```dart
// 位置记录（类似元组）
(String, int) record = ('Alice', 30);
var record2 = ('Bob', 25);  // 类型推导：(String, int)

// 访问字段（$1, $2, ...）
print(record.$1);  // Alice
print(record.$2);  // 30

// 返回多值（替代 out 参数）
(int, int) divide(int a, int b) => (a ~/ b, a % b);

var (quotient, remainder) = divide(10, 3);
print('$quotient remainder $remainder');  // 3 remainder 1
```

### 11.2 命名字段

```dart
// 命名记录
({String name, int age}) person = (name: 'Alice', age: 30);
var person2 = (name: 'Bob', age: 25);  // 类型推导：({String name, int age})

// 访问命名字段
print(person.name);  // Alice
print(person.age);   // 30

// 解构
var (name: n, age: a) = person;

// 位置 + 命名混合
({String label, int value}) mixed = (label: 'Score', value: 100);

// 五个以上字段用 $N 访问
var quintuple = (1, 2, 3, 4, 5);
print(quintuple.$5);  // 5
```

### 11.3 Records 的相等性

```dart
// Records 使用结构相等（字段值相等即相等）
var r1 = (x: 1, y: 2);
var r2 = (x: 1, y: 2);
print(r1 == r2);  // true（和 Java record 不同，Dart Record 比较字段值）

var r3 = (x: 1, y: 3);
print(r1 == r3);  // false
```

### 11.4 Records 在函数中的应用

```dart
// 替代多个返回值的封装类
({double x, double y}) calculateOffset(double angle, double distance) {
  return (
    x: distance * cos(angle),
    y: distance * sin(angle),
  );
}

// 泛型 Records
({T a, T b}) swap<T>(T a, T b) => (a: b, b: a);

var swapped = swap(1, 2);  // ({int a, int b}) (a: 2, b: 1)

// Records 作为 Map 的键（可行，因为 Records 是可哈希的）
var map = <(String, int), String>{};
map[('key', 1)] = 'value';
```

---

## 12. Dart 3 新增：Patterns（模式匹配）

### 12.1 变量声明模式

```dart
// 解构赋值
var (name, age) = ('Alice', 30);
final (x, y) = (1, 2);

// 列表解构
var [first, second, ...rest] = [1, 2, 3, 4, 5];
// first = 1, second = 2, rest = [3, 4, 5]

// Map 解构
var {'name': name, 'age': age} = {'name': 'Bob', 'age': 25};

// 嵌套解构
var ({String name, int age}, [int x, int y]) = (
  (name: 'Charlie', age: 35),
  [1, 2],
);

// 忽略不需要的值
var [_, _, third] = [1, 2, 3];  // third = 3

// 带类型的解构
var (String n, int a) = ('Dave', 40);
```

### 12.2 switch 模式匹配

```dart
void describe(Object obj) {
  switch (obj) {
    // 类型模式
    case int():
      print('Integer: $obj');
    case String():
      print('String: $obj');
    case List():
      print('List with ${obj.length} items');
    case Map():
      print('Map with ${obj.length} entries');

    // 记录模式 + 类型
    case (String name, int age):
      print('Person: $name, $age');
    case ({String name, int age}):
      print('Named person: $name, $age');

    // 常量模式
    case 0:
      print('Zero');
    case 'none':
      print('None string');

    // 范围模式
    case int n when n > 0 && n < 10:
      print('Single digit: $n');
    case double d when d < 0:
      print('Negative double: $d');

    // 或模式
    case 'yes' || 'y' || 'Y':
      print('Affirmative');
    case 'no' || 'n' || 'N':
      print('Negative');

    // 逻辑与模式
    case int n when n > 0 && n < 100:
      print('In range: $n');

    // 默认
    default:
      print('Unknown: $obj');
  }
}
```

### 12.3 逻辑与（when）子句

```dart
String classify(num value) => switch (value) {
  >= 0 && < 60 => 'F',
  >= 60 && < 70 => 'D',
  >= 70 && < 80 => 'C',
  >= 80 && < 90 => 'B',
  >= 90 && <= 100 => 'A',
  _ => 'Invalid',
};

// 关系模式
void describeNumber(num n) {
  switch (n) {
    case > 0:
      print('Positive');
    case < 0:
      print('Negative');
    case 0:
      print('Zero');
  }
}
```

### 12.4 列表与 Map 模式

```dart
// 列表模式（匹配固定长度）
void handleCommand(List<String> parts) {
  switch (parts) {
    case ['exit']:
      print('Exiting...');
    case ['cd', String dir]:
      print('Changing to $dir');
    case ['rm', ...List rest]:
      print('Removing ${rest.length} items');
    case ['mv', String from, String to]:
      print('Moving $from to $to');
    case [_, _, _]:
      print('Three items');
    default:
      print('Unknown command');
  }
}

// Map 模式
void describeConfig(Map<String, dynamic> config) {
  switch (config) {
    case {'host': String host, 'port': int port}:
      print('Server at $host:$port');
    case {'debug': true}:
      print('Debug mode enabled');
    case {'timeout': int t} when t > 0:
      print('Timeout: $t');
    case {}:
      print('Empty config');
    default:
      print('Other config');
  }
}

// 提取并同时匹配
var {'name': String name, 'age': int age} = {'name': 'Eve', 'age': 28};
```

### 12.5 Null-Check 与 Null-Assert 模式

```dart
void process(String? input) {
  // Null-check 模式：匹配非 null 并绑定类型
  if (input case String s) {
    print('Got string of length ${s.length}');
  }

  // 带 where 子句
  if (input case String s when s.isNotEmpty) {
    print('Non-empty: $s');
  }

  // switch 中的 null-check
  switch (input) {
    case null:
      print('Null input');
    case String s:
      print('String: $s');
  }
}

// Null-assert 模式（假设非 null）
void process2(String? input) {
  switch (input) {
    case String s!:
      print('Non-null: $s');  // 如果是 null 会匹配 default
    default:
      print('Null');
  }
}
```

---

## 13. Dart 3 新增：Class Modifiers（类修饰符）

### 13.1 概览

| 修饰符 | 含义 | 能被扩展 | 能被实现 | 能被混合入 |
|--------|------|---------|---------|-----------|
| `class`（无修饰符） | 普通类 | ✅ | ✅ | ❌ |
| `base class` | 基类 | ✅ | ✅ | ❌ |
| `interface class` | 接口类 | ❌ | ✅ | ❌ |
| `final class` | 密封类 | ❌ | ✅ | ❌ |
| `mixin class` | 混合类 | ❌ | ✅ | ✅ |
| `abstract class` | 抽象类 | ✅ | ✅ | ❌ |
| `mixin` | 纯 mixin | ❌ | ❌ | ✅ |

### 13.2 base class（基类）

只能在本库内被扩展，防止库外子类化：

```dart
// library_a.dart
base class BaseRepository {
  Future<List<String>> fetchAll() async {
    return ['a', 'b', 'c'];
  }
}

// main.dart
base class MyRepo extends BaseRepository {  // ✅ 同一库
  // ...
}

// 外部库无法扩展
// class ExternalRepo extends BaseRepository {}  // ❌ 编译错误

// 但可以实现
class ExternalRepo implements BaseRepository {
  @override
  Future<List<String>> fetchAll() => Future.value([]);
}
```

### 13.3 interface class（接口类）

只能被实现，不能被扩展（强制使用者通过 implements 使用）：

```dart
interface class Clickable {
  void onClick() => print('Clicked');
  void onHover() => print('Hovering');
}

class Button implements Clickable {
  @override
  void onClick() => print('Button clicked');
  @override
  void onHover() => print('Button hovering');
}

// 不能继承
// class BetterButton extends Clickable {}  // ❌ 编译错误
```

### 13.4 final class（最终类）

不能被扩展或实现（完全密封，防止外部子类化）：

```dart
final class ApiException implements Exception {
  final String message;
  final int code;

  const ApiException(this.message, this.code);

  @override
  String toString() => 'ApiException($code): $message';
}

// 外部库无法继承或实现
// class MyException extends ApiException {}  // ❌
// class MyError implements ApiException {}  // ❌
```

### 13.5 sealed class（密封类）

子类型受限，switch 必须穷举：

```dart
sealed class Result<T> {}

final class Success<T> extends Result<T> {
  final T data;
  Success(this.data);
}

final class Failure extends Result<Never> {
  final String message;
  Failure(this.message);
}

// 穷举 switch（编译器保证没有遗漏）
String describe<T>(Result<T> result) => switch (result) {
  Success(data: var d) => 'Success: $d',
  Failure(message: var m) => 'Failure: $m',
};
// ✅ 编译通过，不需要 default

// 密封层次结构的价值：
// 1. 穷举检查：新增子类时所有 switch 都要更新
// 2. 外部库无法扩展
```

### 13.6 mixin class（混合类）

既可以被混合，也可以直接实例化：

```dart
mixin class Logger {
  void log(String msg) => print('[LOG] $msg');
  void info(String msg) => print('[INFO] $msg');
  void error(String msg) => print('[ERROR] $msg');
}

// 方式1：作为 mixin 使用
class Service with Logger {
  void doSomething() {
    log('Doing something...');
  }
}

// 方式2：直接实例化
var logger = Logger();
logger.info('Direct instance');
```

### 13.7 修饰符组合

```dart
// abstract interface = 不能实例化 + 不能扩展 + 只能实现
abstract interface class Repository<T> {
  Future<List<T>> getAll();
  Future<void> save(T item);
}

// base interface = 能扩展 + 能实现 + 不能实例化
base interface class Animal {
  void speak();
}

base class Dog extends Animal {
  @override
  void speak() => print('Woof!');
}
```

---

## 14. Dart 3 新增：Switch Expressions（Switch 表达式）

### 14.1 基本语法

Switch 表达式**返回值**，是表达式而非语句：

```dart
// Switch 语句（不返回值）
switch (status) {
  case Status.pending:
    print('Pending');
    break;
  default:
    print('Unknown');
}

// Switch 表达式（返回值）
var label = switch (status) {
  Status.pending => 'Waiting',
  Status.active => 'In Progress',
  Status.completed => 'Done',
  Status.failed => 'Failed',
};
```

### 14.2 多值分支

```dart
var type = 'ping';

var result = switch (type) {
  'ping' || 'PING' || 'Ping' => 'Pinging...',
  'pong' || 'PONG' => 'Ponging...',
  _ => 'Unknown',
};
```

### 14.3 Record + switch 表达式

```dart
(String, int) getPoint() => ('x', 10);

var description = switch (getPoint()) {
  ('x', int y) when y > 0 => 'On positive X-axis: $y',
  ('y', int x) when x > 0 => 'On positive Y-axis: $x',
  ('x', int y) when y < 0 => 'On negative X-axis',
  ('y', int x) when x < 0 => 'On negative Y-axis',
  (String x, int y) when x == y => 'Diagonal at ($x, $y)',
  _ => 'Other point',
};
```

---

## 15. Mixins

### 15.1 基本使用

Mixin 是可以在多个类层次结构中复用的类：

```dart
// 定义 mixin
mixin Flyable {
  void fly() => print('Flying');
}

mixin Walkable {
  void walk() => print('Walking');
}

// 使用 with 混入
class Robot with Flyable, Walkable {}

var robot = Robot();
robot.fly();   // Flying
robot.walk();  // Walking
```

### 15.2 带接口约束的 Mixin

```dart
// Mixin 可以要求混入者实现某个接口
mixin Printer on Animal {
  void print() => print(toString());
}

class Animal {
  String name;
  Animal(this.name);

  @override
  String toString() => name;
}

// Dog 可以使用 Printer，因为它扩展了 Animal
class Dog extends Animal with Printer {
  Dog(super.name);
}

var dog = Dog('Buddy');
dog.print();  // Buddy（toString() 来自 Animal）
```

### 15.3 Mixin vs 抽象类 vs 接口

| 特性 | Mixin | 抽象类 | 接口 |
|------|-------|--------|------|
| 多继承 | ✅ | ❌ | ✅ |
| 有状态 | ✅ | ✅ | ❌ |
| 有构造方法 | ✅ | ✅ | ❌ |
| 强制实现 | ✅（通过 `on`） | ✅ | ✅ |
| 代码复用 | ✅ | ✅ | ❌ |

---

## 16. 扩展方法（Extension Methods）

### 16.1 定义扩展方法

```dart
extension StringExtension on String {
  // 扩展方法
  String get capitalized {
    if (isEmpty) return this;
    return '${this[0].toUpperCase()}${substring(1)}';
  }

  bool get isEmail => contains('@') && contains('.');

  // 扩展 getter
  int? get toInt => int.tryParse(this);

  // 扩展运算符
  String operator *(int times) => List.filled(times, this).join();

  // 扩展静态方法
  static String repeat(String s, int times) => s * times;
}

// 使用
var s = 'hello';
s.capitalized;        // 'Hello'
'hello'.isEmail;      // false
'123'.toInt;          // 123
'hello' * 3;          // 'hellohellohello'
```

### 16.2 泛型扩展

```dart
extension ListExtension<T> on List<T> {
  T? firstWhereOrNull(bool Function(T) test) {
    for (var element in this) {
      if (test(element)) return element;
    }
    return null;
  }

  List<List<T>> chunked(int size) {
    var chunks = <List<T>>[];
    for (var i = 0; i < length; i += size) {
      chunks.add(sublist(i, min(i + size, length)));
    }
    return chunks;
  }
}

var nums = [1, 2, 3, 4, 5, 6, 7];
nums.chunked(3);           // [[1,2,3], [4,5,6], [7]]
nums.firstWhereOrNull((n) => n > 4);  // 5
```

### 16.3 扩展命名空间

```dart
extension on String {
  int get wordCount => trim().split(RegExp(r'\s+')).length;
  String get reversed => split('').reversed.join();
}

// 避免命名冲突：给扩展命名
extension StringUtilities on String {
  String get shouted => toUpperCase();
}

extension AlternativeStringUtils on String {
  String get shouted => '${toUpperCase()}!';  // 不冲突
}

// 调用时显式指定
var s = 'hello';
StringUtilities(s).shouted;  // 'HELLO'
AlternativeStringUtils(s).shouted;  // 'HELLO!'
```

---

## 17. 类型别名（Type Aliases）

### 17.1 基本类型别名

```dart
// 类型别名
typedef IntList = List<int>;
typedef StringMap<T> = Map<String, T>;
typedef Comparison<T> = int Function(T a, T b);

// 使用
IntList numbers = [1, 2, 3];
StringMap<int> scores = {'Alice': 95};
Comparison<int> cmp = (a, b) => a.compareTo(b);

// 函数类型别名（更简洁）
typedef BinaryOp = int Function(int a, int b);
typedef VoidCallback = void Function();
typedef Predicate<T> = bool Function(T value);
```

### 17.2 Record 类型别名

```dart
// Record 类型别名（Dart 3.1+）
typedef Coordinate = ({double x, double y});
typedef Pair<T> = (T, T);

Coordinate origin = (x: 0, y: 0);
Pair<int> range = (0, 100);
```

---

## 18. 元数据注解

### 18.1 内置注解

```dart
// @deprecated — 标记为废弃
@deprecated
void oldFunction() {
  print('Use newFunction instead');
}

// @Deprecated 带消息
@Deprecated('Use newFunction instead')
void oldFunction2() {
  print('Use newFunction instead');
}

// @override — 覆写父类方法
class Child extends Parent {
  @override
  void method() { }
}

// @protected / @visibleForTesting / @internal
// 需要导入 dart:core 的 metadata
import 'dart:core' show protected, visibleForTesting;

// @pragma — 编译器指令
@pragma('vm:entry-point')
void main() { }  // 确保不被 tree-shaking 移除

@pragma('dart2js:assume-namespace')
library;

// @try-cable — 实验性

// @staticInterop — JS 互操作
```

### 18.2 自定义注解

```dart
// 定义注解
class MyAnnotation {
  final String value;
  final int priority;

  const MyAnnotation({this.value = '', this.priority = 0});
}

// 在类/方法/字段上使用
@MyAnnotation(value: 'custom', priority: 1)
class AnnotatedClass {
  @MyAnnotation(priority: 2)
  String annotatedField = '';

  @MyAnnotation()
  void annotatedMethod() { }
}

// 反射读取注解（使用 dart:mirrors，但 Flutter 不推荐）
// 在代码生成中使用更常见
// import 'package:source_gen/source_gen.dart';
```

### 18.3 代码生成注解（Flutter 中常用）

```dart
// json_serializable 用法示例
import 'package:json_annotation/json_annotation.dart';

part 'user.g.dart';

@JsonSerializable()
class User {
  final String name;
  final int age;

  User({required this.name, required this.age});

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
  Map<String, dynamic> toJson() => _$UserToJson(this);
}

// Hive 注解用法
import 'package:hive_ce/hive.dart';

part 'item.g.dart';

@HiveType(typeId: 0)
class Item {
  @HiveField(0)
  String name;

  @HiveField(1)
  int value;
}
```

---

## 19. 集合（Collections）

### 19.1 List（列表）

```dart
// 创建
var empty = <int>[];
var list = [1, 2, 3, 4, 5];
var fixed = List.filled(5, 0);  // [0, 0, 0, 0, 0]
var generated = List.generate(5, (i) => i * 2);  // [0, 2, 4, 6, 8]
var from = List<int>.from([1, 2, 3]);
var unmodifiable = List.unmodifiable([1, 2, 3]);

// 增删改
list.add(6);
list.addAll([7, 8, 9]);
list.insert(0, 0);           // 在索引0插入
list.insertAll(1, [10, 11]); // 在索引1插入多个
list.remove(5);              // 移除第一个匹配
list.removeAt(0);           // 移除索引0
list.removeLast();
list.removeRange(0, 2);     // 移除范围
list.removeWhere((e) => e > 3);  // 按条件移除
list.retainWhere((e) => e <= 3);  // 按条件保留
list.clear();

// 访问
list[0];          // 第一个
list.last;        // 最后一个
list.length;      // 长度
list.isEmpty;     // 是否为空
list.isNotEmpty;  // 是否非空
list.contains(3); // 是否包含

// 查找
list.indexOf(3);           // 第一个匹配索引
list.lastIndexOf(3);       // 最后一个匹配索引
list.indexWhere((e) => e > 3);  // 条件索引
list.lastIndexWhere((e) => e > 3);

// 排序
list.sort();               // 自然排序
list.sort((a, b) => a.compareTo(b));  // 自定义排序
list.sort((a, b) => b.compareTo(a));   // 降序
list.shuffle();           // 随机打乱

// 切片
list.sublist(1, 3);  // [list[1], list[2]]

// 遍历
for (var e in list) print(e);
list.forEach(print);
list.forEachIndexed((i, e) => print('$i: $e'));

// 转换
var doubled = list.map((e) => e * 2).toList();
var filtered = list.where((e) => e > 3).toList();
var summed = list.reduce((a, b) => a + b);  // 累加：1+2+3+4+5=15
var flattened = [[1,2],[3,4]].expand((e) => e).toList();  // [1,2,3,4]

// 条件转换
var first = list.firstWhere((e) => e > 3, orElse: () => -1);
var any = list.any((e) => e > 5);
var all = list.every((e) => e > 0);
```

### 19.2 Set（集合）

```dart
// 创建
var empty = <String>{};
var set = {1, 2, 3, 3, 3};  // {1, 2, 3}，自动去重
var from = Set<int>.from([1, 2, 3]);

// 增删
set.add(4);
set.addAll([5, 6]);
set.remove(3);
set.removeAll([4, 5]);
set.clear();

// 操作
var a = {1, 2, 3, 4};
var b = {3, 4, 5, 6};

a.union(b);         // {1, 2, 3, 4, 5, 6} 并集
a.intersection(b);  // {3, 4} 交集
a.difference(b);    // {1, 2} 差集
a.symmetricDifference(b);  // {1, 2, 5, 6} 对称差集

// 查询
set.contains(3);       // true
set.containsAll([1, 2]);  // true
set.lookup(3);         // 返回元素或 null

// 遍历（无序）
for (var e in set) print(e);

// 集合推导式
var doubled = {for (var e in [1, 2, 3]) e * 2};  // {2, 4, 6}
var evens = {for (var e in [1, 2, 3, 4]) if (e.isEven) e};  // {2, 4}
```

### 19.3 Map（映射）

```dart
// 创建
var empty = <String, int>{};
var map = {'a': 1, 'b': 2, 'c': 3};
var fromIterable = Map.fromIterable(['a', 'b'],
    key: (e) => e, value: (e) => e.codeUnitAt(0));
var fromEntries = Map.fromEntries([
  MapEntry('x', 1),
  MapEntry('y', 2),
]);

// 增删改
map['d'] = 4;
map.addAll({'e': 5, 'f': 6});
map.putIfAbsent('g', () => 7);  // 不存在才设置
map.update('a', (v) => v + 10);  // 更新现有
map.updateAll((k, v) => v * 2);  // 更新所有
map.remove('c');
map.removeWhere((k, v) => v > 3);
map.clear();

// 访问
map['a'];         // 1，不存在返回 null
map.keys;         // Iterable<String>
map.values;       // Iterable<int>
map.entries;      // Iterable<MapEntry>
map.length;       // 3
map.isEmpty;
map.containsKey('a');  // true
map.containsValue(1);  // true

// 遍历
map.forEach((k, v) => print('$k: $v'));
for (var entry in map.entries) {
  print('${entry.key} = ${entry.value}');
}

// 转换
var doubled = map.map((k, v) => MapEntry(k, v * 2));
var keys = map.keys.toList();
var values = map.values.toList();

// 查询
map['nonexistent'];  // null
map['nonexistent'] ?? 0;  // 0

// Map 推导式
var squares = {for (var i = 1; i <= 5; i++) i: i * i};
var filtered = {for (var e in map.entries) if (e.value > 2) e.key: e.value};
```

### 19.4 集合的 spread 运算符

```dart
// Spread 展开
var base = [1, 2, 3];
var extended = [0, ...base, 4];  // [0, 1, 2, 3, 4]

// Null-aware spread
List<int>? nullableList;
var withNull = [0, ...?nullableList, 5];  // [0, 5]（nullableList 是 null）

// Map spread
var m1 = {'a': 1, 'b': 2};
var m2 = {'c': 3, ...m1};  // {'c': 3, 'a': 1, 'b': 2}
var m3 = {'a': 0, ...m1};  // {'a': 1, 'b': 2}（后者覆盖前者）
```

### 19.5 集合的 collection-if 和 collection-for

```dart
var showExtra = true;
var items = [
  1,
  2,
  if (showExtra) 3,  // 条件包含
  4,
  if (!showExtra) 5, // else 条件
];

var includePair = false;
var pairs = {
  for (var i in [1, 2, 3])
    if (includePair) 'p$i': i,
  for (var i in [1, 2, 3])
    'd$i': i * 2,
};
```

### 19.6 Queue 和双向队列

```dart
import 'dart:collection';

// Queue（FIFO 队列）
var queue = Queue<int>();
queue.add(1);
queue.add(2);
queue.add(3);
queue.removeFirst();  // 1
queue.removeLast();   // 3

// DoubleLinkedQueue（双向队列）
var deque = DoubleLinkedQueue<String>();
deque.addFirst('first');
deque.addLast('last');
deque.removeFirst();  // 'first'
deque.removeLast();   // 'last'

// PriorityQueue（优先队列）
import 'dart:collection';

var pq = PriorityQueue<int>((a, b) => a.compareTo(b));  // 最小堆
pq.add(5);
pq.add(1);
pq.add(3);
pq.first;    // 1（最小）
pq.removeFirst();  // 移除 1
pq.isEmpty;

// 最大堆
var maxPq = PriorityQueue<int>((a, b) => b.compareTo(a));
maxPq.addAll([3, 1, 5, 2]);
maxPq.first;  // 5
```

---

## 20. 库与导入

### 20.1 导入语法

```dart
// 导入系统库
import 'dart:math';
import 'dart:async';
import 'dart:io';
import 'dart:convert';
import 'dart:collection';
import 'dart:typed_data';
import 'dart:isolate';

// 导入 Pub 包
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'package:dio/dio.dart';
import 'package:mobx/mobx.dart';

// 导入本地文件
import 'utils/logger.dart';
import '../modules/user.dart';
import 'utils.dart';  // 扩展名 .dart 可省略

// 导入并起别名
import 'dart:io' as io;
import 'package:flutter/material.dart' as material;
```

### 20.2 show / hide / as

```dart
// 只导入指定成员（show）
import 'dart:math' show pi, sqrt, pow;
// 使用：pi, sqrt, pow ✓，sin, cos ✗

// 隐藏指定成员（hide）
import 'dart:math' hide sin, cos;
// 使用：pi, sqrt, pow, sin, cos ✓，但 sin, cos 不可用

// 导入并起别名
import 'dart:math' as math;
math.sqrt(16);  // 使用别名

// 组合
import 'package:mobx/mobx.dart' show observable, action;
```

### 20.3 库组织

```dart
// library 声明
library my_library;

// part / part of（拆分文件）
// utils.dart
library;
part 'strings.dart';
part 'numbers.dart';

// strings.dart
part of 'utils.dart';
String capitalize(String s) => ...;

// 导出（export）
// library_a.dart
library;
export 'utils.dart';
export 'models.dart' show User;  // 只导出 User

// 条件导出（平台差异）
export 'stub.dart' if (dart.library.io) 'io.dart' if (dart.library.html) 'web.dart';
```

### 20.4 Deferred 延迟加载

```dart
// 延迟导入（按需加载）
import 'package:heavy_lib/heavy_lib.dart' deferred as heavy;

Future<void> load() async {
  await heavy.loadLibrary();  // 首次使用时加载
  var lib = heavy.HeavyLib();
  lib.doSomething();
}
```

---

## 21. 常用工具库

### 21.1 dart:math

```dart
import 'dart:math' as math;

math.pi;              // 3.14159...
math.e;               // 2.71828...
math.sqrt(16);        // 4.0
math.pow(2, 10);     // 1024.0
math.abs(-5);         // 5
math.max(3, 7);      // 7
math.min(3, 7);       // 3
math.ceil(4.2);      // 5
math.floor(4.8);     // 4
math.round(4.5);      // 5
math.truncate(4.8);   // 4

math.sin(math.pi / 2);  // 1.0
math.cos(0);            // 1.0
math.tan(math.pi / 4);  // 1.0
math.log(math.e);       // 1.0
math.log10(100);        // 2.0

// Random
var random = math.Random();
random.nextInt(100);    // 0-99
random.nextDouble();    // 0.0 - 1.0
random.nextBool();      // true/false

// 带种子
var seeded = math.Random(42);

// 计算
math.clamp(15, 0, 10);  // 10（超出范围时限制）
```

### 21.2 dart:convert（JSON/编码）

```dart
import 'dart:convert';

// JSON 编解码
var map = {'name': 'Alice', 'age': 30, 'scores': [95, 87]};
var json = jsonEncode(map);  // '{"name":"Alice","age":30,"scores":[95,87]}'
var decoded = jsonDecode(json) as Map<String, dynamic>;
decoded['name'];  // 'Alice'

// 自定义 toEncodable
var encoded = jsonEncode({'date': DateTime.now()}, toEncodable: (o) {
  if (o is DateTime) return o.toIso8601String();
  return o;
});

// Base64
var encoded = base64Encode([72, 101, 108, 108, 111]);  // 'SGVsbG8='
var decoded = base64Decode('SGVsbG8=');  // [72, 101, 108, 108, 111]

// UTF-8
var bytes = utf8.encode('Hello, 世界');
var str = utf8.decode(bytes);
```

### 21.3 dart:uri（URL 处理）

```dart
import 'dart:io' show Platform;

var uri = Uri.parse('https://example.com:8080/path?query=value#fragment');
uri.scheme;       // 'https'
uri.host;          // 'example.com'
uri.port;          // 8080
uri.path;         // '/path'
uri.query;         // 'query=value'
uri.fragment;      // 'fragment'
uri.queryParameters['query'];  // 'value'
uri.queryParametersAll['query'];  // ['value']

// 构建 URI
var built = Uri(
  scheme: 'https',
  host: 'api.example.com',
  port: 443,
  path: '/users',
  queryParameters: {'page': '1', 'limit': '20'},
);

// 编码
var encoded = Uri.encodeComponent('hello world');    // 'hello%20world'
var decoded = Uri.decodeComponent('hello%20world');  // 'hello world'
Uri.encodeQueryComponent('a+b');  // 'a%2Bb'

// 路径拼接
var base = Uri.parse('https://example.com/api/');
base.resolve('users/123');  // 'https://example.com/api/users/123'

// 文件路径
var fileUri = Uri.file('/path/to/file.txt');
fileUri.toFilePath();  // '/path/to/file.txt'
```

### 21.4 DateTime 与 Duration

```dart
// DateTime
var now = DateTime.now();
var utc = DateTime.utc(2025, 6, 15, 10, 30);
var local = DateTime(2025, 6, 15);
var parsed = DateTime.parse('2025-06-15T10:30:00.000Z');

// 组件
now.year;         // 2025
now.month;        // 6
now.day;          // 15
now.hour;         // 10
now.minute;       // 30
now.second;       // 0
now.millisecond;  // 0
now.weekday;     // 1(Mon) - 7(Sun)
now.isUtc;       // false

// 操作
var tomorrow = now.add(Duration(days: 1));
var yesterday = now.subtract(Duration(hours: 24));
var diff = tomorrow.difference(yesterday);  // Duration

// Duration
var dur = Duration(days: 5, hours: 12, minutes: 30, seconds: 45);
dur.inDays;        // 5
dur.inHours;       // 132
dur.inMinutes;     // 7950
dur.inSeconds;     // 477045

// 比较
now.isAfter(utc);
now.isBefore(utc);
now.isAtSameMomentAs(utc);

// 时间戳
now.millisecondsSinceEpoch;  // 毫秒时间戳
DateTime.fromMillisecondsSinceEpoch(1234567890000);
```

### 21.5 正则表达式

```dart
// 创建
var regex = RegExp(r'\d+');  // 匹配数字
var emailRegex = RegExp(r'[\w.-]+@[\w.-]+\.\w+');

// 匹配
var text = 'My phone: 123-456-7890';
var match = regex.firstMatch(text);
match?.group(0);  // '123'（第一个匹配）
match?.start;     // 10（起始位置）
match?.end;       // 13（结束位置）

// 全局匹配
for (var m in regex.allMatches(text)) {
  print('Found: ${m.group(0)}');
}

// 判断
regex.hasMatch(text);  // true/false

// 替换
text.replaceAll(RegExp(r'\d'), '*');  // 'My phone: ***-***-****'
text.replaceFirst(RegExp(r'\d'), '*');  // 只替换第一个

// 分割
'a,b;c:d'.split(RegExp(r'[,:]'));

// 分组
var groupRegex = RegExp(r'(\w+)@(\w+)\.(\w+)');
var emailMatch = groupRegex.firstMatch('user@example.com');
emailMatch?.group(0);  // 'user@example.com'（整体）
emailMatch?.group(1);  // 'user'
emailMatch?.group(2);  // 'example'
emailMatch?.group(3);  // 'com'

// 命名分组
var named = RegExp(r'(?<user>\w+)@(?<domain>\w+)\.(?<tld>\w+)');
var m = named.firstMatch('user@example.com');
m?.namedGroup('user');    // 'user'
m?.namedGroup('domain');   // 'example'
m?.namedGroup('tld');      // 'com'
```

### 21.6 数字处理

```dart
// toString
42.toString();              // '42'
42.toRadixString(2);        // '101010'（二进制）
42.toRadixString(16);       // '2a'（十六进制）
3.14159.toStringAsFixed(2); // '3.14'
3.14159.toStringAsPrecision(3);  // '3.14'

// parse
int.parse('42');            // 42
double.parse('3.14');       // 3.14
num.parse('42');           // 42（int）
int.tryParse('abc');       // null（安全）
double.tryParse('3.14');   // 3.14

// BigInt（大整数）
var big = BigInt.from(1);
big = big * 1024 * 1024 * 1024;  // 不会溢出
big.toString();
```

---

## 22. Dart 与 Java 关键差异速查

### 22.1 语法差异

| 特性 | Java | Dart |
|------|------|------|
| 空安全 | `@Nullable`（可选，运行时检查） | 默认不可空，编译时检查 |
| 类型推导 | `var`（Java 10+） | `var`（Dart 1.x 起） |
| 构造函数 | `new Class()` | `Class()`（不需要 `new`） |
| 多返回值 | 封装成类/数组/out 参数 | Records `(a, b)` |
| 模式匹配 | `instanceof` + cast | `switch (obj) { case Type(): }` |
| 扩展方法 | 工具类 + 静态方法 | `extension on Type {}` |
| Mixin | 接口 + default 方法 | `mixin` + `with` |
| 泛型 | `List<String>` | `List<String>` |
| 不可变 | `final` | `final` |
| 编译时常量 | `static final` / `const` | `const` |
| 注解 | `@Override` 等 | `@override` 等（几乎相同） |
| Lambda | `()->{}` / `->`（Java 8+） | `(){}` / `=>` |
| 异步 | `CompletableFuture` / `async` | `Future` / `async-await` |
| 并发 | `Thread` / `ExecutorService` | `Isolate` |

### 22.2 空安全对比

```java
// Java
String name = "Alice";     // 可赋值 null
@Nullable String nullable;  // 需要注解才可为 null

// Dart
String name = 'Alice';      // 不可为 null
String? nullable;           // 可空类型，显式声明
```

### 22.3 类修饰符对比

```java
// Java
public class A {}           // 公开
final class B {}            // 不可继承（Java 17+ sealed）
abstract class C {}         // 抽象
sealed class D {}          // 密封层次（Java 17+）

// Dart
class A {}                  // 可继承、可实现
base class B {}            // 可继承、可实现，但只能本库扩展
final class C {}            // 不可继承
abstract class D {}         // 抽象
sealed class E {}          // 子类受限，switch 穷举检查
interface class F {}        // 只能实现
mixin class G {}           // 可混合
```

### 22.4 Records vs Java Record

```java
// Java 17+ Record
public record Point(int x, int y) {}
var p = new Point(1, 2);

// Dart Records
var p = (x: 1, y: 2);
// 访问：p.x, p.y
```

### 22.5 Switch 对比

```java
// Java 14+ Switch Expression
String result = switch (status) {
    case PENDING -> "Waiting";
    case ACTIVE -> "In Progress";
    default -> "Unknown";
};

// Dart 3 Switch Expression
var result = switch (status) {
  Status.pending => 'Waiting',
  Status.active => 'In Progress',
  _ => 'Unknown',
};
```

---

## 附录：常用资源

- [Dart 官方文档](https://dart.dev/guides)
- [Dart 语言规范](https://dart.dev/language)
- [Effective Dart（编码规范）](https://dart.dev/effective-dart)
- [Dart 官方教程](https://dart.dev/tutorials)
- [Dart SDK 文档](https://api.dart.dev)
- [Pub.dev（包仓库）](https://pub.dev)

---

> 本文档参考 Dart 官方文档编写，涵盖 Dart 3.x 最新特性。随着 Dart 版本更新，部分特性可能会有变化，请以官方文档为准。
