redis有两种持久化方式
1.快照 
fork出来一个子进程来负责快照的保存，fork耗内存，但性能好。一般5分钟一次fork
2.append
追加记录对redis进行持久化：always,everysec,no
lsof -p 9085  查看文件描述符的号码
strace -f -T -tt -p 9085 查看系统调用，结果配合lsof可得出 系统调用情况

查看io strace...这种东西
sudo strace -f -p $(pidof redis-server) -T -e trace=fdatasync,write 2>&1 | grep -v '0.0' | grep -v unfinished


前段时间刚找到一个由于内存数据被交换到swap文件中导致内存数据遍历效率变低的问题
问题定位过程是使用pidstat命令发现进程cpu使用率变低 mpstat命令观察到系统iowait升
高 由此怀疑跟io有什么关系 perf命令观察发现内存数据遍历过程中swap相关调用时间占
比有点异常 然后使用pidstat命令+r参数 也观察到进程在那段时间主缺页中断升高 由此确
定问题

url:
http://cache.baiducontent.com/c?m=9f65cb4a8c8507ed19fa950d100b92235c4380146d8b804b2281d25f93130a1c187ba9e0747f4e538892227a43b2435de8f23d71341420c0c18ed714c9fecf68798767632647c75613a30edebf5152b037e65bfedc1df0bb8025e5adc5a2ab4325ce44737097f1fb4d7617dd19f10341e2b1ef43025f61ad9c3a72885c6059ec3431c150f890251e719681dd4b3cb43da465&p=9c36c64ad4d511a05bed9427155683&newp=996ec54ad5c341e610b5c7710f5185231610db2151d3d001298ffe0cc4241a1a1a3aecbf2d241100d8c6776101aa4d5beef73c703d0034f1f689df08d2ecce7e66c07f&user=baidu&fm=sc&query=Redis%CF%EC%D3%A6%D1%CF%D6%D8%D1%D3%B3%D9%2C%C8%E7%BA%CE%BD%E2%BE%F6%3F&qid=dee0e2c40013484a&p1=1