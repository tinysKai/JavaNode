##  Python核心技术与实践 - 函数篇

### 函数定义

```python
def name(param1, param2, ..., paramN):
    statements
    return/yield value # optional
```

需要注意，主程序调用函数时，必须保证这个函数此前已经定义过，不然就会报错.

```python
my_func('hello world')
def my_func(message):
    print('Got a message: {}'.format(message))
    
# 输出
NameError: name 'my_func' is not defined
```

但是，如果我们在函数内部调用其他函数，函数间哪个声明在前、哪个在后就无所谓，因为 def 是可执行语句，函数在调用之前都不存在，我们只需保证调用时，所需的函数都已经声明定义

```python
def my_func(message):
    my_sub_func(message) # 调用my_sub_func()在其声明之前不影响程序执行
    
def my_sub_func(message):
    print('Got a message: {}'.format(message))

my_func('hello world')

# 输出
Got a message: hello world
```

### 函数默认值

Python 函数的参数可以设定默认值，比如下面这样的写法：

```python
def func(param = 0):
    ...
```

在调用函数 func() 时，如果参数 param 没有传入，则参数默认为 0；而如果传入了参数 param，其就会覆盖默认值。

### 参数多态/无类型

Python 不用考虑输入的数据类型，而是将其交给具体的代码去判断执行，同样的一个函数，可以同时应用在整型、列表、字符串等等的操作中。当然，如果两个参数的数据类型不同，比如一个是列表、一个是字符串，两者无法相加，那就会报错。如:

```python
def my_sum(a, b):
    return a + b

result = my_sum(3, 5)
print(result)

# 输出
8

# 参数是列表
print(my_sum([1, 2], [3, 4]))

# 输出
[1, 2, 3, 4]


# 参数是字符串
print(my_sum('hello ', 'world'))

# 输出
hello world
```

在编程语言中，我们把这种行为称为多态。这也是 Python 和其他语言，比如 Java、C 等很大的一个不同点。当然，Python 这种方便的特性，在实际使用中也会带来诸多问题。因此，必要时请你在开头加上数据的类型检查。

### 函数嵌套

Python 支持函数的嵌套。所谓的函数嵌套，就是指函数里面又有函数 

函数的嵌套，主要有下面两个方面的作用。

第一，函数的嵌套能够保证内部函数的隐私。内部函数只能被外部函数所调用和访问，不会暴露在全局作用域，因此，如果你的函数内部有一些隐私数据（比如数据库的用户、密码等），不想暴露在外，那你就可以使用函数的的嵌套，将其封装在内部函数中，只通过外部函数来访问。比如：

```python
def connect_DB():
    def get_DB_configuration():
        ...
        return host, username, password
    conn = connector.connect(get_DB_configuration())
    return conn
```

这里的函数 get_DB_configuration，便是内部函数，它无法在 connect_DB() 函数以外被单独调用。也就是说，下面这样的外部直接调用是错误的：

第二，合理的使用函数嵌套，能够提高程序的运行效率。我们来看下面这个例子：

```python
def factorial(input):
    # validation check
    if not isinstance(input, int):
        raise Exception('input must be an integer.')
    if input < 0:
        raise Exception('input must be greater or equal to 0' )
    ...

    def inner_factorial(input):
        if input <= 1:
            return 1
        return input * inner_factorial(input-1)
    return inner_factorial(input)


print(factorial(5))
```

这里，我们使用递归的方式计算一个数的阶乘。因为在计算之前，需要检查输入是否合法，所以我写成了函数嵌套的形式，这样一来，输入是否合法就只用检查一次。而如果我们不使用函数嵌套，那么每调用一次递归便会检查一次，这是没有必要的，也会降低程序的运行效率。

### 函数变量作用域

Python 函数中变量的作用域和其他语言类似。如果变量是在函数内部定义的，就称为`局部变量`，只在函数内部有效。一旦函数执行完毕，局部变量就会被回收，无法访问

相对应的，全局变量则是定义在整个文件层次上的,我们不能在函数内部随意改变全局变量的值。比如，下面的写法就是错误的：

```python
MIN_VALUE = 1
MAX_VALUE = 10
def validation_check(value):
    ...
    MIN_VALUE += 1
    ...
validation_check(5)

# UnboundLocalError: local variable 'MIN_VALUE' referenced before assignment
```

这是因为，Python 的解释器会默认函数内部的变量为局部变量，但是又发现局部变量 MIN_VALUE 并没有声明，因此就无法执行相关操作。所以，如果我们一定要在函数内部改变全局变量的值，就必须加上 global 这个声明：

```python
MIN_VALUE = 1
MAX_VALUE = 10
def validation_check(value):
    global MIN_VALUE
    ...
    MIN_VALUE += 1
    ...
validation_check(5)
```

这里的 global 关键字，并不表示重新创建了一个全局变量 MIN_VALUE，而是告诉 Python 解释器，函数内部的变量 MIN_VALUE，就是之前定义的全局变量，并不是新的全局变量，也不是局部变量。这样，程序就可以在函数内部访问全局变量，并修改它的值了。

> 另外，如果遇到函数内部局部变量和全局变量同名的情况，那么在函数内部，局部变量会覆盖全局变量，比如下面这种：

类似的，对于嵌套函数来说，内部函数可以访问外部函数定义的变量，但是无法修改，若要修改，必须加上 nonlocal 这个关键字：

```python
def outer():
    x = "local"
    def inner():
        nonlocal x # nonlocal关键字表示这里的x就是外部函数outer定义的变量x
        x = 'nonlocal'
        print("inner:", x)
    inner()
    print("outer:", x)
outer()
# 输出
inner: nonlocal
outer: nonlocal
```

### 闭包

闭包其实和刚刚讲的嵌套函数类似，不同的是，这里外部函数返回的是一个函数，而不是一个具体的值。返回的函数通常赋于一个变量，这个变量可以在后面被继续执行调用。

```python
def nth_power(exponent):
    def exponent_of(base):
        return base ** exponent
    return exponent_of # 返回值是exponent_of函数

square = nth_power(2) # 计算一个数的平方
cube = nth_power(3) # 计算一个数的立方 
square
# 输出
<function __main__.nth_power.<locals>.exponent(base)>

cube
# 输出
<function __main__.nth_power.<locals>.exponent(base)>

print(square(2))  # 计算2的平方
print(cube(2)) # 计算2的立方
# 输出
4 # 2^2
8 # 2^3
```

### 匿名函数 - lambda

#### 基本格式

```python
lambda argument1, argument2,... argumentN : expression
```

#### 例子

```python
square = lambda x: x**2
square(3)

# 输出9


[(lambda x: x*x)(x) for x in range(10)]
# 输出
[0, 1, 4, 9, 16, 25, 36, 49, 64, 81]



l = [(1, 20), (3, 0), (9, 10), (2, -1)]
l.sort(key=lambda x: x[1]) # 按列表中元组的第二个元素排序
print(l)
# 输出
[(2, -1), (3, 0), (9, 10), (1, 20)]
```

#### 使用匿名函数的作用

使用匿名函数 lambda，可以帮助我们大大简化代码的复杂度，提高代码的可读性。

```python
# 使用匿名函数来对一个列表来做指数计算
squared = map(lambda x: x**2, [1, 2, 3, 4, 5])

# 使用常规函数
def square(x):
    return x**2

squared = map(square, [1, 2, 3, 4, 5])

# 函数 map(function, iterable) 的第一个参数是函数对象，第二个参数是一个可以遍历的集合，
# 它表示对 iterable 的每一个元素，都运用 function 这个函数。
```

### Python的函数式编程

所谓函数式编程，是指代码中每一块都是不可变的（immutable），都由纯函数（pure function）的形式组成。这里的纯函数，是指函数本身相互独立、互不影响，对于相同的输入，总会有相同的输出，没有任何副作用。

函数式编程的优点，主要在于其纯函数和不可变的特性使程序更加健壮，易于调试（debug）和测试；缺点主要在于限制多，难写。当然，Python 不同于一些语言（比如 Scala），它并不是一门函数式编程语言，不过，Python 也提供了一些函数式编程的特性，值得我们了解和学习。

Python 主要提供了这么几个函数：map()、filter() 和 reduce()，通常结合匿名函数 lambda 一起使用。

```python
# map(function, iterable) 函数
# map 对 iterable 中的每个元素，都运用 function 这个函数，最后返回一个新的可遍历的集合。
l = [1, 2, 3, 4, 5]
new_list = map(lambda x: x * 2, l) # [2， 4， 6， 8， 10]

# filter(function, iterable) 函数
# filter 对 iterable 中的每个元素，都使用 function 判断，并返回 True 或者 False，
# 最后将返回 True 的元素组成一个新的可遍历的集合。
l = [1, 2, 3, 4, 5]
new_list = filter(lambda x: x % 2 == 0, l) # [2, 4]

# reduce(function, iterable) 函数 
# function 同样是一个函数对象，规定它有两个参数，表示对 iterable 中的每个元素以及上一次调用后的结果，
# 运用 function 进行计算，所以最后返回的是一个单独的数值。
l = [1, 2, 3, 4, 5]
product = reduce(lambda x, y: x * y, l) # 1*2*3*4*5 = 120
```



### 总结

+ Python 中函数的参数可以接受任意的数据类型，使用起来需要注意，必要时请在函数开头加入数据类型的检查；
+ 和其他语言不同，Python 中函数的参数可以设定默认值；
+ 嵌套函数的使用，能保证数据的隐私性，提高程序运行效率；
+ 合理地使用闭包，则可以简化程序的复杂度，提高可读性。