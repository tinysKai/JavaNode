## Python学习笔记 -- 进阶篇

### 函数

```python
# 学习函数


def say(content):
    print("say %s" % content)


say("Hello World")

print("\n======函数指定默认值========")


# 为参数指定默认值
def do(something="sleep"):
    print("i will %s" % something)


do()


# 多个参数时可指定参数名来赋值
def call(*name, phone, thing):  # 使用*来表示可任意多个值(包括0个)
    print("call %s in the num %s to tell %s" % (name, phone, thing))


# 在需跳过某个参数时可直接指定参数名
call(phone="110", thing="nothing")


# 定义一个有返回值的函数
def transferWord(word):
    return str(word).upper()


print(transferWord("hello world"))


# 使用默认值和判断来重载方法
def tell(first, last=''):
    if last:            # py将非空字符串当做True
        print("first is %s , last is %s " % (first, last))
    else:
        print("first is %s , last is empty " % first)


tell("one")             # 走else分支
tell("one", "two")      # 走if分支


print("\n==================列表方法操作===============")


def popNames(names):
    while names:
        name = names.pop()
        print(name)


names = list(range(1, 5))
popNames(names[:])
print(names)
popNames(names)
print(names)


print("\n==================传递任意数量的参数===============")


def friends(*persons):
    for friend in persons:
        print(friend)


friends("刘备", "关羽", "张飞")


print("\n==================基于任意数量参数来构建字典==============")


def buildPersonMessage(first_name, last_name, **info):
    person_info = {first_name: first_name, last_name: last_name}
    for key, val in info.items():
        person_info[key] = val
    return person_info


print(buildPersonMessage("tinys", "huang", age=24, sex="男"))

```

### 类

```python
# 学习类
# 类中的函数称为方法


class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def info(self):
        print("name is %s , age is %s" % (self.name, self.age))


person = Person("kai", 24)
person.info()


# 类继承
class Student(Person):
    def __init__(self, name, age, class_num):
        super().__init__(name, age)
        self.classNum = class_num

    def info(self):
        super().info()      # 注意方法的调用为super()
        print("%s is in the class %s" % (self.name, self.classNum))


student = Student("kai", 6, 101)
student.info()


# 类的引用
class Teacher(Person):
    def __init__(self, name, age):
        super().__init__(name, age)
        self.student = Student("tinys", 6, 101)

    def info(self):
        print("%s teacher is %s old" % (self.name, self.age))
        print("%s student is %s" % (self.name, self.student.name))


teacher = Teacher("老师", 24)
teacher.info()
```

### import引用

```python
# 学习import引入模块
from object.func import buildPersonMessage as bpm  # 可使用as来重命名

# 引入一个类就得整个类都会执行,所以以后定义类时只能定义一个执行方法(正常情况也应该是这样的)
print(bpm("tinys", "huang", age=24, sex="男"))

```

### 异常

```python
# 学习异常

# 最基本的异常捕获
try:
    print(5/1)
except Exception as e:
    print(e)

# else分支
first = 3
second = 0
try:
    answer = first / second
except Exception as e:
    print(e)
else:
    # 发生异常的话不会执行这里
    print(answer)
```

### 文件操作

```python
# 学习文件操作

# 写入空文件,会覆盖原有内容
with open("test.txt", "w") as file:
    file.write("Hello world\n")
    file.write("first end\n")

# 针对文件进行内容追加
with open("test.txt", "a") as file:
    file.write("Append  end\n")


# 一次性读取全部内容
with open("test.txt") as file:
    content = file.read()
    print(content)


# 一次一行读取文件内容
with open("test.txt") as file:
    contents = file.readlines()

for content in contents:
    print(content.strip())

```

### 单元测试

```python
# 学习单元测试
import unittest


def func(word):
    return str(word).upper()


# 基本测试基类
class FirstTest(unittest.TestCase):

    # setUp方法可做一些初始化的事情
    def setUp(self):
        print("init")

    # 单元测试方法,方法必须以test开头
    def test_func(self):
        val = func("hello")
        # 断言判断
        self.assertEqual(val, "HELLO")

```

