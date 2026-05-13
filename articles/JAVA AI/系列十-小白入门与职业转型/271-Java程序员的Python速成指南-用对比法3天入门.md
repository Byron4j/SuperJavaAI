# Java 程序员的 Python 速成指南：用对比法 3 天入门，语法对照表直接抄

## 开篇：你只需要 20% 的 Python

Python 的语法海了去了。但作为一个 Java 程序员转 Python，你根本不需要学全。

你只需要学**做 AI 开发用得到的那部分**就行。剩下的你用到时 Google 一下，1 分钟就搞定——因为你已经有编程思维了。

今天这篇文章，我用 Java 和 Python 逐条对比的方式，让你在**半小时内**就能开始写 Python。文末有一张完整的语法对照表，建议直接截图保存。

## 一、环境准备：3 分钟搞定

```bash
# 检查 Python 是否已安装
python3 --version
# Python 3.11.x 或更高版本即可

# 如果没有，macOS 用 Homebrew 安装
brew install python@3.12

# 安装 AI 开发必备的包
pip install openai langchain python-dotenv requests fastapi uvicorn
```

**跟 Java 最大的不同**：Python 不需要编译，写完直接跑。也不需要配置 classpath 这种反人类的东西。

```bash
# Java 开发流程
javac Hello.java   # 第一步：编译
java Hello         # 第二步：运行

# Python 开发流程
python hello.py    # 一步搞定！
```

## 二、变量和类型：Python 是"懒得声明"的 Java

### 变量声明

```python
# Python — 不需要声明类型，直接赋值
name = "张三"
age = 30
salary = 15000.50
is_active = True

# Java 等价写法
# String name = "张三";
# int age = 30;
# double salary = 15000.50;
# boolean isActive = true;
```

### 类型检查

```python
# Python 动态类型，变量可以随时换类型
x = 10       # x 是 int
x = "hello"  # x 现在是 str，完全合法！
# Java 里这种操作会编译报错

# 用 type() 查看类型
print(type(x))  # <class 'str'>
```

### 空值

```python
# Python 的 null 叫 None
result = None

# Java 的 null 叫 null
# Object result = null;
```

## 三、字符串操作：终于不用写 StringBuilder 了

```python
# Python 字符串的快乐
name = "张三"
age = 30

# 拼接 — Python 方式
greeting = f"你好，{name}，你今年{age}岁了"  # f-string，Java 羡慕哭了
# Java: "你好，" + name + "，你今年" + age + "岁了"

# 多行字符串 — 用三引号
sql = """
SELECT id, name, age
FROM users
WHERE age > {age}
ORDER BY id DESC
"""
# Java 15+ 才有 text blocks: """..."""

# 常用方法
s = "  Hello World  "
print(s.strip())        # "Hello World" — 去首尾空格
print(s.lower())        # "  hello world  "
print(s.upper())        # "  HELLO WORLD  "
print(s.replace("World", "Python"))  # "  Hello Python  "
print(s.split(" "))     # ['', '', 'Hello', 'World', '', ''] — 按空格分割
print("---".join(["a", "b", "c"]))  # "a---b---c" — Java 没有的 join 方式
```

**对比总结**：Python 的字符串操作比 Java 简洁太多，尤其是 f-string 和 join。

## 四、条件判断和循环：告别大括号地狱

### if-else

```python
# Python — 用缩进代替大括号
score = 85

if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
elif score >= 70:
    grade = "C"
else:
    grade = "D"

# Java 等价
# if (score >= 90) {
#     grade = "A";
# } else if (score >= 80) {
#     grade = "B";
# } ...
```

### for 循环

```python
# Python 的 for 循环 — 遍历一切可迭代的东西
# 不像 Java 那样必须写 for(int i=0; ...)

# 遍历列表
fruits = ["苹果", "香蕉", "橘子"]
for fruit in fruits:
    print(fruit)

# 带索引遍历
for i, fruit in enumerate(fruits):
    print(f"{i}: {fruit}")  # 0: 苹果, 1: 香蕉, 2: 橘子

# 遍历字典
user = {"name": "张三", "age": 30, "city": "北京"}
for key, value in user.items():
    print(f"{key} = {value}")

# range：生成数字序列
for i in range(5):      # 0, 1, 2, 3, 4
    print(i)

for i in range(2, 6):   # 2, 3, 4, 5
    print(i)
```

### while 循环

```python
# 和 Java 基本一样，只是没有大括号
count = 0
while count < 5:
    print(count)
    count += 1
```

### 列表推导式：Python 最爽的特性

```python
# List Comprehension — Java 程序员第一次见会惊呼牛逼

# 例子1：生成平方数列表
squares = [x**2 for x in range(10)]
# [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
# Java 等价：
# List<Integer> squares = IntStream.range(0, 10).map(x -> x * x).boxed().collect(Collectors.toList());

# 例子2：带条件过滤
evens = [x for x in range(20) if x % 2 == 0]
# [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

# 例子3：字典推导式
word_lengths = {word: len(word) for word in ["hello", "world", "python"]}
# {'hello': 5, 'world': 5, 'python': 6}
```

## 五、数据结构：告别冗长的类型声明

### 列表（List） — 就是 Java 的 ArrayList，但好用 100 倍

```python
# 创建列表
numbers = [1, 2, 3, 4, 5]          # 不需要 new ArrayList<>()
empty = []                          # 空列表
mixed = [1, "hello", 3.14, True]   # 可以混合类型（但不建议）

# 索引和切片（切片是 Python 的超能力！）
print(numbers[0])      # 1 — 和 Java 一样
print(numbers[-1])     # 5 — 倒数第一个！Java 没有
print(numbers[1:3])    # [2, 3] — 切片：取索引1到2
print(numbers[:3])     # [1, 2, 3] — 从头到索引2
print(numbers[2:])     # [3, 4, 5] — 从索引2到末尾
print(numbers[::-1])   # [5, 4, 3, 2, 1] — 反转！

# 常用操作
numbers.append(6)         # 追加 — add()
numbers.insert(0, 0)      # 插入 — add(index, element)
numbers.pop()             # 弹出最后一个 — removeLast()
numbers.pop(0)            # 弹出索引0的元素
numbers.remove(3)         # 按值删除 — remove(Object)
len(numbers)              # 长度 — size()
3 in numbers              # 是否包含 — contains()

# 列表合并
a = [1, 2, 3]
b = [4, 5, 6]
c = a + b                  # [1, 2, 3, 4, 5, 6] — 直接用 + 合并
```

### 元组（Tuple） — 不可变列表

```python
# 元组：创建后不能修改
point = (10, 20)
rgb = (255, 128, 0)

x, y = point  # 解包 — 一行给多个变量赋值
print(x, y)   # 10 20

# 常用于函数返回多个值
def get_user():
    return ("张三", 30, "北京")

name, age, city = get_user()
# Java 里要实现类似效果，你得定义一个类或使用 Pair/Triple
```

### 字典（Dict） — 就是 Java 的 HashMap，但简洁 10 倍

```python
# 创建字典
user = {
    "name": "张三",
    "age": 30,
    "city": "北京"
}

# 访问
print(user["name"])        # "张三"
print(user.get("email", "未知"))  # "未知" — 安全访问，有默认值

# 修改/添加
user["email"] = "zhangsan@example.com"  # 直接赋值，无需 put()

# 遍历（前面讲过）
for k, v in user.items():
    print(f"{k}: {v}")

# 合并字典
defaults = {"page": 1, "size": 10}
params = {"size": 20, "keyword": "Python"}
merged = {**defaults, **params}  # {'page': 1, 'size': 20, 'keyword': 'Python'}
# 后面覆盖前面！
```

### 集合（Set） — Java 的 HashSet

```python
# 创建
tags = {"python", "java", "ai"}

# 操作
tags.add("machine-learning")
tags.remove("java")
print("python" in tags)   # True

# 集合运算
a = {1, 2, 3}
b = {2, 3, 4}
print(a | b)  # {1, 2, 3, 4} — 并集
print(a & b)  # {2, 3}       — 交集
print(a - b)  # {1}          — 差集
```

## 六、函数：简单到令人发指

```python
# 定义函数 — 不用写 public static，不用写返回值类型
def add(a, b):
    return a + b

# 默认参数
def greet(name, greeting="你好"):
    return f"{greeting}，{name}！"

print(greet("张三"))              # 你好，张三！
print(greet("张三", "早上好"))    # 早上好，张三！
# Java 没有默认参数，得用重载或 Builder 模式

# 可变参数
def sum_all(*args):
    return sum(args)

print(sum_all(1, 2, 3, 4, 5))  # 15

# 关键字参数
def create_user(name, age, **kwargs):
    user = {"name": name, "age": age}
    user.update(kwargs)  # 把额外参数合并进去
    return user

user = create_user("张三", 30, city="北京", email="zs@example.com")
# {'name': '张三', 'age': 30, 'city': '北京', 'email': 'zs@example.com'}
```

## 七、类和面向对象：比 Java 随意得多

```python
# 类的定义
class User:
    # 构造函数（__init__ 相当于 Java 的构造器）
    def __init__(self, name, age):
        self.name = name    # self 相当于 Java 的 this
        self.age = age
        self._secret = "xxx"  # 惯例：_开头表示"私有"（实际不强制）
    
    # 方法
    def greet(self):
        return f"你好，我是{self.name}，今年{self.age}岁"
    
    # 静态方法
    @staticmethod
    def create_anonymous():
        return User("匿名", 0)
    
    # 字符串表示（相当于 toString()）
    def __str__(self):
        return f"User({self.name}, {self.age})"

# 使用
user = User("张三", 30)
print(user.greet())

# 继承
class Admin(User):
    def __init__(self, name, age, permissions):
        super().__init__(name, age)  # 调用父类构造器
        self.permissions = permissions
    
    def has_permission(self, perm):
        return perm in self.permissions

# 注意事项：
# 1. Python 支持多继承（Java 不支持）
# 2. Python 没有真正的 private，全靠约定
# 3. Python 没有 interface 关键字，但有抽象基类（ABC）
```

## 八、异常处理：try-catch 变成了 try-except

```python
# Python 的异常处理
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"除零错误：{e}")
except Exception as e:
    print(f"未知错误：{e}")
else:
    print("没有异常时执行")     # Java 没有 else 分支
finally:
    print("无论如何都会执行")   # 和 Java 的 finally 一样

# 抛异常
def divide(a, b):
    if b == 0:
        raise ValueError("除数不能为0")
    return a / b

# Java 对比
# try { ... } 
# catch (ZeroDivisionError e) { ... } 
# catch (Exception e) { ... }
# finally { ... }
```

## 九、文件操作：with 语句是真香

```python
# 读取文件 — with 自动关闭，不用写 try-finally
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()           # 读取全部
    # lines = f.readlines()      # 读取所有行
    # for line in f:             # 逐行读取
    #     print(line.strip())

# 写入文件
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("Hello, Python!\n")
    f.write("第二行内容\n")

# Java 对比
# try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
#     String line;
#     while ((line = reader.readLine()) != null) { ... }
# }
```

## 十、导入和模块：import 的逻辑和 Java 完全相反

```python
# Python 的 import
from openai import OpenAI             # 从包中导入特定类
import json                           # 导入整个模块
import numpy as np                    # 导入并起别名
from pathlib import Path              # 从标准库导入

# 这等价于 Java 的
# import com.openai.sdk.OpenAI;
# import org.json.*;
# import com.numpy.Numpy as Np;  // Java没有别名机制

# 你自己的模块
# 文件 my_utils.py
def helper():
    return "helper"

# 在另一个文件中导入
from my_utils import helper
```

## 十一、常用内置函数速查

```python
# 这些函数 Java 程序员经常找不到对应
len([1, 2, 3])           # 3 — 获取长度
type(42)                 # <class 'int'> — 查看类型
isinstance(42, int)      # True — 类型检查（instanceof）
str(42)                  # "42" — 转字符串
int("42")                # 42 — 转整数
float("3.14")            # 3.14 — 转浮点数
list("hello")            # ['h', 'e', 'l', 'l', 'o'] — 转列表
dict(name="张三")        # {'name': '张三'} — 创建字典
set([1, 2, 2, 3])        # {1, 2, 3} — 去重
sorted([3, 1, 2])        # [1, 2, 3] — 排序
reversed([1, 2, 3])      # 反向迭代器
enumerate(["a", "b"])    # [(0, 'a'), (1, 'b')]
zip([1, 2], ["a", "b"])  # [(1, 'a'), (2, 'b')] — 并行遍历
map(str, [1, 2, 3])      # 对每个元素应用函数
filter(bool, [0, 1, 0, 2]) # 过滤
sum([1, 2, 3])           # 6
max([1, 2, 3])           # 3
min([1, 2, 3])           # 1
abs(-5)                  # 5
round(3.14159, 2)        # 3.14
```

## 十二、AI 开发必须掌握的 Python 特性

### 装饰器

```python
# 装饰器 = Java 的 AOP（面向切面编程）
# 在不修改原函数的前提下增加功能

import time

def timer(func):
    """测量函数执行时间的装饰器"""
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} 耗时: {end - start:.4f}秒")
        return result
    return wrapper

@timer  # 使用装饰器，相当于 slow_function = timer(slow_function)
def slow_function():
    time.sleep(2)
    return "done"

slow_function()  # 输出：slow_function 耗时: 2.000x秒
```

### 生成器

```python
# 生成器 = 懒加载的迭代器
# 处理大数据时避免一次性加载到内存

def read_large_file(filepath):
    with open(filepath, "r") as f:
        for line in f:
            yield line.strip()    # yield 而不是 return

# 一次只处理一行，内存友好
for line in read_large_file("huge_file.txt"):
    process(line)

# Java 等价：自定义 Iterator，但代码量多 10 倍
```

### 上下文管理器

```python
# with 语句 = Java 的 try-with-resources
# 但你可以自定义！

class DatabaseConnection:
    def __enter__(self):
        print("连接数据库")
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        print("关闭数据库连接")
        # 这里自动执行，即使发生异常

with DatabaseConnection() as db:
    print("执行查询")
# 输出：
# 连接数据库
# 执行查询
# 关闭数据库连接
```

## 十三、实战：用 Python 写一个调用 OpenAI API 的脚本

现在把学到的都用上，写一个完整的 Python 程序：

```python
#!/usr/bin/env python3
"""
一个简单的 AI 聊天客户端
Java 程序员转 Python 的第一天练习
"""

import os
from openai import OpenAI
from dotenv import load_dotenv

# 加载 .env 文件中的环境变量
load_dotenv()

class AIChatClient:
    """AI 聊天客户端"""
    
    def __init__(self, model="gpt-4o-mini"):
        self.client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        self.model = model
        self.history = []  # 对话历史
    
    def chat(self, user_message):
        """发送消息并获取回复"""
        # 添加用户消息到历史
        self.history.append({
            "role": "user",
            "content": user_message
        })
        
        # 调用 API
        response = self.client.chat.completions.create(
            model=self.model,
            messages=self.history
        )
        
        # 提取回复
        assistant_message = response.choices[0].message.content
        
        # 添加助手回复到历史
        self.history.append({
            "role": "assistant",
            "content": assistant_message
        })
        
        return assistant_message
    
    def clear_history(self):
        """清空对话历史"""
        self.history = []


def main():
    """主函数"""
    print("=" * 50)
    print("AI 聊天客户端 (输入 'quit' 退出, 'clear' 清空历史)")
    print("=" * 50)
    
    # 检查 API Key
    if not os.getenv("OPENAI_API_KEY"):
        print("错误：请设置 OPENAI_API_KEY 环境变量")
        print("在 .env 文件中添加：OPENAI_API_KEY=sk-your-key-here")
        return
    
    client = AIChatClient()
    
    while True:
        try:
            user_input = input("\n你: ").strip()
            
            if not user_input:
                continue
            if user_input.lower() == 'quit':
                print("再见！")
                break
            if user_input.lower() == 'clear':
                client.clear_history()
                print("对话历史已清空")
                continue
            
            # 流式输出（模拟）
            print(f"AI: ", end="", flush=True)
            response = client.chat(user_input)
            print(response)
            
        except KeyboardInterrupt:
            print("\n\n再见！")
            break
        except Exception as e:
            print(f"\n错误：{e}")


if __name__ == "__main__":
    main()
```

运行它：
```bash
# 1. 创建 .env 文件
echo 'OPENAI_API_KEY=sk-your-key-here' > .env

# 2. 运行
python chat_client.py
```

## 十四、完整语法对照表（建议截图保存）

| 功能 | Java | Python |
|------|------|--------|
| 变量声明 | `int x = 10;` | `x = 10` |
| 常量 | `final int X = 10;` | `X = 10`（约定大写） |
| 字符串拼接 | `"a" + "b"` | `"a" + "b"` 或 `f"{a}{b}"` |
| 条件判断 | `if (x > 0) { }` | `if x > 0:` |
| 多条件 | `else if` | `elif` |
| for 循环 | `for (int i=0; i<n; i++)` | `for i in range(n):` |
| for-each | `for (var item : list)` | `for item in list:` |
| while 循环 | `while (x > 0) { }` | `while x > 0:` |
| 列表 | `List<String> list = new ArrayList<>();` | `list = []` |
| 添加元素 | `list.add("a");` | `list.append("a")` |
| 长度 | `list.size()` | `len(list)` |
| 字典/Map | `Map<String, Integer> map = new HashMap<>();` | `map = {}` |
| 存放键值 | `map.put("k", 1);` | `map["k"] = 1` |
| 获取值 | `map.get("k")` | `map["k"]` 或 `map.get("k")` |
| 函数定义 | `public static int add(int a, int b) { return a+b; }` | `def add(a, b): return a + b` |
| 类定义 | `public class Foo { }` | `class Foo:` |
| 构造器 | `public Foo() { }` | `def __init__(self):` |
| 实例化 | `Foo f = new Foo();` | `f = Foo()` |
| this/self | `this.name = name;` | `self.name = name` |
| 继承 | `class B extends A { }` | `class B(A):` |
| 接口 | `interface Foo { }` | `class Foo(ABC):`（抽象基类） |
| 空值 | `null` | `None` |
| 布尔值 | `true / false` | `True / False`（大写） |
| 逻辑与 | `&&` | `and` |
| 逻辑或 | `\|\|` | `or` |
| 逻辑非 | `!` | `not` |
| 异常捕获 | `try { } catch (Exception e) { } finally { }` | `try: ... except Exception as e: ... finally: ...` |
| 打印 | `System.out.println("hello");` | `print("hello")` |
| 导入 | `import java.util.List;` | `from typing import List` |
| 注释 | `// 单行` / `/* 多行 */` | `# 单行` / `""" 多行 """` |
| 空语句块 | `{ }` | `pass` |
| 类型检查 | `x instanceof String` | `isinstance(x, str)` |

## 十五、三天学习计划

### 第一天（2-3 小时）：基础语法
1. 安装 Python 和 VS Code
2. 跑通本文的 AI Chat 客户端示例
3. 把上面对照表里的每个语法都手敲一遍

### 第二天（2-3 小时）：项目实战
1. 用 Python 写一个简单的 API 调用脚本（调用任何免费 API）
2. 处理 JSON 数据、读写文件
3. 用 `requests` 库做一个 HTTP 抓取工具

### 第三天（2-3 小时）：AI 方向
1. 跑通 OpenAI API 的所有基本功能
2. 了解 `langchain` 的基本用法
3. 尝试用 Python 调用你以前写的 Java 服务

---

**下篇预告**：你已经会 Python 了。但你以前用 Spring Boot 写的那些 Web 项目怎么办？下一篇我教你用 FastAPI 重建它们，你会发现写 Web 服务从未如此简单。10 行代码跑起一个 HTTP 服务！
