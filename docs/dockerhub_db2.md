# intro
* 聽說銀行業界使用DB2比較多，但是我其實只在學生時期聽過DB2。現在container 的時代已經來臨了，DB2還是優先透過container 啟動吧。

* 馬上找到ibm container db2 的文章，直接使用吧。[Installing the Db2 Community Edition Docker image on Windows systems](https://www.ibm.com/docs/en/db2/11.5?topic=SSEPGG_11.5.0/com.ibm.db2.luw.qb.server.doc/doc/t_install_db2CE_win_img.html)

* 流程也相當簡單，使用 `.env_list` 的方式啟動，也不需要調整甚麼。

* 透過 `docker logs db2server`，也看到啟動的過程非常順利。

```
The execution completed successfully.

For more information see the DB2 installation log at "/tmp/db2icrt.log.71".
DBI1446I  The db2icrt command is running.


DBI1070I  Program db2icrt completed successfully.
```

* 很高興地打開 `dbeaver` 連上 db2，localhost/testdb, db2inst1/password，結果跳出了錯誤訊息`DB2 Connection refused: connect。 ERRORCODE=-4499, SQLSTATE=08001`。去查了一下IBM的官網文件，大概的意思就是請遵循官方文件再次設定，直指防火牆問題。另外因為我是透過docker啟動的，也有人指出是docker network 的問題。因為AP 和 DB 的路由不通，需要透過bridge 設定才能打通。為此我還把docker network 的狀態查詢了一便。但是仍然無解。

* 我從 container db2 連過去其實也是順暢的。有明確看到db2 database & tables 。就只是外面連不進去。

* 結果查到這篇文章，找的流程很棒推薦給大家。[DB2 Connection refused: connect。 ERRORCODE=-4499, SQLSTATE=08001_m0_37607945](http://www.cxyzjd.com/article/m0_37607945/105573158)

# 問題處理

* 先查看docker port 是否開放，明顯有`0.0.0.0:50000->50000/tcp`
```
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              POR                                  NAMES
3ee57359b00d        ibmcom/db2:11.5.0.0   "/var/db2_setup/lib/…"   23 hours ago        Up 10 minutes       22/7/tcp, 0.0.0.0:50000->50000/tcp   mydb2
```

* 再查看db2 服務使用的port號, 結果發現新大陸 `db2c_db2inst1      25010/tcp`，甚麼??結果居然開在 25010 !!!!
```
[bash] docker exec -it db2server bash -c "su - db2inst1"

[db2inst1@3ee57359b00d ~]$ db2 get dbm cfg|grep SVCENAME
 TCP/IP Service name                          (SVCENAME) = db2c_db2inst1
 SSL service name                         (SSL_SVCENAME) =

[db2inst1@3ee57359b00d ~]$ cat /etc/services | grep db2c_db2inst1
db2c_db2inst1      25010/tcp
```

# 修正啟動語法
* 官方文件寫錯，真的害死人。不過基礎的image可能隨時也會調整，目前發現這個問題是在tag `11.5.6.0`。當然在此建議下，請大家改連 25010。另外應該也不會另外製作 image 去更改 port 設定。如果未來有需要特別把port num 改為 50000的話，可能再透過 image 製作的方式重新啟動服務的 port 號。

* docker startup
```
docker run --name db2server --restart=always --detach --privileged=true -p 25010:25010 -p 50000:50000 --env-file .env_list -v $(pwd)/database:/database ibmcom/db2:docker pull ibmcom/db2:11.5.6.0
```
* .evn_list 參考
```
LICENSE=accept
DB2INSTANCE=db2inst1
DB2INST1_PASSWORD=password
DBNAME=testdb
BLU=false
ENABLE_ORACLE_COMPATIBILITY=false
UPDATEAVAIL=NO
TO_CREATE_SAMPLEDB=false
REPODB=false
IS_OSXFS=false
PERSISTENT_HOME=false
HADR_ENABLED=false
ETCD_ENDPOINT=
ETCD_USERNAME=
ETCD_PASSWORD=
```