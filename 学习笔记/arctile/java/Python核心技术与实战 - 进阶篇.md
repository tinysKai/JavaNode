## Python核心技术与实战 - 进阶篇

### 对象的比较

'=='操作符比较对象之间的值是否相等,而'is'操作符比较的是对象的身份标识是否相等，即它们是否是同一个对象，是否指向同一个内存地址。

### 对象的拷贝

#### 概念

浅拷贝，是指重新分配一块内存，创建一个新的对象，里面的元素是原对象中子对象的引用。

所谓深度拷贝，是指重新分配一块内存，创建一个新的对象，并且将原对象中的元素，以递归的方式，通过创建新的子对象拷贝到新对象中。因此，新对象和原对象没有任何关联。

#### 常见的浅拷贝

```python
l1 = [1, 2, 3]
l2 = list(l1)

l2
[1, 2, 3]

l1 == l2
True

l1 is l2
False

s1 = set([1, 2, 3])
s2 = set(s1)

s2
{1, 2, 3}

s1 == s2
True

s1 is s2
False

# l2 就是 l1 的浅拷贝，s2 是 s1 的浅拷贝

# 对于可变的序列，我们还可以通过切片操作符':'完成浅拷贝
l1 = [1, 2, 3]
l2 = l1[:]

l1 == l2
True

l1 is l2
False
```

需要注意的是，对于元组，使用 tuple() 或者切片操作符':'不会创建一份浅拷贝，相反，它会返回一个指向相同元组的引用：

```python
t1 = (1, 2, 3)
t2 = tuple(t1)

t1 == t2
True

t1 is t2
True

# 元组 (1, 2, 3) 只被创建一次，t1 和 t2 同时指向这个元组。
```

#### 深拷贝

```python
import copy
l1 = [[1, 2], (30, 40)]
l2 = copy.deepcopy(l1)
l1.append(100)
l1[0].append(3)

l1
[[1, 2, 3], (30, 40), 100]

l2 
[[1, 2], (30, 40)]

# 无论 l1 如何变化，l2 都不变。因为此时的 l1 和 l2 完全独立，没有任何联系。
```

### 赋值传递/引用传递

Python 的参数传递是赋值传递 （pass by assignment），或者叫作对象的引用传递（pass by object reference）。Python 里所有的数据类型都是对象，所以参数传递时，只是让新变量与原变量指向相同的对象而已，并不存在值传递或是引用传递一说。

> 需要注意的是，这里的赋值或对象的引用传递，不是指向一个具体的内存地址，而是指向一个具体的对象。

#### 例子1

这里的参数传递，使变量 a 和 b 同时指向了 1 这个对象。但当我们执行到 b = 2 时，系统会重新创建一个值为 2 的新对象，并让 b 指向它；而 a 仍然指向 1 这个对象。所以，a 的值不变，仍然为 1。

```python
def my_func1(b):
  b = 2

a = 1
my_func1(a)
a
1
```

#### 例子2

当可变对象当作参数传入函数里的时候，改变可变对象的值，就会影响所有指向它的变量。

```python
def my_func3(l2):
  l2.append(4)

l1 = [1, 2, 3]
my_func3(l1)
l1
[1, 2, 3, 4]
```

#### 例子3

第一个方法中的`l2 = l2 + [4]`，表示创建了一个“末尾加入元素 4“的新列表，并让 l2 指向这个新的对象。这个过程与 l1 无关，因此 l1 的值不变。

如果要改变 l1 的值，我们就得让上述函数返回一个新列表，再赋予 l1 即可,如方法二

```python
def my_func4(l2):
  l2 = l2 + [4]

l1 = [1, 2, 3]
my_func4(l1)
l1
[1, 2, 3]



def my_func5(l2):
  l2 = l2 + [4]
  return l2

l1 = [1, 2, 3]
l1 = my_func5(l1)
l1
[1, 2, 3, 4]
```

### 装饰器

#### 无参数的装饰器

```python
def my_decorator(func):
    def wrapper():
        print('wrapper of decorator')
        func()
    return wrapper

def greet():
    print('hello world')

greet = my_decorator(greet)
greet()

# 输出
wrapper of decorator
hello world
```

等价于下面的代码,这里的@，我们称之为语法糖，@my_decorator就相当于前面的greet=my_decorator(greet)语句，只不过更加简洁。

```python
def my_decorator(func):
    def wrapper():
        print('wrapper of decorator')
        func()
    return wrapper

@my_decorator
def greet():
    print('hello world')

greet()
```

#### 有参数的装饰器

简单的办法,在wrapper上传参数,这种方式如果需要用在其他方法上,但其参数是多个的话则无法复用该装饰器

```python
def my_decorator(func):
    def wrapper(message):
        print('wrapper of decorator')
        func(message)
    return wrapper


@my_decorator
def greet(message):
    print(message)


greet('hello world')

# 输出
wrapper of decorator
hello world
```

接收任意类型和数量的装饰器

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print('wrapper of decorator')
        func(*args, **kwargs)
    return wrapper

# 关于*args, **kwargs的解释 
# *agrs 主要作用是用来匹配任意个参数
# **kwargs的作用是匹配任意个的键值对,如下
def greet_me(**kwargs):
    for key, value in kwargs.items():
        print("{0} = {1}".format(key, value))

>>> greet_me(name="yasoob")
name = yasoob
```

#### 类装饰器

类装饰器主要依赖于函数__call__()，每当你调用一个类的示例时，函数__call__()就会被执行一次。

```python
class Count:
    def __init__(self, func):
        self.func = func
        self.num_calls = 0

    def __call__(self, *args, **kwargs):
        self.num_calls += 1
        print('num of calls is: {}'.format(self.num_calls))
        return self.func(*args, **kwargs)

@Count
def example():
    print("hello world")

example()

# 输出
num of calls is: 1
hello world

example()

# 输出
num of calls is: 2
hello world

...
```

#### 装饰器业务使用例子

@functools.wrap，它会帮助保留原函数的元信息（也就是将原函数的元信息，拷贝到对应的装饰器函数里）。

```python
# 身份认证
import functools

def authenticate(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        request = args[0]
        if check_user_logged_in(request): # 如果用户处于登录状态
            return func(*args, **kwargs) # 执行函数post_comment() 
        else:
            raise Exception('Authentication failed')
    return wrapper
    
@authenticate
def post_comment(request, ...)
    ...
 
```

```python
# 日志记录
import time
import functools

def log_execution_time(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        res = func(*args, **kwargs)
        end = time.perf_counter()
        print('{} took {} ms'.format(func.__name__, (end - start) * 1000))
        return res
    return wrapper
    
@log_execution_time
def calculate_similarity(items):
    ...
```

```python
# 参数合法性检查
import functools

def validation_check(input):
    @functools.wraps(func)
    def wrapper(*args, **kwargs): 
        ... # 检查输入是否合法
    
@validation_check
def neural_network_training(param1, param2, ...):
    ...
```

### 生成器

生成器在你调用 next() 函数的时候，才会生成下一个变量。生成器在 Python 的写法是用小括号括起来，(i for i in range(100000000))，即初始化了一个生成器。**生成器是懒人版本的迭代器。**

```python
import os
import psutil

# 显示当前 python 程序占用的内存大小
def show_memory_info(hint):
    pid = os.getpid()
    p = psutil.Process(pid)
    
    info = p.memory_full_info()
    memory = info.uss / 1024. / 1024
    print('{} memory used: {} MB'.format(hint, memory))
```

如下运行结果可以清晰地看到，生成器并不会像迭代器一样占用大量内存，只有在被使用的时候才会调用。而且生成器在初始化的时候，并不需要运行一次生成操作，相比于 test_iterator() ，test_generator() 函数节省了一次生成一亿个元素的过程，因此耗时明显比迭代器短。

```python

def test_iterator():
    show_memory_info('initing iterator')
    list_1 = [i for i in range(100000000)]
    show_memory_info('after iterator initiated')
    print(sum(list_1))
    show_memory_info('after sum called')

def test_generator():
    show_memory_info('initing generator')
    list_2 = (i for i in range(100000000))
    show_memory_info('after generator initiated')
    print(sum(list_2))
    show_memory_info('after sum called')

%time test_iterator()
%time test_generator()

########## 输出 ##########

initing iterator memory used: 48.9765625 MB
after iterator initiated memory used: 3920.30078125 MB
4999999950000000
after sum called memory used: 3920.3046875 MB
Wall time: 17 s
initing generator memory used: 50.359375 MB
after generator initiated memory used: 50.359375 MB
4999999950000000
after sum called memory used: 50.109375 MB
Wall time: 12.5 s
```

### 并发&并行

+ 并发通常应用于 I/O 操作频繁的场景，比如你要从网站上下载多个文件，I/O 操作的时间可能会比 CPU 运行处理的时间长得多。
+ 并行则更多应用于 CPU heavy 的场景，比如 MapReduce 中的并行计算，为了加快运行速度，一般会用多台机器、多个处理器来完成。

Python 中之所以同一时刻只允许一个线程运行，其实是由于全局解释器锁的存在。但是对 I/O 操作而言，当其被 block 的时候，全局解释器锁便会被释放，使其他线程继续执行。

单线程的爬虫方式

```python
import requests
import time

def download_one(url):
    resp = requests.get(url)
    print('Read {} from {}'.format(len(resp.content), url))
    
def download_all(sites):
    for site in sites:
        download_one(site)

def main():
    sites = [
        'https://en.wikipedia.org/wiki/Portal:Arts',
        'https://en.wikipedia.org/wiki/Portal:History',
        'https://en.wikipedia.org/wiki/Portal:Society',
        'https://en.wikipedia.org/wiki/Portal:Biography',
        'https://en.wikipedia.org/wiki/Portal:Mathematics',
        'https://en.wikipedia.org/wiki/Portal:Technology',
        'https://en.wikipedia.org/wiki/Portal:Geography',
        'https://en.wikipedia.org/wiki/Portal:Science',
        'https://en.wikipedia.org/wiki/Computer_science',
        'https://en.wikipedia.org/wiki/Python_(programming_language)',
        'https://en.wikipedia.org/wiki/Java_(programming_language)',
        'https://en.wikipedia.org/wiki/PHP',
        'https://en.wikipedia.org/wiki/Node.js',
        'https://en.wikipedia.org/wiki/The_C_Programming_Language',
        'https://en.wikipedia.org/wiki/Go_(programming_language)'
    ]
    start_time = time.perf_counter()
    download_all(sites)
    end_time = time.perf_counter()
    print('Download {} sites in {} seconds'.format(len(sites), end_time - start_time))
    
if __name__ == '__main__':
    main()

# 输出
Read 129886 from https://en.wikipedia.org/wiki/Portal:Arts
Read 184343 from https://en.wikipedia.org/wiki/Portal:History
Read 224118 from https://en.wikipedia.org/wiki/Portal:Society
Read 107637 from https://en.wikipedia.org/wiki/Portal:Biography
Read 151021 from https://en.wikipedia.org/wiki/Portal:Mathematics
Read 157811 from https://en.wikipedia.org/wiki/Portal:Technology
Read 167923 from https://en.wikipedia.org/wiki/Portal:Geography
Read 93347 from https://en.wikipedia.org/wiki/Portal:Science
Read 321352 from https://en.wikipedia.org/wiki/Computer_science
Read 391905 from https://en.wikipedia.org/wiki/Python_(programming_language)
Read 321417 from https://en.wikipedia.org/wiki/Java_(programming_language)
Read 468461 from https://en.wikipedia.org/wiki/PHP
Read 180298 from https://en.wikipedia.org/wiki/Node.js
Read 56765 from https://en.wikipedia.org/wiki/The_C_Programming_Language
Read 324039 from https://en.wikipedia.org/wiki/Go_(programming_language)
Download 15 sites in 2.464231112999869 seconds
```

并发式爬虫

```python
import concurrent.futures
import requests
import threading
import time

def download_one(url):
    resp = requests.get(url)
    print('Read {} from {}'.format(len(resp.content), url))

# 重点在这里,这里使用了线程池的方式来执行任务,最大的线程数为5,最终耗时为0.2s左右
def download_all(sites):
    with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
        executor.map(download_one, sites)

def main():
    sites = [
        'https://en.wikipedia.org/wiki/Portal:Arts',
        'https://en.wikipedia.org/wiki/Portal:History',
        'https://en.wikipedia.org/wiki/Portal:Society',
        'https://en.wikipedia.org/wiki/Portal:Biography',
        'https://en.wikipedia.org/wiki/Portal:Mathematics',
        'https://en.wikipedia.org/wiki/Portal:Technology',
        'https://en.wikipedia.org/wiki/Portal:Geography',
        'https://en.wikipedia.org/wiki/Portal:Science',
        'https://en.wikipedia.org/wiki/Computer_science',
        'https://en.wikipedia.org/wiki/Python_(programming_language)',
        'https://en.wikipedia.org/wiki/Java_(programming_language)',
        'https://en.wikipedia.org/wiki/PHP',
        'https://en.wikipedia.org/wiki/Node.js',
        'https://en.wikipedia.org/wiki/The_C_Programming_Language',
        'https://en.wikipedia.org/wiki/Go_(programming_language)'
    ]
    start_time = time.perf_counter()
    download_all(sites)
    end_time = time.perf_counter()
    print('Download {} sites in {} seconds'.format(len(sites), end_time - start_time))

if __name__ == '__main__':
    main()

## 输出
Read 151021 from https://en.wikipedia.org/wiki/Portal:Mathematics
Read 129886 from https://en.wikipedia.org/wiki/Portal:Arts
Read 107637 from https://en.wikipedia.org/wiki/Portal:Biography
Read 224118 from https://en.wikipedia.org/wiki/Portal:Society
Read 184343 from https://en.wikipedia.org/wiki/Portal:History
Read 167923 from https://en.wikipedia.org/wiki/Portal:Geography
Read 157811 from https://en.wikipedia.org/wiki/Portal:Technology
Read 91533 from https://en.wikipedia.org/wiki/Portal:Science
Read 321352 from https://en.wikipedia.org/wiki/Computer_science
Read 391905 from https://en.wikipedia.org/wiki/Python_(programming_language)
Read 180298 from https://en.wikipedia.org/wiki/Node.js
Read 56765 from https://en.wikipedia.org/wiki/The_C_Programming_Language
Read 468461 from https://en.wikipedia.org/wiki/PHP
Read 321417 from https://en.wikipedia.org/wiki/Java_(programming_language)
Read 324039 from https://en.wikipedia.org/wiki/Go_(programming_language)
Download 15 sites in 0.19936635800002023 seconds
```

并行方式

```python
# 将上述的线程替换为进程的模式,表示创建进程池
with futures.ThreadPoolExecutor(workers) as executor
=>
with futures.ProcessPoolExecutor() as executor: 
```

原始并发代码调用

```python

import concurrent.futures
import requests
import time

def download_one(url):
    resp = requests.get(url)
    print('Read {} from {}'.format(len(resp.content), url))

# 使用了submit提交任务,as_completed来监测任务完成了没    
def download_all(sites):
    with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
        to_do = []
        for site in sites:
            future = executor.submit(download_one, site)
            to_do.append(future)
            
        for future in concurrent.futures.as_completed(to_do):
            future.result()
def main():
    sites = [
        'https://en.wikipedia.org/wiki/Portal:Arts',
        'https://en.wikipedia.org/wiki/Portal:History',
        'https://en.wikipedia.org/wiki/Portal:Society',
        'https://en.wikipedia.org/wiki/Portal:Biography',
        'https://en.wikipedia.org/wiki/Portal:Mathematics',
        'https://en.wikipedia.org/wiki/Portal:Technology',
        'https://en.wikipedia.org/wiki/Portal:Geography',
        'https://en.wikipedia.org/wiki/Portal:Science',
        'https://en.wikipedia.org/wiki/Computer_science',
        'https://en.wikipedia.org/wiki/Python_(programming_language)',
        'https://en.wikipedia.org/wiki/Java_(programming_language)',
        'https://en.wikipedia.org/wiki/PHP',
        'https://en.wikipedia.org/wiki/Node.js',
        'https://en.wikipedia.org/wiki/The_C_Programming_Language',
        'https://en.wikipedia.org/wiki/Go_(programming_language)'
    ]
    start_time = time.perf_counter()
    download_all(sites)
    end_time = time.perf_counter()
    print('Download {} sites in {} seconds'.format(len(sites), end_time - start_time))

if __name__ == '__main__':
    main()

# 输出
Read 129886 from https://en.wikipedia.org/wiki/Portal:Arts
Read 107634 from https://en.wikipedia.org/wiki/Portal:Biography
Read 224118 from https://en.wikipedia.org/wiki/Portal:Society
Read 158984 from https://en.wikipedia.org/wiki/Portal:Mathematics
Read 184343 from https://en.wikipedia.org/wiki/Portal:History
Read 157949 from https://en.wikipedia.org/wiki/Portal:Technology
Read 167923 from https://en.wikipedia.org/wiki/Portal:Geography
Read 94228 from https://en.wikipedia.org/wiki/Portal:Science
Read 391905 from https://en.wikipedia.org/wiki/Python_(programming_language)
Read 321352 from https://en.wikipedia.org/wiki/Computer_science
Read 180298 from https://en.wikipedia.org/wiki/Node.js
Read 321417 from https://en.wikipedia.org/wiki/Java_(programming_language)
Read 468421 from https://en.wikipedia.org/wiki/PHP
Read 56765 from https://en.wikipedia.org/wiki/The_C_Programming_Language
Read 324039 from https://en.wikipedia.org/wiki/Go_(programming_language)
Download 15 sites in 0.21698231499976828 seconds
```

