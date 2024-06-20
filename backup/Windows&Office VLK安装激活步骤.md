# Office官网VLK安装

**1.office 软件部署工具：https://www.microsoft.com/en-us/download/details.aspx?id=49117**

**2.office 版本自定义工具：https://config.office.com/deploymentsettings**

**3.基于KMS的 GVLK：https://learn.microsoft.com/zh-cn/deployoffice/vlactivation/gvlks**

**4.通过官方下载安装批量授权版：**

**下载命令：**

setup /download config.xml

**安装命令:**

setup /configure config.xml

**office 激活命令: (CMD以管理员运行)**

cscript ospp.vbs /sethst:kms.03k.org

cscript ospp.vbs /act

# win10/11 专业版激活

https://www.microsoft.com/zh-cn/software-download/windows11

https://learn.microsoft.com/zh-cn/windows-server/get-started/kms-client-activation-keys

slmgr /ipk W269N-WFGWX-YVC9B-4J6C9-T83GX

slmgr /skms [kms.03k.org](http://kms.03k.org/)

slmgr /ato