# RouterOS Cloudflare DDNS 更新器

这是一个用于 Mikrotik RouterOS 的DDNS脚本，当路由器的公网 IP 地址发生变化时可以自动更新 Cloudflare DNS 记录。

## 功能特点

- 支持 Cloudflare API v4
- 支持通过 Mikrotik Cloud 或直接指定接口获取公网 IP
- （多接口支持）可以指定存IP的临时文件，指定不同文件来支持多个接口

## 使用前准备

你需要准备以下信息：

1. Cloudflare API Token
2. Zone ID (在 Cloudflare 域名概览页面可以找到)
3. 需要更新的域名(支持通配符*， 例如\*.example.com 或者 a.example.com)
4. DNS Record ID

### 获取 Cloudflare API Token

1. 登录 Cloudflare 控制面板
2. 点击右上角的个人资料图标，选择 "My Profile"
3. 点击 "API Tokens"，然后点击 "Create Token"
4. 选择 "Edit zone DNS" 模板
5. 在 "Zone Resources" 中选择你的域名
6. 创建并保存 Token

### 获取 DNS Record ID

1.为你的DDNS 域名先创建一个DNS 解析，解析到任意IP地址。

2.以下命令会取得你当前域名下 Record ID(ID)

在执行下面的 curl 命令之前，请填写 4 个初始变量并将其粘贴到 shell 中（或直接在 curl 命令中设置值）。  

```
CFDomain=""
CFtkn=""
CFzoneid=""

curl -s -X GET "https://api.cloudflare.com/client/v4/zones/$CFzoneid/dns_records?type=A&name=$CFDomain" \
	-H "Authorization: Bearer $CFtkn" \
	-H "Content-Type: application/json" | jq
```



## 配置说明

在脚本中需要配置以下变量：

```
:local WANInterface "pppoe_out_1"    # WAN 接口名称
:local tempFile "ddns_wan_ip"				 # 历史IP（记录IP是否变化）
:local CFDomain ""                   # 需要更新的域名
:local CFtkn ""                      # Cloudflare API Token
:local CFzoneid ""                   # Zone ID
:local CFid ""                     	 # DNS Record ID
```





## 安装步骤

1. 登录到 RouterOS Web 界面或使用 WinBox
2. 进入 System > Scripts
3. 点击 Add New
4. 设置脚本名称（例如：cf-ddns）
5. 在 Policy 中选择：read, write, policy, test
6. 将脚本内容复制到 Source 字段
7. 点击 Apply 保存

## 设置定时任务

1. 进入 System > Scheduler
2. 点击 Add New
3. 设置任务名称
4. 设置执行间隔（建议：00:05:00，即每5分钟）
5. Policy 选择：read, write, policy, test
6. On Event 填入脚本名称（例如：cf-ddns）
7. 点击 Apply 保存

## 调试

如果需要调试，可以将脚本中的 `CFDebug` 变量设置为 "true"：

```
:local CFDebug "true"
```

调试信息将写入系统日志。

## IP 获取方式

脚本支持两种获取公网 IP 的方式：

1. 通过 Mikrotik Cloud（设置 `CFCloud = "true"`）
2. 通过指定接口（默认方式）

## 注意事项

- 确保路由器能够访问 Cloudflare API (api.cloudflare.com)
- 建议将 TTL 设置为较小的值（如 120 秒）
- 首次运行时会创建临时文件保存当前 IP
- 确保 API Token 具有足够的权限

## 许可证

MIT License

这个 README 提供了完整的项目说明、配置步骤和注意事项。您可以根据实际需求进行修改和补充。

```
:local WANInterface "pppoe_out_1"
:local tempFile "ddns_wan_ip"

:local CFDomain ""
:local CFtkn ""
:local CFzoneid ""
:local CFid ""

:local CFrecordType "A"
:local CFrecordTTL "120"

:local previousIP ""
:local WANip ""

:local CFDebug "false"
:local CFCloud "false"

################# Build CF API Url (v4) #################
:local CFurl "https://api.cloudflare.com/client/v4/zones/"
:set CFurl ($CFurl . "$CFzoneid/dns_records/$CFid");

################# Get or set previous IP-variables #################
:if ($CFCloud = "true") do={
    :set WANip [/ip cloud get public-address]
} else={
    :local currentIP [/ip address get [/ip address find interface=$WANInterface ] address];
    :set WANip [:pick $currentIP 0 [:find $currentIP "/"]];
}

:if ([/file find name=$tempFile] = "") do={
    :log error "No previous ip address file found, creating..."
    :set previousIP $WANip;
    :execute script=":put $WANip" file=$tempFile;
    :log info ("CF: Updating CF, setting $CFDomain = $WANip")
    /tool fetch http-method=put mode=https output=none url="$CFurl" http-header-field="Authorization:Bearer $CFtkn,content-type:application/json" http-data="{\"type\":\"$CFrecordType\",\"name\":\"$CFDomain\",\"ttl\":$CFrecordTTL,\"content\":\"$WANip\"}"
    :error message="No previous ip address file found."
} else={
    :if ( [/file get [/file find name=$tempFile] size] > 0 ) do={ 
        :local content [/file get [/file find name=$tempFile] contents];
        :local contentLen [:len $content];  
        :local lineEnd 0;
        :local line "";
        :local lastEnd 0;   
        :set lineEnd [:find $content "\n" $lastEnd];
        :set line [:pick $content $lastEnd $lineEnd];
        :set lastEnd ($lineEnd + 1);   
        :if ([:pick $line 0 1] != "#") do={   
            :set previousIP [:pick $line 0 $lineEnd];
            :set previousIP [:pick $previousIP 0 [:find $previousIP "\r"]];
        }
    }
}

######## Write debug info to log #################
:if ($CFDebug = "true") do={
    :log info "Updating IP $CFDomain ..."
    :log info ("CF: hostname = $CFDomain")
    :log info ("CF: previousIP = $previousIP")
    :log info ("CF: currentIP = $currentIP")
    :log info ("CF: WANip = $WANip")
    :log info ("CF: CFurl = $CFurl&content=$WANip")
    :log info ("CF: Command = \"/tool fetch http-method=put mode=https url=\"$CFurl\" http-header-field=\"Authorization:Bearer $CFtkn,content-type:application/json\" output=none http-data=\"{\"type\":\"$CFrecordType\",\"name\":\"$CFDomain\",\"ttl\":$CFrecordTTL,\"content\":\"$WANip\"}\"")
};

######## Compare and update CF if necessary #####
:if ($previousIP != $WANip) do={
    :log info ("CF: Updating CF, setting $CFDomain = $WANip")
    /tool fetch http-method=put mode=https url="$CFurl" http-header-field="Authorization:Bearer $CFtkn,content-type:application/json" output=none http-data="{\"type\":\"$CFrecordType\",\"name\":\"$CFDomain\",\"ttl\":$CFrecordTTL,\"content\":\"$WANip\"}"
    /ip dns cache flush
    :if ( [/file get [/file find name=$tempFile] size] > 0 ) do={
        /file remove $tempFile
        :execute script=":put $WANip" file=$tempFile
    }
} 

```