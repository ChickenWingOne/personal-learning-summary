```
查看锁表进程SQL语句2： 
select * from v$session t1, v$locked_object t2 where t1.sid = t2.SESSION_ID; 

杀掉锁表进程： 
如有記錄則表示有lock，記錄下SID和serial# ，將記錄的ID替換下面的738,1429，即可解除LOCK 
alter system kill session '738,1429'; 
```