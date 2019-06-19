>查看linux机器的ip : ifconfig

>文件上传下载 : rz,sz

>Centos查看端口占用情况命令，比如查看80端口占用情况使用如下命令：
  
    lsof -i tcp:80

>查看磁盘空间  : df -h

>查看内存使用情况
```
free -m
注意查看内存得看-/+ buffer cache 这一行
```


>查找的find命令
```
普通的查找指定目录
   find  /dir
 
查找某目录下指定的文件名
   find /dir -name "abc.txt"
   find /dir -name "*.txt"  //可以使用通配符来过滤  
   find /dir -iname "abc.txt" //忽略大小写的查找
   
限制目录查找的深度
   find /dir -maxdepth 2 -name "*.txt" //表示查找当前文件夹递归查找的深度为2,如果为1表示只在当前目前查找
   
反向查找
   find /dir -not -name "*.txt"  //查找所有非txt文件名结尾的文件
   
复合查找(And)
   find /dir -name "pro*" -not -name "*.txt" //查找所有以pro开头但不是以txt结尾的文件
   
复合操作(OR)
   find /dir -name "*.sh" -o -name "*.log"  //查找脚本或者是日志的文件 
   
指定搜索类型
   find ./dir -type f -name "asd*" //只查找asd开头的文件
   find ./dir -type d -name "asd*" //只查找asd开头的文件目录
   
查找在指定大小的文件
   find /dir -size +50M -size -100M
```

>history命令
```
history n //列出最近n笔的历史命令
```
![history](https://github.com/tinysKai/JavaNote/blob/master/image/settle/history.png)

>linux上登录MySQL : mysql -h 127.0.0.1 -u mysql -p

>linux逻辑命令

    cmd1 &amp;&amp; cmd2 : cmd1执行成功会接着执行cmd2;cmd1执行失败则不会会执行cmd2
    cmd1 || cmd2 : cmd1执行成功,则不运行cmd2;cmd1执行失败,则运行cmd2
    
>tar命令

    压缩 : tar -zcv -f filename.tar.gz 要压缩的文件或目录
    解压 : tar -zxv -f filename.tar.gz  [-C 解压的地址] 
    查看 : tar -ztv -f filename.tar,gz
    
![tar-](https://github.com/tinysKai/JavaNote/blob/master/image/settle/tar-.png)    
     
>tar仅解压单一文件的方法
![tar](https://github.com/tinysKai/JavaNote/blob/master/image/settle/tar.png)


>grep的用法  
grep [-A] [-B] [-color] '搜索字符串' filename  
grep -w word file  -w选项指定要搜索的单词

>grep-搜索在文件中包含某字符串的文件名  
grep -l "string" filename  
![tar](https://github.com/tinysKai/JavaNote/blob/master/image/settle/grep-l.png)

>添加用户 

     使用者账号有关的有两个非常重要的文件，
     一个是管理使用者 UID/GID 重要参数的 /etc/passwd ，
     一个则是专门管理口令相关数据的 /etc/shadow ！
     
     第一步 : 添加用户
          useradd username
     第二步 : 使用root赋予密码..[注意不加username是修改自己的密码]
          passwd username


>linux同时更改文件或目录的所有者和用户组  
chown -R owner:group folder

>本地复制到远程scp  
scp  local_file remote_username@remote_ip:remote_folder  
scp -r local_folder remote_username@remote_ip:remote_folder

>vim之后可是使用ctrl + z 把vim切换到后台..想切换回来可以使用fg命令切换回来..

>后台执行命令：&和nohup command & 

     1、&amp;
     当在前台运行某个作业时，终端被该作业占据；可以在命令后面加上& 实现后台运行。例如：sh test.sh &
     适合在后台运行的命令有f i n d、费时的排序及一些s h e l l脚本。在后台运行作业时要当心：需要用户交互的命令不要放在后台执行，因为这样你的机器就会在那里傻等。不过，作业在后台运行一样会将结果输出到屏幕上，干扰你的工作。如果放在后台运行的作业会产生大量的输出，最好使用下面的方法把它的输出重定向到某个文件中：
     [plain] view plain copy
     command  >  out.file  2>&amp;1 &amp;  
     这样，所有的标准输出和错误输出都将被重定向到一个叫做out.file 的文件中。
     注意：当你成功地提交进程以后，就会显示出一个进程号，可以用它来监控该进程，或杀死它。(ps -ef | grep 进程号     或者   kill -9 进程号)
     
     2、nohup命令：
     使用&命令后，作业被提交到后台运行，当前控制台没有被占用，但是一但把当前控制台关掉(退出帐户时)，作业就会停止运行。nohup命令可以在你退出帐户之后继续运行相应的进程。nohup就是不挂起的意思( no hang up)。该命令的一般形式为： nohup command &
     如果使用nohup命令提交作业，那么在缺省情况下该作业的所有输出都被重定向到一个名为nohup.out的文件中，除非另外指定了输出文件： 
     nohup command > myout.file 2>&amp;1
     
     注意使用nohup时要注意使用exit退出终端来避免后台运行的线程退出..


>xargs  
![xargs](https://github.com/tinysKai/JavaNote/blob/master/image/settle/xargs.png)

>通过进程号查找对应的启动路径
![xargs](https://github.com/tinysKai/JavaNote/blob/master/image/settle/proc.png)

ps -ef |grep redis | grep -v grep | awk '{print "/proc/"$2}' | xargs ls -al | grep exe
![xargs](https://github.com/tinysKai/JavaNote/blob/master/image/settle/proc1.png)

>找出当前系统cpu使用量较高的进程  
 ps -ef | sort -rnk 4 |head -10   
 sort解释:-n是按照数字大小排序，-r是以相反顺序，-k是指定需要爱排序的栏位，-t可用于指定栏位分隔符

>查看进程的所有变量  
strings /proc/进程号/environ

>查找某进程下消耗cpu最好的线程
top -p 进程号 
按下"shift + h"

>查看环境变量  
cat /etc/profile  |  cat ~/.profile   |  cat ~/.bashrc

>删除文件时注意事项
删除软链接时要特别注意,在软链接后面不能加"/"