- 1.清空yum:yum clean all  
- 2.重启服务：systemctl restart 服务名
- 3.设置开机自启动：systemctl enable 服务名
- 4.使配置文件生效：systemctl daemon-reload
- 5.关闭防火墙：systemctl disable firewalld
			   systemctl stop firewalld
- 6.开启防火墙：systemctl start firewalld
- 7.开端口：firewall-cmd --zone=public --add-port=8080/tcp --permanent
- 8.提交端口修改：firewall-cmd --reload
- 9.命令关机：poweroff
- 10.命令重启：reboot 
- 11.安装vim编辑器：yum -y install vim*
   - vim /etc/vimrc添加： 
        set showmode    " 设置在命令行界面最下面显示当前模式等
        set ruler       " 在右下角显示光标所在的行数等信息
        set autoindent  " 设置每次单击Enter键后，光标移动到下一行时与上一行的起始字符对齐
        syntax on       " 即设置语法检测，当编辑C或者Shell脚本时，关键字会用特殊颜色显示

