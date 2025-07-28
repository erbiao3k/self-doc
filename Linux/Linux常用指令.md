#### 匹配key后，批量删除redis key
```
redis-cli  -p 6380 -n 1 keys "*tick*20220120*" | xargs redis-cli -p 6380  -n 1 del
```
#### 判断某个字段是否匹配指定值
```
awk  -F","  '{if($4=="value"){print $1} else {print  $0}}' file.txt
```
#### 批量修改文件后缀名
```
find /data/ -name "*.pdf"  |while read name;do na=$(echo $name|sed s'/pdf/PDF/g'); mv $name $na; done
```
```
rename .txt .sh *.txt
```
#### 在文件内容每个行首添加内容
```
sed  s'/^/black_ip_/g' file.txt
```
```
sed s'/^/del /g'  file.txt
```
#### 在文件内容每个行末追加内容
```
sed s'/$/del /g'  file.txt
```
#### 将匹配关键字的进程kill掉
```
ps -ef |grep '关键字' |awk '{print $2}' |xarg kill -9
```
#### 匹配到某个关键字时，输出某个字段信息
```
ps -ef |awk '/mongod/{print $2}'
```
#### 杀死匹配正则表达式的进程
```
killall -r 'regular expression'
```
#### 生成6位随机字符串
```
cat /dev/urandom | head -1 | md5sum | head -c 6
```
#### 显示文件中关键词前三行和后两行
```
grep -A 2 -B 3 file.txt
```
#### 实时监视一个文件是否改动
```
watch -d -n 0.01 "cat file.txt"
```
#### 同时解压多个包
```
for tar in *.tar.gz;  do tar zxvf $tar; done
```
#### 使用user账户执行一个命令，但不切换到该用户
```
sudo -u user cat file.txt
```
#### 字符串大写转小写
```
tr '[a-z]' '[A-Z]' < input.txt >output.txt
```
#### 小写转大写，大写转小写
```
echo a-z-as-d-a-d-a-a-d-a-sd-asd-A-F-G-H--H-JJ-J-J-S-FSFS-- |tr 'a-zA-Z' 'A-Za-z'
```
#### 删除变量中的"-"字符
```
UUID="131237812-48122908348120-98371209381203";echo ${UUID//1/}
```
#### 删除同一目录下的多个具体文件
```
rm -rf /tmp/{file1.txt,file2.txt}
```
#### 删除同一目录下的多个文件，模糊匹配
```
rm -rf ${SERVICE_PATH}/{nginx*.tar.gz,openssl*.tar.gz,php*.tar.gz}
```
#### 将文件FILE1的access时间和modify时间同步给FILE2，但此操作将更新FILE2的change时间为命令执行时间
```
touch -r FILE1 FILE2
```
#### 使用echo检查命令,避免误操作
```
echo rm *.txt
```
#### 让执行的命令不被记录到history里
```
cat |bash
```
#### 查看指定PID的进程数量
```
ps uH PID_of_PROCESS |wc -l
```

#### 变量自增
```
((x++))
```
#### 用vim远程编辑文件/root/bin/10rsh，保存时需要密码。目录前面多一个"/"
```
vim scp://172.25.1.1//root/bin/10rsh
```
#### 将文件复制到多个位置
```
cat file |tee dest1 dest2 >dev/null 2>&1
```
#### 显示字符串的同时输出到文件
```
echo "hello world" |tee -a file.txt
```
#### 获取文件或目录的绝对路径
```
readlink -f file.txt
```
#### 请输入密码实现
```
read -p"请输入你的密码："　　　　　　明文显示你的输入
```
```
read -s -p"请输入你的密码："　　　　不显示你的输入
```
#### 追踪top命令并在vim中打开实时刷新
```
strace top 2>&1 > /dev/null |vim -c ':set syntax=strace -'
```
#### 打开文件并搜索"关键字"
```
vim +关键字 file.txt
```
#### 创建文件备份
```
cp file.txt{,.bak}
```
#### 关闭文件系统自检（fsck）
```
tune2fs  -c -1 -i 0  /dev/sdb1
```
#### 查看系统逻辑处理器个数
```
grep processor /proc/cpuinfo |wc -l
```
#### 追踪php-fpm进程的系统调用信息
```
rm -rvf /tmp/strace_* && nohup ps -ef | grep  php-fpm | awk '{print " -p " $2" -s 10000 -o /tmp/strace_"$2".log"}' | xargs strace &
```
#### 将内网服务器端口反向绑定到具备公网IP的服务器
```
# /etc/ssh/sshd_config文件中确保：GatewayPorts clientspecified
# 新建账户专用，禁止使用root
ssh -o ServerAliveInterval=20 \
    -o ServerAliveCountMax=5 \
    -o ExitOnForwardFailure=yes \
    -o TCPKeepAlive=yes \
    -f -N -R 0.0.0.0:18888:192.168.1.248:6666 tunn@77.44.33.22
```
