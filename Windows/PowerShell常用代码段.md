#### 变量与环境变量设置
```
# 设置变量设置
$Username=user
$Password=password

# 临时环境变量设置，powershell窗口关闭则失效
$env:Username="user"
$env:FadadaPassword="password"

# 永久生效的环境变量，系统级别
[environment]::SetEnvironmentvariable("JRE_HOME","C:\Program Files\Java\jdk1.7.0_80\jre","Machine")

# 永久生效的环境变量，当前用户
[environment]::SetEnvironmentvariable("JRE_HOME","C:\Program Files\Java\jdk1.7.0_80\jre","User")

# 通配查看环境变量
ls Env:test-*

# 使用环境变量
$env:test
```

#### 下载不保存powershell脚本直接运行
```
$Username=user
$password=password
$Script="https://example.com/example.ps1"
# 下载地址为http时，去掉下面一行
[Net.ServicePointManager]::SecurityProtocol=[Net.SecurityProtocolType]::Tls12
$Wc=new-object System.Net.WebClient
$CredCache=new-object System.Net.CredentialCache
$Creds=new-object System.Net.NetworkCredential($env:ExampleUsername,$password)
$Wc.Credentials=$CredCache.Remove("$Script","Basic")
$CredCache.Add("$Script", "Basic", $Creds)
$Wc.Credentials=$CredCache
iex $Wc.DownloadString($Script)
```

#### 一个简单的带时间戳的日志输出函数
```
function LogTime($Log)
{
    Get-Date -Format 'yyyy-MM-dd-HH_hh:mm:ss' | Tee-Object -Variable NowTime
    Write-Host "$NowTime $Log"
    "$NowTime $Log"|Out-File -Append "C:\info.log"
}

# 使用方法
LogTime($Log)
```

#### http状态码检测函数，状态码大于400表示网络联通性无误
```
function TestHttpStatusCode($url)
{
	$req = [system.Net.WebRequest]::Create($url)

	try {
		$res = $req.GetResponse()
	} 
	catch [System.Net.WebException] {
		$res = $_.Exception.Response
	}
    $HttpStatusCode=[int]$res.StatusCode
    if ( $HttpStatusCode -gt 400 -and $HttpStatusCode -ne 401) {
        LogTime("${url}无法访问，状态码为${HttpStatusCode}！！！")
        exit
    }
    else {
        LogTime("${url}连通性检测通过！！！")
    }
}

# 使用方法
$Addrs="https://baidu.com","http://qq.com"
foreach ($Addr in $Addrs) {
    TestHttpStatusCode("$Addr")
}
```

#### 下载文件函数
```
function DownloadFile($Rfile)
{
    $Tempdir="C:\Tempdir"
    $Username="user"
    $Password="password"
    $Password=ConvertTo-SecureString $Password -AsPlainText -Force
    $Redential=New-Object System.Management.Automation.PSCredential($Username,$Password)

    $Filename=$Rfile.Split("/")[$Rfile.Split("/").Length-1]
    LogTime("正在下载文件：${Rfile},存放路径为：${Tempdir}\${Filename}")

    # 访问地址为http时，去掉下面一行
    [Net.ServicePointManager]::SecurityProtocol=[Net.SecurityProtocolType]::Tls12

    Invoke-WebRequest -Uri $Rfile -OutFile "${Tempdir}\${Filename}" -Credential $Redential
    if (-not $?){
        exit
    }
}

# 使用方法
$Files = $Hashtable.ExampleSystemSignPkg,"https://example.com/1.txt","https://example.com/2.txt"
foreach ($File in $Files) {
    DownloadFile("$File")
}
```

#### 创建程序快捷方式至桌面函数
```
$Desktop = [System.Environment]::GetFolderPath('Desktop')
function SaveDesktopShortcut([string]${TargetFile}, [string]${ShortcutName}, [string]${IconLocation})
{
$Shell = New-Object -ComObject WScript.Shell
Remove-Item ${Desktop}\${ShortcutName}'.lnk' -Recurse -ErrorAction SilentlyContinue
$Shortcut = $Shell.CreateShortcut("${Desktop}\${ShortcutName}"+'.lnk')
$Shortcut.TargetPath = "$TargetFile"
$Shortcut.IconLocation = "$IconLocation"
$Shortcut.Save()
}

# 使用方法
SaveDesktopShortcut "C:\navicat\Navicat for MySQL\navicat.exe" "NavicatForMySQL" "C:\navicat\Navicat for MySQL\navicat.exe"
```

#### 读取配置文件
```
# example.conf内容
k1=123
k2=456

$Hashtable = @{}
$Payload = Get-Content -Path 'example.conf' |
Where-Object { $_ -like '*=*' } |
ForEach-Object {
    $Infos = $_ -split '='
    $Key = $Infos[0].Trim()
    $Value = $Infos[1].Trim()
    $Hashtable.$Key = $Value
}

# 使用方法,k1为example.conf中的键
$Hashtable.k1
$Hashtable.k2
```

#### 获取系统CPU、内存信息
```
$SysMemory=$([Math]::Round((Get-WmiObject -Class Win32_ComputerSystem).TotalPhysicalMemory /1gb))
$Cpu=get-wmiobject win32_processor
$SysCpuProcessors=@($Cpu).count*$Cpu.NumberOfLogicalProcessors
```

#### for循环
```
for($t=5; $t -ge 0; $t=$t-1)
{
    LogTime("${t}秒后检查系统资源（CPU、内存、磁盘）是否充足！！！")
    Start-Sleep -Seconds 1
}
```

#### 防火墙管理
```
# 启用Windows防火墙
Set-NetFirewallProfile -All -Enabled true

# 清理法大大历史防火墙规则
Remove-NetfirewallRule -DisplayName "Tcp8080" -ErrorAction SilentlyContinue
Remove-NetfirewallRule -DisplayName "Tcp8887" -ErrorAction SilentlyContinue

# 创建防火墙规则：Tcp8080 Tcp8887
New-NetFirewallRule -Name Tcp8080 -Direction Inbound -DisplayName 'Tcp8080' -LocalPort 8080 -Protocol 'TCP' -ErrorAction SilentlyContinue
New-NetFirewallRule -Name Tcp8887 -Direction Inbound -DisplayName 'Tcp8887' -LocalPort 8887 -Protocol 'TCP' -ErrorAction SilentlyContinue
```

#### 文本字符串替换
```
$ConfigFile=a.conf
$CatalinaFile=catalina.bat
SystemConfig=System.conf

# 简单字符串替换，
(Get-Content "$ConfigFile") | Foreach-Object {$_ -replace '256',"${SystemMaxMemoryMB}"} | Set-Content "$ConfigFile"
# 字符串替换，带字符串拼接
(Get-Content -Encoding utf8 "$CatalinaFile") | Foreach-Object {$_ -replace 'Xmx3g',"Xmx${SystemMaxMemory}g"} | Set-Content -Encoding utf8 "$CatalinaFile"
# 字符串替换,以test开头的全部替换
(Get-Content -Encoding utf8 "$SystemConfig") | Foreach-Object {$_ -replace '(?<=^)test.*',"qcloud.secretkey=$Hashtable.ExampleQcloudSecretKey"} | Set-Content -Encoding utf8 "$SystemConfig"
# 字符串替换，带转义的
(Get-Content -Encoding utf8 "$SystemConfig") | Foreach-Object {$_ -replace '=D\\:','=C:'}  | Set-Content -Encoding utf8 "$SystemConfig"
```

#### 查询公网IP
```
$Req=Invoke-WebRequest -Uri "http://ip.sb"
$Req.content
```
