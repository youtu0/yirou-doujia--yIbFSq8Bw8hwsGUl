
VMware Cloud Foundation 环境中，软件包仓库的来源支持两种方式，分别是 **Online Depot** 和 **Offline Depot**。第一种方式，是在 VCF 环境能够连接互联网的情况下，仅需要配置账号密码后，就可以直接从 VMware 官方在线存储库获取软件包；第二种方式，是在 VCF 环境不能连接互联网的情况下，通过在本地搭建一个 Web 服务器并搭配 Offline Bundle Transfer Utility（OBTU）工具来作为 VCF 环境的离线/脱机存储库。


在线仓库的方式固然非常方便，但是，往往很多环境因为各种原因无法连接互联网，这时候就只能使用离线仓库了。VMware 针对 VCF 环境推出了一种配置离线仓库的方式，这在很大程度上能为客户提供便利性，特别是环境中具有多个 VCF 实例的场景。你只需在本地找一台能够连接互联网的 Linux 服务器并配置为 Web 服务器，然后使用 Offline Bundle Transfer Utility（OBTU）工具将 VCF 相关软件包下载到这个服务器内，最后再到 SDDC Manager 中配置脱机库后，就能实现与在线仓库一样的效果。说到这里，你是不是会发现这跟 vSphere 环境中 [Update Manager Download Service（UMDS）](https://github.com) 的使用方式非常相似？！


[![](https://img2024.cnblogs.com/blog/2313726/202410/2313726-20241029100225137-1146928245.png)](https://github.com)


当然，上面这种 OBTU \+ Web 服务器来作为 VCF 的脱机库只是一种使用方式，你依然可以通过以下方式去手动下载离线软件包，然后再手动上传至 SDDC Manager，来完成组件的生命周期管理。不过这还是比较麻烦，所以，下面我将演示如何去准备一个 OBTU 服务器来配置作为 VCF 环境的脱机库。


* [Offline Download of VMware Cloud Foundation 5\.2\.x Upgrade Bundles](https://github.com)
* [Offline Download of Async Patch Bundles](https://github.com)
* [Offline Download of Flexible BOM Upgrade Bundles](https://github.com)
* [Offline Download of Independent SDDC Manager Bundles](https://github.com)


本文以下内容参考 VMware 官方产品文档[《VMware Cloud Foundation Lifecycle Management》](https://github.com)、知识库文章（[KB 312168](https://github.com)）以及相关博客文章（[VMware Cloud Foundation Offline Depot Introduction](https://github.com)）。


 



## 一、准备 OBTU 服务器



此次环境准备了一台基于 CentOS 发行版的 OBTU 服务器用于作为 VCF 环境的脱机存储库。



```
[root@localhost ~]# cat /etc/os-release 
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"

[root@localhost ~]# cat /etc/centos-release
CentOS Linux release 7.9.2009 (Core)
[root@localhost ~]# 
[root@localhost ~]# uname -a
Linux localhost 3.10.0-1160.el7.x86_64 #1 SMP Mon Oct 19 16:18:59 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
[root@localhost ~]# 
```

配置主机名、Hosts 文件，将 Selinux 和 Firewalld 防火墙关闭。



```
[root@localhost ~]# hostnamectl set-hostname vcf-obtu
[root@localhost ~]# 
[root@localhost ~]# bash
[root@vcf-obtu ~]# 
[root@vcf-obtu ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.32.56 vcf-obtu.mulab.local
[root@vcf-obtu ~]# vim /etc/selinux/config 
[root@vcf-obtu ~]# cat /etc/selinux/config 

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted 


[root@vcf-obtu ~]# systemctl stop firewalld
[root@vcf-obtu ~]# systemctl status firewalld
   firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
[root@vcf-obtu ~]# systemctl disable firewalld
```

请确保准备足够的空间（/obtu）用于存放 VCF 软件包，本次环境中新增加了一块硬盘。



```
[root@vcf-obtu ~]# df -hT
Filesystem              Type      Size  Used Avail Use% Mounted on
devtmpfs                devtmpfs  3.9G     0  3.9G   0% /dev
tmpfs                   tmpfs     3.9G     0  3.9G   0% /dev/shm
tmpfs                   tmpfs     3.9G  8.9M  3.9G   1% /run
tmpfs                   tmpfs     3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root xfs        14G  2.6G   11G  20% /
/dev/sdb1               ext4      493G   73M  467G   1% /obtu
/dev/sda1               xfs      1014M  150M  865M  15% /boot
tmpfs                   tmpfs     783M     0  783M   0% /run/user/0
[root@vcf-obtu ~]#
```

 



## 二、安装 Apache 服务器



使用 YUM 安装 Apache 服务器，注意需要同时安装 **`mod_ssl`** 模块。



```
[root@vcf-obtu ~]# yum install -y httpd mod_ssl jq
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package httpd.x86_64 0:2.4.6-99.el7.centos.1 will be installed
---> Package jq.x86_64 0:1.6-2.el7 will be installed
---> Package mod_ssl.x86_64 1:2.4.6-99.el7.centos.1 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

============================================================================================================================
 Package                  Arch                    Version                                    Repository                Size
============================================================================================================================
Installing:
 httpd                    x86_64                  2.4.6-99.el7.centos.1                      updates                  2.7 M
 jq                       x86_64                  1.6-2.el7                                  epel                     167 k
 mod_ssl                  x86_64                  1:2.4.6-99.el7.centos.1                    updates                  116 k

Transaction Summary
============================================================================================================================
Install  3 Packages

Total download size: 3.0 M
Installed size: 10 M
Downloading packages:
(1/3): httpd-2.4.6-99.el7.centos.1.x86_64.rpm                                                        | 2.7 MB  00:00:00     
(2/3): jq-1.6-2.el7.x86_64.rpm                                                                       | 167 kB  00:00:00     
(3/3): mod_ssl-2.4.6-99.el7.centos.1.x86_64.rpm                                                      | 116 kB  00:00:00     
----------------------------------------------------------------------------------------------------------------------------
Total                                                                                       6.1 MB/s | 3.0 MB  00:00:00     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : httpd-2.4.6-99.el7.centos.1.x86_64                                                                       1/3 
  Installing : 1:mod_ssl-2.4.6-99.el7.centos.1.x86_64                                                                   2/3 
  Installing : jq-1.6-2.el7.x86_64                                                                                      3/3 
  Verifying  : httpd-2.4.6-99.el7.centos.1.x86_64                                                                       1/3 
  Verifying  : jq-1.6-2.el7.x86_64                                                                                      2/3 
  Verifying  : 1:mod_ssl-2.4.6-99.el7.centos.1.x86_64                                                                   3/3 

Installed:
  httpd.x86_64 0:2.4.6-99.el7.centos.1         jq.x86_64 0:1.6-2.el7         mod_ssl.x86_64 1:2.4.6-99.el7.centos.1        

Complete!
[root@vcf-obtu ~]#
```

启动 Apache 服务并加入开机自启动。



```
[root@vcf-obtu ~]# systemctl start httpd
[root@vcf-obtu ~]# 
[root@vcf-obtu ~]# systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2024-10-24 18:36:01 CST; 10s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 1903 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/httpd.service
           ├─1903 /usr/sbin/httpd -DFOREGROUND
           ├─1904 /usr/sbin/httpd -DFOREGROUND
           ├─1905 /usr/sbin/httpd -DFOREGROUND
           ├─1906 /usr/sbin/httpd -DFOREGROUND
           ├─1907 /usr/sbin/httpd -DFOREGROUND
           └─1908 /usr/sbin/httpd -DFOREGROUND

Oct 24 18:36:01 vcf-obtu.mulab.local systemd[1]: Starting The Apache HTTP Server...
Oct 24 18:36:01 vcf-obtu.mulab.local systemd[1]: Started The Apache HTTP Server.
[root@vcf-obtu ~]# 
[root@vcf-obtu ~]# systemctl enable httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
[root@vcf-obtu ~]#
```

 



## 三、配置 Apache 服务器



创建用于向 Web 服务器进行身份验证的用户，用户名是**`depot`**，密码是**`vmware`**。生成 OBTU 服务器的 SSL 自签名证书。



```
[root@vcf-obtu ~]# htpasswd -b -c /etc/httpd/.htpasswd depot vmware
Adding password for user depot
[root@vcf-obtu ~]# 
[root@vcf-obtu ~]# openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
>     -subj "/CN=vcf-obtu.mulab.local" \
>     -keyout /etc/pki/tls/private/offline_depot.key \
>     -out /etc/pki/tls/certs/offline_depot.crt
Generating a 2048 bit RSA private key
......+++
.............+++
writing new private key to '/etc/pki/tls/private/offline_depot.key'
-----
[root@vcf-obtu ~]#
```

创建自定义网站目录，新建一个 Web 配置文件，内容如下所示。然后，测试 Apache 配置文件的正确性并重新启动 Apache 服务。



```
[root@vcf-obtu ~]# mkdir -p /obtu/www/offline_depot
[root@vcf-obtu ~]# 
[root@vcf-obtu ~]# chown $USER:$USER /obtu/www/offline_depot
[root@vcf-obtu ~]# 
[root@vcf-obtu ~]# vim /etc/httpd/conf.d/offline_depot_httpd.conf
[root@vcf-obtu ~]# 
[root@vcf-obtu ~]# cat /etc/httpd/conf.d/offline_depot_httpd.conf


ServerName vcf-obtu.mulab.local
DocumentRoot /obtu/www/offline_depot

SSLEngine on
SSLCertificateFile /etc/pki/tls/certs/offline_depot.crt
SSLCertificateKeyFile /etc/pki/tls/private/offline_depot.key


AuthType Basic
AuthName “depot”
AuthUserFile /etc/httpd/.htpasswd
Require valid-user


Alias /products/v1/bundles/lastupdatedtime /obtu/www/offline_depot/PROD2/vsan/hcl/lastupdatedtime.json
Alias /products/v1/bundles/all /obtu/www/offline_depot/PROD2/vsan/hcl/all.json
Alias /Compatibility/VxrailCompatibilityData.json /obtu/www/offline_depot/PROD2/evo/vmw/Compatibility/VxrailCompatibilityData.json



[root@vcf-obtu ~]#
[root@vcf-obtu ~]# apachectl configtest
Syntax OK
[root@vcf-obtu ~]# 
[root@vcf-obtu ~]# systemctl reload httpd
[root@vcf-obtu ~]# 
```

现在，在网站目录中创建一个 html 测试网页，使用 curl 命令并配合用户名和密码认证来测试 OBTU 服务器的连接状态是否正常。你也可以直接通过浏览器访问，然后输入用户名和密码来测试看是否正常。



```
[root@vcf-obtu ~]# 
[root@vcf-obtu ~]# echo "Offline Depot OK" > /obtu/www/offline_depot/index.html
[root@vcf-obtu ~]# 
[root@vcf-obtu ~]# curl https://vcf-obtu.mulab.local -k --silent -u depot:vmware
Offline Depot OK
[root@vcf-obtu ~]# 
```

由于 OBTU 服务器使用了自签名 SSL 证书，所以需要将其添加到 SDDC Manager 的受信任证书。使用以下命令获取 OBTU 服务器的证书。



```
[root@vcf-obtu ~]# echo '{ "certificate" : '$(jq -sR . /etc/pki/tls/certs/offline_depot.crt)',
"certificateUsageType" : "TRUSTED_FOR_OUTBOUND" }'
{ "certificate" : "-----BEGIN CERTIFICATE-----\nMIIDETCCAfmgAwIBAgIJAJOqS5hXP2lbMA0GCSqGSIb3DQEBCwUAMB8xHTAbBgNV\nBAMMFHZjZi1vYnR1Lm11bGFiLmxvY2FsMB4XDTI0MTAyNDEwNTUzMFoXDTI1MTAy\nNDEwNTUzMFowHzEdMBsGA1UEAwwUdmNmLW9idHUubXVsYWIubG9jYWwwggEiMA0G\nCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDVDi8aICqq//qp0mBgX3bW7K6v3xnL\n2ZZBkhLOyuiYjK3b18mFPQj7T9G0whygz0WUKg1TPUkykhJ1OfRcEjFZkLP+LkLj\n0Z1in6xby2DQiJ5LlTTFhQIRv1w8++E4syR40+lszglWBVe54EtCBSUmZhc4LyZy\nEHt040S5+pPIl/QqaERHN44Kw7/bWr/iC2bAp/Oszhpv3kcTx2/qmnAEoTaZNqP0\n24/xV3f4d1xmRe8pHR1UCLybj0xKVzEALMAp/6FXaFHas71eB2gYyQFiQ0FTaJl6\nG7XpUmHdnfEFRKHdGqoXKraB7aRDKh3RcpKJk/fs4ZKpeHeBvZ85xa3XAgMBAAGj\nUDBOMB0GA1UdDgQWBBSgS5Pvan0sHT9qB7MIzXhPZA5FgjAfBgNVHSMEGDAWgBSg\nS5Pvan0sHT9qB7MIzXhPZA5FgjAMBgNVHRMEBTADAQH/MA0GCSqGSIb3DQEBCwUA\nA4IBAQAls3oBsAqKDB/bo54d9u7GWT97UNeN7Rbt0LvxFPQXM0qMfCVuTjEscPXF\nwPup6XKUXEvjmRyPlU2oJAMs4muoYWPuzHsGP3KAyi8ndUR4EyyQL7o/QxrpLqzG\nP8hnoHsiPxbZ7bnY1BQMs7nBAM230LrkjnNP84Hu183qauNIL31nWm3pPrJLERC5\npCP22azuTIgkeQvdmd59Aa/vHWaKVoUuSZrDxr/7Hj+z4J7FtZfqMS3vA9YV1A8M\njjYDIAQZyhqO9aM5Vw2aZHD3eihgw2zdHiW5ySWsNuYuPHtFraVQ3+xlk90afc2E\n0DavEodrcgZDBLUgnR3cU7BjJv1E\n-----END CERTIFICATE-----\n",
  "certificateUsageType" : "TRUSTED_FOR_OUTBOUND"
}
[root@vcf-obtu ~]#
```

登陆 SDDC Manager UI，导航到开发人员中心，然后使用 `v1/sddc-manager/trusted-certifcates` 上传生成的受信任证书。


[![](https://img2024.cnblogs.com/blog/2313726/202410/2313726-20241029105842785-10260420.png)](https://github.com):[飞数机场](https://ze16.com)


[![](https://img2024.cnblogs.com/blog/2313726/202410/2313726-20241029105854142-764384671.png)](https://github.com)


 



## 四、下载 VCF 软件包



完成 Web 服务器的相关配置后，现在需要下载 VCF 环境所使用的软件包。通过 [Broadcom 支持门户](https://github.com)下载最新的 OBTU 工具包（lcm\-tools\-prod.tar.gz），然后上传到 OBTU 服务器并解压，可以通过 `--help` 选项查看 OBTU 工具的使用帮助。



```
[root@vcf-obtu ~]# mkdir -p /opt/obtu
[root@vcf-obtu ~]# chmod 755 /opt/obtu/
[root@vcf-obtu ~]# chown $USER:$USER /opt/obtu/
[root@vcf-obtu ~]# ls lcm*
lcm-tools-prod.tar.gz
[root@vcf-obtu ~]# tar -xf lcm-tools-prod.tar.gz --directory=/opt/obtu/
[root@vcf-obtu ~]# chmod +x /opt/obtu/bin/lcm-bundle-transfer-util
[root@vcf-obtu ~]# cd /opt/obtu/bin/
[root@vcf-obtu bin]# ls -l
total 36
-rwxr-x--x 1 201 201 8761 Jan  1  2000 lcm-bundle-transfer-util
-rw-r----- 1 201 201 7422 Jan  1  2000 lcm-bundle-transfer-util.bat
-rw-r----- 1 201 201 5853 Jan  1  2000 vcf-async-patch-tool
-rw-r----- 1 201 201 4381 Jan  1  2000 vcf-async-patch-tool.bat
[root@vcf-obtu bin]# ./lcm-bundle-transfer-util --help
```

将 Broadcom 支持门户的密码（BSP\-PASSWORD）输入到一个文本文件内，然后使用以下命令执行 VCF 软件包的下载过程。注，**`BSP-USER`** 是 Broadcom 支持门户的账号，**`--sourceVersion`**是当前 VCF 环境的版本，OBTU 会根据当前版本自动去检测可用的更新版本（比如 5\.2\.1\.0）并执行下载。访问 [KB 96099](https://github.com) 了解 VCF 软件包的更多信息。



```
[root@vcf-obtu bin]# echo "BSP-PASSWORD" > ~/BSP-PASSWORD.txt
[root@vcf-obtu bin]# 
[root@vcf-obtu bin]# ./lcm-bundle-transfer-util --setUpOfflineDepot \
>   --offlineDepotRootDir /obtu/www/offline_depot \
>   --offlineDepotUrl https://vcf-obtu.mulab.local \
>   --depotUser BSP-USER \
>   --depotUserPasswordFile ~/BSP-PASSWORD.txt \
>   --sourceVersion 5.2.0.0
*********Welcome to OBTU tool***********

Make sure to download the most recent metadata files and upload them to the SDDC Manager appliance before
downloading bundles. If you do not have the most recent metadata files, the metadata for most recent upgrades will be
missing and will impact the upgrades. The following metadata files are required:  LCM manifest and VMware compatibility
data (For 5.0 or upgrade to 5.0), vSAN HCL data (For 5.1 or upgrade to 5.1). VxRail requires these additional metadata files: 
VxRail compatibility data (For 5.0 or upgrade to 5.0) and partner bundle manifest (PBM).
https://docs.vmware.com/en/VMware-Cloud-Foundation/5.0/context?id=vcf_451\n
OpenJDK 64-Bit Server VM warning: Ignoring option --illegal-access=warn; support was removed in 17.0
Setting up the offline depot
Print warnings
----------------------------------------------------------------------------------------------------
                                              WARNING                                               
* Have you configured TCP keepalive in your SSH client to prevent socket connection timeouts when
using the Bundle Transfer Tool for long-running operations?
----------------------------------------------------------------------------------------------------
Validate passwords
Validate the password for VMware depot
Validating the depot user credentials...
Download HCL file
Downloading vSAN HCL attributes to path: /obtu/www/offline_depot/PROD2/vsan/hcl/lastupdatedtime.json
Downloading the vSAN HCL file to path: /obtu/www/offline_depot/PROD2/vsan/hcl/all.json
Successfully completed downloading vSAN HCL file
Download VVS data
User has not set the path using the default path
Directory to download data is existing or created at path /obtu/www/offline_depot/PROD2/evo/vmw/
Download VMware compatibility matrix to directory /obtu/www/offline_depot/PROD2/evo/vmw/Compatibility/VmwareCompatibilityData.json
2024-10-29T13:47:59.122+08:00  INFO   --- [           main] c.v.v.c.c.i.v.rest.client.VvsApiClient   : vvs uri with query params: https://vvs.esp.vmware.com/v1/products/bundles/type/vcf-lcm-v2-bundle?format=json
vvs uri with query params: https://vvs.esp.vmware.com/v1/products/bundles/type/vcf-lcm-v2-bundle?format=json
Successfully downloaded VMWARE_COMPAT compatibility data to file /obtu/www/offline_depot/PROD2/evo/vmw/Compatibility/VmwareCompatibilityData.json
Compatibility metadata has been downloaded, to upload to SDDC Manager use this path as input: /obtu/www/offline_depot/PROD2/evo/vmw/
Modified file permissions to 777 on the file: /obtu/www/offline_depot/PROD2/evo/vmw
Creating delta file
Downloading LCM Manifest to: /obtu/www/offline_depot/PROD2/evo/vmw
Successfully completed downloading file
Default manifest file found, attempting to read into manifest object.
Copping /obtu/www/offline_depot/PROD2/evo/vmw/tmp/index.v3 to /obtu/www/offline_depot/PROD2/evo/vmw/index.v3

List of applicable bundles: 

-------------------------------------------------------------------------------------------------------------------------------------------------
Bundle                               | Product Version  |      Bundle Size | Bundle Component                                   | Bundle Type    
-------------------------------------------------------------------------------------------------------------------------------------------------
bundle-133762                        | 5.2.1.0          |         606.4 MB | ESX_HOST-8.0.3-24280767                            | PATCH          
bundle-133763                        | 5.2.1.0          |        8895.2 MB | NSX_T_MANAGER-4.2.1.0.0-24304122                   | PATCH          
bundle-133760                        | 5.2.1.0          |        2364.8 MB | SDDC_MANAGER_VCF-5.2.1.0-24307856                  | PATCH          
bundle-133761                        | 5.2.1.0          |           0.0 MB | SDDC_MANAGER_VCF-5.2.1.0-24307856                  | PATCH (Drift)  
bundle-133765                        | 5.2.1.0          |       18817.0 MB | VCENTER-8.0.3.00300-24305161                       | PATCH          
bundle-130870                        | 5.2.1.0          |        4238.5 MB | NSX_ALB-22.1.7-24190832                            | INSTALL        
bundle-133764                        | 5.2.1.0          |       11636.7 MB | NSX_T_MANAGER-4.2.1.0.0-24304122                   | INSTALL        
bundle-133766                        | 5.2.1.0          |       11832.3 MB | VCENTER-8.0.3.00300-24305161                       | INSTALL        
-------------------------------------------------------------------------------------------------------------------------------------------------

Created delta file
Total applicable bundles: 8
Starting downloading bundles...
Checking for sufficient disk space before downloading bundles
Available disk space on download directory: /obtu/www/offline_depot/PROD2/evo/vmw is 503734.8 MB
Required disk space to download the bundles is 78231.9 MB
Downloading bundle: bundle-133762.
Downloading bundle: bundle-130870.
Downloading bundle: bundle-133763.
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-130870.manifest
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133762.manifest
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133763.manifest
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-130870.manifest.sig
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133762.manifest.sig
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133763.manifest.sig
Download Progress of bundle tar : bundle-133762.tar : 0.1 MB, Average Speed: 1.01 Mbps, Total Size:  : 606.4 MB
Download Progress of bundle tar : bundle-130870.tar : 0.2 MB, Average Speed: 0.63 Mbps, Total Size:  : 4238.5 MB
Download Progress of bundle tar : bundle-133763.tar : 0.0 MB, Average Speed: 0.30 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-133762.tar : 28.0 MB, Average Speed: 13.03 Mbps, Total Size:  : 606.4 MB
Download Progress of bundle tar : bundle-130870.tar : 31.3 MB, Average Speed: 13.88 Mbps, Total Size:  : 4238.5 MB
Download Progress of bundle tar : bundle-133763.tar : 25.3 MB, Average Speed: 11.91 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-130870.tar : 96.0 MB, Average Speed: 15.17 Mbps, Total Size:  : 4238.5 MB
Download Progress of bundle tar : bundle-133762.tar : 92.3 MB, Average Speed: 14.74 Mbps, Total Size:  : 606.4 MB
Download Progress of bundle tar : bundle-133763.tar : 75.4 MB, Average Speed: 12.29 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-133762.tar : 185.3 MB, Average Speed: 12.97 Mbps, Total Size:  : 606.4 MB
Download Progress of bundle tar : bundle-133763.tar : 147.9 MB, Average Speed: 10.46 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-130870.tar : 200.0 MB, Average Speed: 13.88 Mbps, Total Size:  : 4238.5 MB
Download Progress of bundle tar : bundle-130870.tar : 352.9 MB, Average Speed: 11.59 Mbps, Total Size:  : 4238.5 MB
Download Progress of bundle tar : bundle-133762.tar : 271.9 MB, Average Speed: 8.92 Mbps, Total Size:  : 606.4 MB
Download Progress of bundle tar : bundle-133763.tar : 246.8 MB, Average Speed: 8.11 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-130870.tar : 789.4 MB, Average Speed: 12.64 Mbps, Total Size:  : 4238.5 MB
Download Progress of bundle tar : bundle-133762.tar : 539.2 MB, Average Speed: 8.61 Mbps, Total Size:  : 606.4 MB
Download Progress of bundle tar : bundle-133763.tar : 542.7 MB, Average Speed: 8.68 Mbps, Total Size:  : 8895.2 MB
Bundle bundle-133762. checksum validation successful
Successfully downloaded bundle: bundle-133762.
Completed downloading:1 of total:8
Downloading bundle: bundle-133764.
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133764.manifest
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133764.manifest.sig
Download Progress of bundle tar : bundle-133764.tar : 0.0 MB, Average Speed: 0.30 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-133764.tar : 24.1 MB, Average Speed: 10.54 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-133764.tar : 78.7 MB, Average Speed: 12.52 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-133764.tar : 202.3 MB, Average Speed: 14.12 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-133764.tar : 440.4 MB, Average Speed: 14.49 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-130870.tar : 1804.4 MB, Average Speed: 14.73 Mbps, Total Size:  : 4238.5 MB
Download Progress of bundle tar : bundle-133763.tar : 1413.1 MB, Average Speed: 11.53 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-133764.tar : 860.0 MB, Average Speed: 13.78 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-130870.tar : 2708.9 MB, Average Speed: 14.85 Mbps, Total Size:  : 4238.5 MB
Download Progress of bundle tar : bundle-133763.tar : 2370.4 MB, Average Speed: 12.98 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-133764.tar : 1687.3 MB, Average Speed: 13.78 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-130870.tar : 3587.8 MB, Average Speed: 14.80 Mbps, Total Size:  : 4238.5 MB
Download Progress of bundle tar : bundle-133763.tar : 3268.9 MB, Average Speed: 13.47 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-133764.tar : 2558.2 MB, Average Speed: 14.02 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-133763.tar : 4242.8 MB, Average Speed: 14.01 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-133764.tar : 3454.6 MB, Average Speed: 14.25 Mbps, Total Size:  : 11636.7 MB
Bundle bundle-130870. checksum validation successful
Successfully downloaded bundle: bundle-130870.
Completed downloading:2 of total:8
Downloading bundle: bundle-133760.
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133760.manifest
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133760.manifest.sig
Download Progress of bundle tar : bundle-133760.tar : 0.0 MB, Average Speed: 0.00 Mbps, Total Size:  : 2364.8 MB
Download Progress of bundle tar : bundle-133760.tar : 15.9 MB, Average Speed: 7.66 Mbps, Total Size:  : 2364.8 MB
Download Progress of bundle tar : bundle-133760.tar : 76.1 MB, Average Speed: 12.22 Mbps, Total Size:  : 2364.8 MB
Download Progress of bundle tar : bundle-133760.tar : 173.6 MB, Average Speed: 12.20 Mbps, Total Size:  : 2364.8 MB
Download Progress of bundle tar : bundle-133760.tar : 404.7 MB, Average Speed: 13.36 Mbps, Total Size:  : 2364.8 MB
Download Progress of bundle tar : bundle-133763.tar : 5058.0 MB, Average Speed: 13.94 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-133764.tar : 4270.7 MB, Average Speed: 14.12 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-133760.tar : 875.0 MB, Average Speed: 14.04 Mbps, Total Size:  : 2364.8 MB
Download Progress of bundle tar : bundle-133763.tar : 5990.9 MB, Average Speed: 14.16 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-133764.tar : 5109.3 MB, Average Speed: 14.09 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-133760.tar : 1754.4 MB, Average Speed: 14.34 Mbps, Total Size:  : 2364.8 MB
Download Progress of bundle tar : bundle-133763.tar : 6931.3 MB, Average Speed: 14.35 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-133764.tar : 5963.6 MB, Average Speed: 14.11 Mbps, Total Size:  : 11636.7 MB
Bundle bundle-133760. checksum validation successful
Successfully downloaded bundle: bundle-133760.
Completed downloading:3 of total:8
Downloading bundle: bundle-133761.
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133761.manifest
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133761.manifest.sig
Download Progress of bundle tar : bundle-133761.tar : 0.0 MB, Average Speed: 0.73 Mbps, Total Size:  : 0.0 MB
Bundle bundle-133761. checksum validation successful
Successfully downloaded bundle: bundle-133761.
Completed downloading:4 of total:8
Downloading bundle: bundle-133765.
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133765.manifest
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133765.manifest.sig
Download Progress of bundle tar : bundle-133765.tar : 0.0 MB, Average Speed: 0.00 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133765.tar : 1.2 MB, Average Speed: 0.61 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133765.tar : 21.3 MB, Average Speed: 3.41 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133763.tar : 7993.1 MB, Average Speed: 14.71 Mbps, Total Size:  : 8895.2 MB
Download Progress of bundle tar : bundle-133765.tar : 61.2 MB, Average Speed: 4.29 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133764.tar : 6899.8 MB, Average Speed: 14.30 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-133765.tar : 163.7 MB, Average Speed: 5.41 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133765.tar : 405.1 MB, Average Speed: 6.50 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133764.tar : 8036.1 MB, Average Speed: 14.80 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-133765.tar : 1105.1 MB, Average Speed: 9.03 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133764.tar : 9371.9 MB, Average Speed: 15.55 Mbps, Total Size:  : 11636.7 MB
Bundle bundle-133763. checksum validation successful
Successfully downloaded bundle: bundle-133763.
Completed downloading:5 of total:8
Downloading bundle: bundle-133766.
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133766.manifest
Deleted the temp dir Manifest File  /obtu/www/offline_depot/PROD2/evo/vmw/tmp/manifests/bundle-133766.manifest.sig
Download Progress of bundle tar : bundle-133766.tar : 0.0 MB, Average Speed: 0.00 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133766.tar : 1.1 MB, Average Speed: 0.52 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133766.tar : 13.8 MB, Average Speed: 2.26 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133766.tar : 42.7 MB, Average Speed: 2.97 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 1593.1 MB, Average Speed: 8.73 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 117.2 MB, Average Speed: 3.84 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133764.tar : 10309.0 MB, Average Speed: 15.55 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-133766.tar : 372.5 MB, Average Speed: 5.96 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 2166.9 MB, Average Speed: 8.94 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133764.tar : 11591.9 MB, Average Speed: 16.03 Mbps, Total Size:  : 11636.7 MB
Download Progress of bundle tar : bundle-133766.tar : 1031.6 MB, Average Speed: 8.42 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 2858.3 MB, Average Speed: 9.45 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 1724.6 MB, Average Speed: 9.45 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 3561.3 MB, Average Speed: 9.83 Mbps, Total Size:  : 18817.0 MB
Bundle bundle-133764. checksum validation successful
Successfully downloaded bundle: bundle-133764.
Completed downloading:6 of total:8
Download Progress of bundle tar : bundle-133766.tar : 2419.9 MB, Average Speed: 9.98 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 4269.3 MB, Average Speed: 10.11 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 3106.3 MB, Average Speed: 10.27 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 4887.9 MB, Average Speed: 10.13 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 3768.0 MB, Average Speed: 10.39 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 5576.8 MB, Average Speed: 10.28 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 4469.1 MB, Average Speed: 10.58 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 6270.1 MB, Average Speed: 10.41 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 5162.3 MB, Average Speed: 10.70 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 6985.8 MB, Average Speed: 10.55 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 5856.8 MB, Average Speed: 10.79 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 7682.6 MB, Average Speed: 10.63 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 6552.3 MB, Average Speed: 10.87 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 8381.2 MB, Average Speed: 10.71 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 7242.4 MB, Average Speed: 10.93 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 9086.1 MB, Average Speed: 10.79 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 7932.4 MB, Average Speed: 10.98 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 9790.5 MB, Average Speed: 10.85 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 8623.1 MB, Average Speed: 11.02 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 10501.3 MB, Average Speed: 10.91 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 9308.9 MB, Average Speed: 11.05 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 11092.7 MB, Average Speed: 10.85 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 9989.4 MB, Average Speed: 11.07 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 11792.1 MB, Average Speed: 10.89 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 10685.3 MB, Average Speed: 11.10 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 12490.6 MB, Average Speed: 10.93 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133766.tar : 11374.2 MB, Average Speed: 11.12 Mbps, Total Size:  : 11832.3 MB
Download Progress of bundle tar : bundle-133765.tar : 13189.7 MB, Average Speed: 10.97 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133765.tar : 13890.8 MB, Average Speed: 11.00 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133765.tar : 14599.4 MB, Average Speed: 11.04 Mbps, Total Size:  : 18817.0 MB
Bundle bundle-133766. checksum validation successful
Successfully downloaded bundle: bundle-133766.
Completed downloading:7 of total:8
Download Progress of bundle tar : bundle-133765.tar : 15298.6 MB, Average Speed: 11.06 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133765.tar : 16003.4 MB, Average Speed: 11.09 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133765.tar : 16709.6 MB, Average Speed: 11.12 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133765.tar : 17418.8 MB, Average Speed: 11.15 Mbps, Total Size:  : 18817.0 MB
Download Progress of bundle tar : bundle-133765.tar : 18121.5 MB, Average Speed: 11.17 Mbps, Total Size:  : 18817.0 MB
Bundle bundle-133765. checksum validation successful
Successfully downloaded bundle: bundle-133765.
Completed downloading:8 of total:8
Successfully completed downloading all VSRN bundles to directory path: /obtu/www/offline_depot/PROD2/evo/vmw
Log file: /var/log/vmware/vcf/lcm/tools/debugtool/tmp/debuglog/lcmdebug.log
[root@vcf-obtu bin]# 
```

最终，你将得到如下所示的目录结构。



```
[root@vcf-obtu ~]# tree /obtu/www/offline_depot/
/obtu/www/offline_depot/
└── PROD2
    ├── evo
    │   └── vmw
    │       ├── bundles
    │       │   ├── bundle-130870.tar
    │       │   ├── bundle-133760.tar
    │       │   ├── bundle-133761.tar
    │       │   ├── bundle-133762.tar
    │       │   ├── bundle-133763.tar
    │       │   ├── bundle-133764.tar
    │       │   ├── bundle-133765.tar
    │       │   └── bundle-133766.tar
    │       ├── Compatibility
    │       │   └── VmwareCompatibilityData.json
    │       ├── deltaFileDownloaded
    │       ├── deltaFileDownloaded.md5
    │       ├── index.v3
    │       ├── lcm
    │       │   └── manifest
    │       │       └── v1
    │       │           └── lcmManifest.json
    │       ├── manifests
    │       │   ├── bundle-130870.manifest
    │       │   ├── bundle-130870.manifest.sig
    │       │   ├── bundle-133760.manifest
    │       │   ├── bundle-133760.manifest.sig
    │       │   ├── bundle-133761.manifest
    │       │   ├── bundle-133761.manifest.sig
    │       │   ├── bundle-133762.manifest
    │       │   ├── bundle-133762.manifest.sig
    │       │   ├── bundle-133763.manifest
    │       │   ├── bundle-133763.manifest.sig
    │       │   ├── bundle-133764.manifest
    │       │   ├── bundle-133764.manifest.sig
    │       │   ├── bundle-133765.manifest
    │       │   ├── bundle-133765.manifest.sig
    │       │   ├── bundle-133766.manifest
    │       │   └── bundle-133766.manifest.sig
    │       └── tmp
    │           ├── index.v3
    │           └── lcmManifest.json
    └── vsan
        └── hcl
            ├── all.json
            └── lastupdatedtime.json

12 directories, 33 files
[root@vcf-obtu ~]#
```

 



## 五、配置 VCF 脱机库



登陆 SDDC Manager UI，导航到管理\-\>库设置，VMware Cloud Foundation 5\.2 及之后的版本可以在这里进行脱机库设置。如果是 VMware Cloud Foundation 5\.1 及之前的版本，请参考文章开头提到的知识库和博客文章，了解具体操作步骤。


[![](https://img2024.cnblogs.com/blog/2313726/202410/2313726-20241029144201577-1337426077.png)](https://github.com)


点击“脱机库”中的设置，填入 OBTU 服务器的地址等信息，然后完成设置。


[![](https://img2024.cnblogs.com/blog/2313726/202410/2313726-20241029144246399-1721595380.png)](https://github.com)


脱机库设置成功。


[![](https://img2024.cnblogs.com/blog/2313726/202410/2313726-20241029144314551-209281427.png)](https://github.com)


稍等片刻后，导航到生命周期管理\-\>包管理，你应该能看到脱机库中所有可用的软件包。参考文章“[更新 VCF 5\.1 至 VCF 5\.2 版本。](https://github.com)”或“[独立更新 SDDC Manager 组件的版本。](https://github.com)”执行后续操作。


[![](https://img2024.cnblogs.com/blog/2313726/202410/2313726-20241029144923319-1722792899.png)](https://github.com)


