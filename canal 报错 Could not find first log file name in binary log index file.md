# canal 报错 Could not find first log file name in binary log index file

__解决方案__  
先停止canal server  
删除 canal/conf/example/meta.dat  
查询数据库 `show master status;`  

<img width="592" height="113" alt="image" src="https://github.com/user-attachments/assets/67fdfa16-272b-4cb8-9191-89ee7adc1e69" />

更新 canal/conf/example/instance.properties文件中  
```
canal.instance.master.journal.name=xxx
canal.instance.master.position=xxx
```
重启 canal server
