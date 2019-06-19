## SQL的各种JOIN方法

LEFT JOIN、RIGHT JOIN、INNER JOIN、OUTER JOIN 相关的 7 种用法

![https://mmbiz.qpic.cn/mmbiz_png/vqlbVFl5Jn2fx4T60E8RNV6gdvX5icOExwpTtQibfXrlOZYdjwtuteH64IkRrgDrulic56ypMMWuRk5f8ZemQSxaA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://ws3.sinaimg.cn/large/005BYqpggy1g2dwc5k246j30gv0dcgrv.jpg)

**Inner Join(内连接)**

![https://mmbiz.qpic.cn/mmbiz_png/vqlbVFl5Jn2fx4T60E8RNV6gdvX5icOExUU3Fx6DNaowhDnIGPW6cntMiaL7rQpFiazPJ0HKVhrLn4oUnxSCCibnxQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://ws3.sinaimg.cn/large/005BYqpggy1g2dwdtdp5yj307504vweu.jpg)

```sql
SELECT <select_list> 
FROM Table_A A
INNER JOIN Table_B B
ON A.Key = B.Key
```



**LEFT JOIN（左连接）**

![https://mmbiz.qpic.cn/mmbiz_png/vqlbVFl5Jn2fx4T60E8RNV6gdvX5icOExw45or3NyHaSTqd1Mmzhf6JRy6jWiamctg6kiad346noicAXIup3E08DoQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://ws3.sinaimg.cn/large/005BYqpggy1g2dwfd36uwj307704t0t8.jpg)

```sql
SELECT <select_list>
FROM Table_A A
LEFT JOIN Table_B B
ON A.Key = B.Key
```

**RIGHT JOIN（右连接）**

![https://mmbiz.qpic.cn/mmbiz_png/vqlbVFl5Jn2fx4T60E8RNV6gdvX5icOExkGOyFZR3MX9icticTh4R08Dic9tHaaV1C7fmf4ZicfeLsPtXQUPFtTG8Hg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://ws3.sinaimg.cn/large/005BYqpggy1g2dwgfqrlwj307804t3yz.jpg)

```sql
SELECT <select_list>
FROM Table_A A
RIGHT JOIN Table_B B
ON A.Key = B.Key
```

**OUTER JOIN（外连接）**

![https://mmbiz.qpic.cn/mmbiz_png/vqlbVFl5Jn2fx4T60E8RNV6gdvX5icOExjh6HuicxI6bfuPJWITH6gL0G2Qfibiax2WYH5G2GKk0LAVQgCH6QicUlPA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://ws3.sinaimg.cn/large/005BYqpggy1g2dwjrmmo1j307504wmxr.jpg)

```sql
SELECT <select_list>
FROM Table_A A
FULL OUTER JOIN Table_B B
ON A.Key = B.Key
```



**LEFT JOIN EXCLUDING INNER JOIN（左连接-内连接）**

![https://mmbiz.qpic.cn/mmbiz_png/vqlbVFl5Jn2fx4T60E8RNV6gdvX5icOExib6x7rn8v34TdcaNgichnjvswLEkEalFQGdcEjz8la7pyPRickEG98fNQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://ws3.sinaimg.cn/large/005BYqpggy1g2dwhyelvrj307804w3yz.jpg)

```sql
SELECT <select_list> 
FROM Table_A A
LEFT JOIN Table_B B
ON A.Key = B.Key
WHERE B.Key IS NULL
```

**RIGHT JOIN EXCLUDING INNER JOIN（右连接-内连接）**

![https://mmbiz.qpic.cn/mmbiz_png/vqlbVFl5Jn2fx4T60E8RNV6gdvX5icOExE0oXX6FpuKyOOsC4lxvSTWbefQK0F7RgtvP2YqAuxsibhjWW9ljfqRw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://ws3.sinaimg.cn/large/005BYqpggy1g2dwlmp80bj307504w0t5.jpg)

```sql
SELECT <select_list>
FROM Table_A A
RIGHT JOIN Table_B B
ON A.Key = B.Key
WHERE A.Key IS NULL
```



**OUTER JOIN EXCLUDING INNER JOIN（外连接-内连接）**

![https://mmbiz.qpic.cn/mmbiz_png/vqlbVFl5Jn2fx4T60E8RNV6gdvX5icOExa24qEdib5Z4EElk8dRbADrfHTqE8icmBicGZibXGm4YpTvdhJuFpNgdD5Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1](https://ws3.sinaimg.cn/large/005BYqpggy1g2dwm8noo9j307704vdgf.jpg)

```sql
SELECT <select_list>
FROM Table_A A
FULL OUTER JOIN Table_B B
ON A.Key = B.Key
WHERE A.Key IS NULL OR B.Key IS NULL
```



**参考链接**

http://url.cn/5O4htik