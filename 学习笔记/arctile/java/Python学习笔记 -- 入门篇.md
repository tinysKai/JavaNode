# Python学习笔记  -- 入门篇

## 基本类型

### 字符串

```python
# 学习py字符串的基本常见方法


message = "hello world"
# 使用title方法来使首字母大写
print(message.title())  # 输出 Hello World
print(message.upper())  # 输出 HELLO WORLD
print(message.lower())  # 输出 hello world

print("==========字符串拼接==========")

# 字符串拼接
firstName = "tinys"
lastName = "huang"
fullName = firstName + "." + lastName
print(fullName)


print("==========字符串前后空白处理==========")
trimMessage = " message "
print(trimMessage)
print(trimMessage.rstrip())     # 去除尾部的空白字符串
print(trimMessage.lstrip())     # 去除头部的空白字符串
print(trimMessage.strip())      # 去除两端的空白字符串

```

### 数字

```python
# 学习数字类型

print(1 + 2)
print(2 * 3)

# 数字的类型转换为字符串
age = 24
message = str(age) + " years old"
print(message)

print("\n======使用range来定义数组列表========")
for i in range(1, 5):       # range的范围为左包括右不包括(左闭右开)
    print(i)

print("步长为2的range")
for i in range(0, 11, 2):       # 步长为2
    print(i)


print("使用range来赋值列表")
nums = list(range(0, 6))
print(nums)


print("\n======数字计算方法========")
print(max(nums))
print(min(nums))
print(sum(nums))


print("\n=======列表推导式========")
squares = [i * i for i in range(1, 6) if(i % 2 == 0)]   # 列表推导式有三块,最左边为执行块,中间为循坏块,最右边为条件模块
print(squares)
```

### 列表

```python
# 学习列表

names = ["刘备", "关羽", "张飞"]
print(names)

# 获取列表第一个元素
print(names[0])
print(names[-1])    # -1表示从列表尾部开始倒数第一个
print(names[-2])    # -1表示从列表尾部开始倒数第二个


print("\n======遍历列表=========")
# 遍历列表
for name in names:
    print(name)

print("\n=========列表元素添加方法================")
print("append方法在列表尾部追加元素")
names.append("赵云")  # append方法在列表尾部追加元素
print(names)
print("insert方法将元素添加到指定的下标")
names.insert(0, "孔明")
names.insert(4, "黄忠")   # 使用insert方法来添加元素
print(names)


print("\n=========列表元素删除方法================")
print("del方法来删除元素")
del names[4]    # 使用del关键字来删除元素
print(names)
print("pop方法来弹出尾部的元素")
popName = names.pop()
print(popName)
popFirstNum = names.pop(0)  # 弹出列表第一个元素
print(popFirstNum)
print(names)
print("remove方法根据值来删除元素")
names.remove("关羽")  # 注意remove方法只会删除第一个找到的元素
print(names)


print("\n=========列表排序方法================")
print("使用sort方法对列表进行永久性排序")
nums = ["y", "z", "d", "a", "u"]
nums.sort()
print(nums)
nums.sort(reverse=True)     # 逆序排序
print(nums)


print("sorted临时性排序")
nums = ["y", "z", "d", "a", "u"]
print(sorted(nums))
print(nums)
print("reverse反转列表")
nums = ["y", "z", "d", "a", "u"]
nums.reverse()
print(nums)


print("\n=================len方法获取列表元素个数===========")
print(len(nums))


print("\n=================列表切片===========")
print(nums[0:3])
print(nums[:3])     # 没指定列表开头则从第一个元素开始
print(nums[1:])     # 没指定列表结尾则遍历到最后一个元素
print(nums[-3:])     # 输出列表中最后三个元素


print("\n=================复制列表===========")
copyNum = nums[:]
print(copyNum)

```

### 元组

```python
# 学习元组,不可变的列表称为元组

nums = (1, 5)
print(nums)

# 遍历元组
for num in nums:
    print(num)

```

### 字典

```python
# 学习字典

person = {"name": "tinys", "age": 24}
print(person["name"] + "is " + str(person["age"]) + " years old")

print("\n=========添加元素==========")
person["country"] = "中国"
print(person)


print("\n=========删除元素==========")
del person["age"]
print(person)


print("\n=========遍历字典==========")
for key, val in person.items():
    print("key is %s val is %s " % (key, val))

print("只遍历key值")
for key in person.keys():
    print("key is %s " % key)

print("只遍历val值")
for val in person.values():
    print("val is %s " % val)


print("\n=========字典嵌套列表==========")
person = {
    "name": "kai",
    "friend": ["刘备", "关羽"]      # 在字典中定义一个key的值为一个列表
}
for val in person["friend"]:
    print(val)


print("\n=========字典嵌套字典==========")
relation = {
    "name": "名字",
    "father": "爸爸",
    "mother": "妈妈",
    "friend": {
        "name": "朋友",
        "father": "朋友的爸爸",
        "mother": "朋友的妈妈"
    }
}
print(relation)
```

## 流程定义

### 判断

```python
# 学习判断语句
import random

num = random.randint(1, 10)
if num > 5:
    print("bigger %s" % num)
else:
    print("smaller %s" % num)

print("'!=' 不相等的判断")
name = "tinys"
if name != "kai":
    print("不相等")


print("\n=============组合判断=============")
num = random.randint(1, 10)
if num > 5 and num % 2 == 0:
    print("bigger and even %s" % num)
elif num <= 5 and num % 2 == 0:
    print("smaller and even %s" % num)
else:
    print("other %s" % num)

print("or组合")
num = random.randint(1, 10)
if num > 5 or num % 2 == 0:
    print("大于5或者是偶数 %s" % num)

print("\n=======in判断======")
nums = list(range(1, 5))    # range不包含右括号的内容
print(nums)
print(5 in nums)
print(5 not in nums)


print("\n=======列表判断======")
emptyList = []
if emptyList:       # 非空列表则返回true
    print("非空列表")
else:
    print("空列表")

```

### 用户输入

```python
# 学习用户输入
import random

num = input("输入数字:\n")
val = random.randint(0, 5)
if val == int(num):     # 注意输入的num为字符串,需转换为数字类型再比较,否则会因为类型不一样而一直不相等
    print("bingo")
else:
    print("wrong,val is %s" % str(val))

```

### 循环

```python
# 学习循环
import random

print("while循环")
num = 0
while num < 5:
    print(num)
    num += 1

print("使用break关键字")
while True:
    val = random.randint(0, 5)
    if val == 2:    # 当值是2的时候直接跳出循环
        break
    if val == 3:    # 当值是3的时候下面最后一句的输出值会忽略
        print("val is 3")
        continue
    print(val)


print("\nfor循环是一种遍历列表的有效方式，但在for循环中不应修改列表，否则将导致Python难以跟踪其中的元素。要在遍历列表的同时对其进行修改，可使用while循环。")
readyPersons = ["刘备", "关羽", "张飞"]
fightPersons = []
while readyPersons:
    person = readyPersons.pop()
    print("%s is ready fine,go to fight" % person)
    fightPersons.append(person)

print(fightPersons)


print("\n使用while循环来多次删除相同元素")
readyPersons = ["刘备", "关羽", "张飞", "刘备"]
while "刘备" in readyPersons:
    readyPersons.remove("刘备")

print(readyPersons)
```

