## Python代码学习笔记

### Hello World

```python
# 第一个py程序
print("Hello World")

width = 8
long = 4
print(width * long)
```

### 元组

```python
# 学习序列的例子

seq = "一二三四五六七八"  # 定义一个字符串序列

print(seq[0])  # 通过序号下标来访问第一个元素

print(seq[-1])  # 通过序号下标来访问最后一个元素

# 包含以及非包含
print("二" in seq)
print("九" in seq)
print("二" not in seq)

# 多次的连接字符串
print(seq * 3)

# 元素组的增加,删除操作
letters = ["a", "b", "c"]
print(letters)
letters.append("d")
print(letters)
letters.remove("c")
print(letters)
```

### 判断

```python
# if逻辑的书写

first = "a"
second = "b"
if(first == second):
    print("值相等")
else:
    print("值不相等")
```

### 循环

```python
# 学习循环体

# 遍历字符串
num = "123456789"
for i in num:
    print(i)

print("------------")

# 遍历数字,并使用占位符
for i in range(1, 10):
    print("输出的值为 %s " %(i))


print("------------")

# while循环
num = 10
while num > 0:
    print(num)
    num = num - 1

```

```python

chinese_zodiac = '猴鸡狗猪鼠牛虎兔龙蛇马羊'
zodiac_name = (u'摩羯座', u'水瓶座', u'双鱼座', u'白羊座', u'金牛座', u'双子座',
           u'巨蟹座', u'狮子座', u'处女座', u'天秤座', u'天蝎座', u'射手座')
zodiac_days = ((1, 20), (2, 19), (3, 21), (4, 21), (5, 21), (6, 22),
              (7, 23), (8, 23), (9, 23), (10, 23), (11, 23), (12, 23))


# 用户输入出生年份月份和日期

year = 2015
month = 12
day = 24


n = 0
while zodiac_days[n] < (month,day):
    if month == 12 and day >23:
        break
    n += 1
# 输出生肖和星座
print(n)
print(zodiac_name[n])


print('%s 年的生肖是 %s' % (year, chinese_zodiac[year % 12]))
```

```python
# 学习使用自定义迭代器


# 自定义一个迭代器,主要使用yield关键字来保存需迭代的值
def ite(start, end, step):
    x = start
    while x < end:
        yield x
        x += step


for i in ite(1, 3, 0.5):
    print(i)
```



### 字典

```python
# 字典学习
emptyDict = {}
print(type(emptyDict))

dictNum = {"a": 1, "b": 2}
dictNum["c"]= 3
print(dictNum)

# 字典的遍历
for key in dictNum:
    print(dictNum[key])
```

### 推导式

```python
# 学习使用推导式

# 列表推导式
alist = []
for i in range(1,11):
    if (i % 2 == 0):
        alist.append( i*i )


print(alist)


blist = [i*i for i in range(1, 11) if(i % 2) == 0]

print(blist)


# 字典推导式
zodiac_name = (u'摩羯座', u'水瓶座', u'双鱼座', u'白羊座', u'金牛座', u'双子座',
           u'巨蟹座', u'狮子座', u'处女座', u'天秤座', u'天蝎座', u'射手座')

z_num = {}
for i in zodiac_name:
    z_num[i] = 0

print(z_num)

z_num2 = {i:0 for i in zodiac_name}
print(z_num2)
```

### 异常

```python
# 学习异常处理
try:
    a = open("none.txt")
except Exception as e:
    print(e)
finally:
    a.close()
```



### 文件读写

```python
# 学习文件的操作的相关

# w模式为写入
filesToWrite = open("test.txt", "w", 50, "UTF-8")
# 写文件,换行为\n
filesToWrite.write("Hello World\n")
# 写元祖
filesToWrite.writelines(["第一\n", "第二"])

filesToWrite.close()


# a模式为追加模式
filesToAppend = open("test.txt", "a", 50, "UTF-8")
filesToAppend.write("\n追加到文件末尾")
filesToAppend.close()
```

```python
# 学习文件读取
# 默认情况下的文件模式为读模式r
fileToRead = open("test.txt", "r", 50, "utf-8")

for line in fileToRead.readlines():
    # print(line, end="")          # 使用end属性来指定print的换行取消即可
    print(line.strip("\n"))         # 也可以使用strip方法来去除换行


print("\n---readLines end---")

fileToRead.seek(0)
content = fileToRead.read()
print(content.strip("\n"))

print("\n---read end---")

fileToRead.close()
```

```python
# 学习使用with来读取文件 相当于java的try-with-resource

# 使用了with语句之后就无需显式关闭文件了
with open('test.txt',  encoding='UTF-8') as file:
    for line in file:
        print(line, end='')

```

### 函数

```python
# 学习使用函数


# 无返回值的
def say(word):
    print("I am say" + word)


say("Good")


# 有返回值的
def add(num1, num2):
    return num1 + num2


print(add(1, 2))
```

```python
# 学习可变长参数

# 参数前面带*的即为可变参数
def func(first, *other):
    print(1 + len(other))


func("a")

func("a", "b")

func("a", "b", "c")
```

```python
# 学习函数作用域


word = "asd"


# 正常情况下函数内部的定义优先于外部的变量定义
def say():
    word = "qwe"
    print(word)


say()
print(word)


#  使用global定义的变量即可覆盖上面定义的全局变量
def globalSay():
    global word
    word = "zxc"
    print(word)


globalSay()
print(word)

```

### lambda表达式

```python
# 学习lambda表达式的例子


from functools import reduce


# 正常的函数定义
def add(a, b):
    return a + b


print(add(1, 2))

# 使用lambda来定义函数
add2 = lambda a, b: a + b
print(add2(2, 3))


# 学习内置函数filter中使用lambda,filter的作用是基于传递的函数来过滤数据
num = [1, 2, 3, 4, 5, 6, 7, 8]
print(list(filter(lambda x: x > 2, num)))   # filter函数的结果是filter对象,需强制转换为list


# 学习map函数,使用lambda表达式来定义一个针对列表每个元素加一的操作
mapNum = [2, 4, 7]
print(list(map(lambda x: x+1, mapNum)))  # map函数的返回结果是map,所以也需转换为list对象


# 学习reduce函数,两两参数相加,并且基于初始值1来递增,如1+1+2+3+4...
print(reduce(lambda x, y: x + y, num, 1))


# 学习zip函数,这里实现了字典的key-val对调
aDict = {"a": "aa", "b": "bb"}
print(list(zip(aDict.values(),aDict.keys())))
```

### 闭包

```python
# 学习闭包


# 使用闭包来解决函数方程,如 a * x + b = y,已知a,b的情况,输入x,求y
def fixNum(a, b):
    def inputNum(x):
        return a * x + b
    return inputNum


inputNum = fixNum(2, 3)
# 使用闭包之后,当a,b值不变时只需传递x的值即可
print(inputNum(5))
print(inputNum(10))

```

```python
# 学习装饰器的例子
import time


# 定义一个装饰器,里面使用闭包的方式,但装饰器传递的是函数,闭包多传递的是参数
def clock(func):
    def wrapper():
        start = time.time()
        func()
        end = time.time()
        print("耗时 %s 秒" % (end - start))

    return wrapper


# 基于注解的语法糖来使用装饰器
@clock
def run():
    time.sleep(1)


run()

#################################################


# 定义一个能传递参数的装饰器
def tips(func):
    def wrapper(a, b):  # 在装饰器的内部类中的定义参数
        print("start")
        print(func(a, b))
        print("end")

    return wrapper


@tips
def add(a, b):
    return a + b


add(1, 2)


####################################


# 装饰器传递装饰器的参数,如这里我们可以传递参数
def tipWithParam(param):
    def newTips(func):
        def wrapper(a, b):  # 在装饰器的内部类中的定义参数
            print("start method %s %s" % (param, func.__name__)) # 方法名的话可直接使用func.__name__来获取
            print(func(a, b))
            print("end")

        return wrapper

    return newTips


@tipWithParam("add_PARAM")  # 可传递一个参数
def newAdd(a, b):
    return a + b


newAdd(1, 2)

```

### 类

```python
# 学习使用面向对象编程


# 定义一个类,其内部对象以self来指定,所有方法的第一个参数都是self
class Person:
    # 类似构造函数,构造函数名为init
    def __init__(self, name, age):
        self.name = name
        self.__age = age  # 在属性前面加两个下划线相当于私有属性,从外部对象无法直接修改,封装的属性子类无法访问

    # 定义方法
    def info(self):
        print("My name is %s and i am %s old" % (self.name, self.__age))


tinys = Person("tinys", 24)
tinys.info()


#############################################################


# 学习类的继承,在定义类后面用一个括号来定义继承的类
class Student(Person):
    # 类似构造函数,构造函数名为init
    def __init__(self, name, age):
        super().__init__(name, age)

    def study(self):
        # 这里就无法直接访问封装了的私有属性age了
        print("My name is %s ,I am a student, I need to study" % self.name)


student = Student("black", 18)
student.info()
student.study()

#############################

# 判断类的类型
print("student的类型为%s" % type(student))

# 判断对象是否属于某个类
print(isinstance(student, Student))
print(isinstance(student, Person))


#############

# 使用with方式来定义类的异常处理
class Handler:
    # 相当于初始化时会调用的方法
    def __enter__(self):
        print("start")

    # 结束时调用的方法
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_tb is None:
            print("OK")
        else:
            print("Exception %s" % exc_tb)


with Handler():
    print("测试开始")


```

### 线程

```python
# 学习多线程
import threading


def MyThread(agr1,arg2):
    print("current thread %s" % (threading.current_thread()))
    print("%s %s" % (agr1, arg2))
    print("异步线程结束")


# 定义一个线程
thread = threading.Thread(target=MyThread, args=("tinys", 24))
# 异步启动线程
thread.start()

# 线程等待主线程
thread.join()

print("主线程结束")

```

### 时间库

```python
# 学习时间标准库

import time
import datetime

print(time.time())  # 输出时间戳
print(time.localtime())  # 输出当前时间,但非标准格式化
print(time.strftime("%Y-%m-%d %H:%M:%S"))  # 格式化当前时间,输出为"2020-06-16 21:53:28"

print("============下面为datetime的学习===============")


print(datetime.datetime.now())  # 获取当前时间,输出为"2020-06-16 21:54:44.934656"
# 时间的增加减操作
addTime = datetime.timedelta(minutes=10)
print(datetime.datetime.now() + addTime)

# 直接指定某一天
one_day = datetime.datetime(2020, 6, 16)
print(one_day)

```

### 数学库

```python
# 学习数学标准库

import random
# 获取随机数,取值范围在1-9之间
print(random.randint(1, 9))
# 在数组中随机获取一个字符串
print(random.choice(["b", "c", "d"]))

```

### 文件路径

```python
# 学习os文件库以及路径库

import os
from pathlib import Path


print("========开始学习os.path==========")

print(os.path.abspath("."))     # 打印当前目录
print(os.path.exists("f:/py"))  # 判断文件/文件夹是否存在
print(os.path.isdir("f://py"))  # 判断是否文件夹

print("========开始学习Path==========")

p = Path(".")   # 当前路径
print(p.resolve())
print(p.is_dir())

# 创建文件夹,并且父目录不存在时也创建其父目录
#dirPath = Path("f://py//test")
#Path.mkdir(dirPath, parents=True)

```

### 正则

```python
# 学习正则表达式的使用

import re


# 常见正则符号 : ., ^, *, +, ?, {m},{m,n},[], \, \d, \D,\s ()
# \D表示非数字
# ^$ 表示空行
# .*? 表示懒惰匹配,尽可能少的匹配,即一旦符合匹配条件即停止不再往下匹配(如.*就会匹配剩下全部的,默认是贪婪匹配)

regex = re.compile("ab*c")
print(regex.match("abbbbc"))


# 学习group
numCompile = re.compile(r"(\d+)-(\d+)-(\d+)")  # r表示原样输出,不转义(如\n换行此时就不换行而是输出"\n"字符串)
print(numCompile.match("2020-06-20").group(0))  # 使用组概念获取到正则的整个值,注意下标为0开始表示整个值的匹配,此处为2020-06-20,默认不加也是取出全部值
print(numCompile.match("2020-06-20").group(1))  # 使用组概念获取到正则的第一个组,注意下标从1开始,此处为2020
print(numCompile.match("2020-06-20").group(3))  # 使用组概念获取到正则的第三个组,注意下标从1开始,此处为20

# 使用groups方法直接赋值
year, month, day = numCompile.match("2020-06-20").groups()
print(year)


# 正则的search方法
print(numCompile.search("aaa2020-05-06bbb"))    # 使用search来搜索,即使第一个字符不匹配也会往下进行匹配搜索

# 正则的sub替换方法,直接将后面的注释替换掉
message = "2020-06-21 # 哪一天在干嘛"
subMessage = re.sub(r"#.*$", "", message)
print(subMessage)

```

