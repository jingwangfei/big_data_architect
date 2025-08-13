# 一：基本环境

```bash
cd /mnt/disk28/logAnalysisPlatform
jps -lvm | grep  LogAnalysisPlatform-1.0.jar
```

<img width="1660" height="192" alt="image" src="https://github.com/user-attachments/assets/589be939-f9d1-4658-9480-284439685c0f" />


```bash
Jinfo 492998
```
<img width="1108" height="567" alt="image" src="https://github.com/user-attachments/assets/5dfebc47-cdda-4b9e-aa74-f54efa4d669a" />


# 二：内存使用分析

```bash
jmap -heap 492998
```
<img width="1754" height="1344" alt="image" src="https://github.com/user-attachments/assets/ff01a636-46fa-4bf1-8405-22633d458634" />


```bash
jmap -dump:format=b,file=heap.hprof  492998
jmap -dump:live,format=b,file=heap.hprof  492998
```


## MAT工具分析
下载地址： https://eclipse.dev/mat/download/  


## MAT工具使用
<img width="1105" height="580" alt="image" src="https://github.com/user-attachments/assets/6c0cb67c-a755-45fb-97f7-4b1189fb3561" />


# 三：GC分析
```bash
jstat -gcutil  986454 1000
```
<img width="1854" height="748" alt="image" src="https://github.com/user-attachments/assets/549dafbf-234c-4825-ae2f-5555f33a8c2a" />

# 四：线程分析
## 3.1 线程状态分析
```bash
jstack -l  986454
```
<img width="1107" height="617" alt="image" src="https://github.com/user-attachments/assets/51e5d2b7-35fc-404e-b121-729e1aa343d5" />


## 3.2 线程CPU与负载分析
```bash
top -H -p 986454
```

<img width="1107" height="691" alt="image" src="https://github.com/user-attachments/assets/33dc8106-bc32-4679-bc36-d61a559015f1" />


## 3. 记录高 CPU 线程的 PID（十进制），转换为十六进制
```bash
printf "%x\n" <线程PID>
```
例如 12345 → 0x3039  


## 3.3 IO负载分析
```bash
pidstat -t -d 5 -p 986454
```
<img width="1105" height="359" alt="image" src="https://github.com/user-attachments/assets/17153b64-ec2b-40d8-aa80-4c463f571899" />


```bash
iotop -p 986454
```
<img width="1106" height="188" alt="image" src="https://github.com/user-attachments/assets/4371513a-a585-4012-abd5-fff2abb1a561" />
