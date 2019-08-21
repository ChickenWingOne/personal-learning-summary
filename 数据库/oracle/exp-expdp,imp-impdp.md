exp/imp命令没有expdp/impdp命令快，expdp/impdp属于数据泵<br/>
exp/imp：
```
exp username/password@ip:port/service owner=username file= log=
imp username/password@ip:port/service full=y file= log= 
```
expdp/impdp：该命令需要用到directory，使用命令之前需要先创建directory，且oracle用户对其具有读写的权限，备份的文件和日志都会放在这里面
```
//oracle管理员登录sqlplus
1.sqlplus username/password
//创建directory
2.create directory dump_dir as '存放dmp文件的目录（该目录一定要事先创建好）';
//赋读写权限
3.grant read,write on directory dump_dir to username;
//执行命令
4.exmpdp username/username@ip:port/service schemas=username directory=dump_dir dumpfile= logfile=
```
### 注意项 ###
exmpdp如果就使用上面的命令，导出的dmp文件中，会包含了创建用户的sql，使用impdp时就会创建用户，该用户已存在的话就会报错。<br/>
<b>expdp命令可加入exclude=user，生成的dmp文件就不会包含创建用户的sql了。</b>