## Linux常用命令

#### find命令

```shell
#普通的查找指定目录
   find  /dir
 
# 查找某目录下指定的文件名
   find /dir -name "abc.txt"
   find /dir -name "*.txt"  #可以使用通配符来过滤  
   find /dir -iname "abc.txt" #忽略大小写的查找
   
#限制目录查找的深度
   find /dir -maxdepth 2 -name "*.txt" #表示查找当前文件夹递归查找的深度为2,如果为1表示只在当前目前查找
   
# 反向查找
   find /dir -not -name "*.txt"  #查找所有非txt文件名结尾的文件
   
# 复合查找(And)
   find /dir -name "pro*" -not -name "*.txt" #查找所有以pro开头但不是以txt结尾的文件
   
# 复合操作(OR)
   find /dir -name "*.sh" -o -name "*.log"  #查找脚本或者是日志的文件 
   
# 指定搜索类型
   find ./dir -type f -name "asd*" #只查找asd开头的文件
   find ./dir -type d -name "asd*" #只查找asd开头的文件目录
   
# 查找在指定大小的文件
   find /dir -size +50M -size -100M
```



#### history命令

![http://ww1.sinaimg.cn/large/8bb38904ly1g4i1d7pg3mj20ge07g3yh.jpg](https://s2.ax1x.com/2019/06/29/ZQ7Qne.png)

#### grep命令

```shell
grep [-A] [-B] [-color] '搜索字符串' filename
grep -w word file # -w选项指定要搜索的单词

#超有用-搜索在文件中包含某字符串的文件名 
grep -l "string" filename

#在一个文件夹中搜索关键字,-r表示递归遍历folderPath其下的全部文件
grep ${keyword} -r ${folderPath}
```



#### SCP命令

```shell
scp local_file remote_username@remote_ip:remote_folder
scp -r local_folder remote_username@remote_ip:remote_folder
```



#### 通过进程号查找对应的启动路径

![http://ww1.sinaimg.cn/large/8bb38904ly1g4i1ixonyuj20zk09qwes.jpg](https://s2.ax1x.com/2019/06/29/ZQ7fuF.png)

`ps -ef |grep redis | grep -v grep | awk '{print "/proc/"$2}' | xargs ls -al | grep exe `

![http://ww1.sinaimg.cn/large/8bb38904ly1g4i1sxe3n3j216304rglo.jpg](https://s2.ax1x.com/2019/06/29/ZQH0xK.png)



#### 找出当前系统cpu使用量较高的进程

`ps -ef | sort -rnk 4 |head -10`
sort解释:-n是按照数字大小排序，-r是以相反顺序，-k是指定需要爱排序的栏位，-t可用于指定栏位分隔符



#### 查看进程的所有变量

`strings /proc/进程号/environ`



#### 查找某进程下消耗cpu最好的线程

` top -p 进程号 按下"shift + h"`

#### 查看环境变量

`cat /etc/profile | cat ~/.profile | cat ~/.bashrc`

