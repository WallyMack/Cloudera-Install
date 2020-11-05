Ubuntu Install Cloudera Manager 5-14-x
=====================================

Target Release: 1.１


Version Number:           1.1	
Version Date:		2018/06/20  
Author:			Mack.Chuang		
Total pages:	

* * *

Contents
--------
[TOC]

* * *

# 1. 文件資訊(History)
## 1.1 文件制／修訂履歷(Table of Changes)
Modification History

|    Data    |    Author   | Version | Description |
|:----------:|:-----------:|:-------:|:-----------:|
| 2018/06/12 | Mack Chuang |    1    |     初版    |
| 2018/06/20 | Mack Chuang |    1.1    |     新增HAProxy   |

## 1.2 文件存放位置(Document Location)
路徑 :
## 1.3 文件發佈資訊(Distribution)
本文件提供以下的人員閱讀:  
>系統開發人員  
>維運人員  
>資訊部人員  


* * *

# 2. 系統環境(System Design)

各主機配備一覽表

|  HostName  |      IP     |              CPU             | RAM |         OS         |
|:----------:|:-----------:|:----------------------------:|:---:|:------------------:|
| cloudera01 | 10.0.40.101 | Intel(R) Xeon(R) CPU E5-2609 |  8G | Ubuntu 16.04.4 LTS |
| cloudera02 | 10.0.40.102 | Intel(R) Xeon(R) CPU E5-2609 |  8G | Ubuntu 16.04.4 LTS |
| cloudera03 | 10.0.40.103 | Intel(R) Xeon(R) CPU E5-2609 | 16G | Ubuntu 16.04.4 LTS |
| cloudera04 | 10.0.40.104 | Intel(R) Xeon(R) CPU E5-2609 | 16G | Ubuntu 16.04.4 LTS |
|HAProxy VIP | 10.0.40.120|

## 2.1 OS系統環境設置
確認群集內所有主機的IP與主機名稱對應關係(所有節點)，指令如下：
>cat /etc/hosts 

![IP_Host](https://i.imgur.com/pN0BDJM.png)

確認防火牆是否關閉(所有節點)，指令如下：
>sudo service iptables status

![IPTab les](https://i.imgur.com/ZG6UfPs.png)

>關閉IPV6以及設定Swappiness(所有節點)，指令如下：
sudo vim /etc/sysctl.conf

![sysctl_conf](https://i.imgur.com/0ioZHrI.png)

>移動至最後一行，指令如下：
:9999

![last_line](https://i.imgur.com/tsJakbo.png)

>編輯檔案，設定關閉IPV6以及設定Swappiness數值，指令如下：  
按一下鍵盤i(進入編輯模式，左下角出現"INSERT")，並貼上設定  
net.ipv6.conf.all.disable_ipv6=1  
net.ipv6.conf.default.disable_ipv6=1  
net.ipv6.conf.lo.disable_ipv6=1  
vm.swappiness=1  

![off＿ipv6](https://i.imgur.com/xPaxqBi.png)

>按一下鍵盤ESC(離開編輯模式，左下角“INSERT"消失)  
儲存檔案並離開，指令如下：  
:wq  

![exit_vim](https://i.imgur.com/EM7wd6o.png)

>重新讀取設定檔，指令如下：  
sudo sysctl -p  
確認IPV6關閉，指令如下(未出現IPV6代表成功)：  
ifconfig  

![IPV6](https://i.imgur.com/RnopP4d.png)

>優化OS系統硬碟的讀取速度(所有節點)，指令如下：  
sudo vim /etc/rc.local  

![IO](https://i.imgur.com/ES1kLka.png)

>按一下鍵盤i(進入編輯模式，左下角出現"INSERT")，並貼上設定  
blockdev --setra 8192 /dev/sda  
echo deadline > /sys/block/sda/queue/scheduler  
echo never > /sys/kernel/mm/transparent_hugepage/defrag  

![IO_1](https://i.imgur.com/PiHSqjE.png)

>按一下鍵盤ESC(離開編輯模式，左下角“INSERT"消失)  
儲存檔案並離開，指令如下：  
:wq  

安裝NTP服務讓叢集主機時間一致性(所有主機)  
>sudo apt-get install ntp  
sudo apt install ntpdate  
sudo apt install ntpstat  
確認NTP服務是否啟動  
sudo systemctl status ntp.service  

![NTP_1](https://i.imgur.com/Exb2DZ2.png)

>關閉NTP服務  
sudo systemctl stop ntp.service  
確認是否關閉  
systemctl status ntp.service  

![NTP_2](https://i.imgur.com/QP2zDtI.png)

>變更設定檔，指令如下：  
sudo vim /etc/ntp.conf  
/prefer  
按一下鍵盤i(進入編輯模式，左下角出現"INSERT")，並貼上設定  
server 10.0.10.102 prefer maxpoll 6 (兩台ＮameNode主機對時，10.0.10.102 詢問網管公司內部  NTP Server IP)  

![NTP_3](https://i.imgur.com/TxEy9y1.png)

>:wq  
其他主機設定檔如下：  
server 10.0.40.101 prefer   
server 10.0.40.102  

![NTP_4](https://i.imgur.com/HN1ihVu.png)

進行對時，指令如下：  
>sudo ntpdate -u 10.0.10.102 (兩台ＮameNode主機)  
sudo systemctl start ntp.service  

![NTP_5](https://i.imgur.com/egyHGDk.png)

>sudo ntpdate -u 10.0.40.101 (其他主機)  
sudo ntpdate -u 10.0.40.102 (其他主機)  
sudo systemctl start ntp.service  

確認對時主機是否正確，指令如下：  
>ntpstat (兩台ＮameNode主機)  

![NTP_6](https://i.imgur.com/FrRAMhe.png) 

>ntpstat (其他主機)  

![NTP_7](https://i.imgur.com/htXY1rh.png)


* * *

# 3. 安裝資料庫
Cloudera 將許多Config以及MetaData存放於Database裡面，故需安裝外部的資料庫  

步驟如下：  
3.1. 登入主機cloudera01以及cloudera02，指令如下：
>ssh bdadmin@10.0.40.101

3.2.安裝外部資料庫，此次選擇MySQL 5.7版，指令如下：
>sudo apt-get install mysql-server [會去抓取當下OS系統最新版的MySQL版本]

3.3.檢查是否安裝成功，指令如下：
>mysql ---version


![MYSQL_Version](https://i.imgur.com/EpvO4LX.png)

3.4.關閉MySQL Server 確保資料庫沒在運行，指令如下：
>sudo service mysql stop

3.5.根據Cloudera 官方推薦設置MySQL Config，指令如下：
>sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf   
註解 bind-address = 127.0.0.1 增加時間設定參數(讓機器可以遠端存取)，指令如下：  
/bind  
log_timestamps=SYSTEM (預設Log時間為UTC改為系統時間)  

![bind_1](https://i.imgur.com/bXOkc9G.png)

>移動至最後一行，指令如下：
:9999

![MySQL](https://i.imgur.com/rIkzgPV.png)

>按一下鍵盤i(進入編輯模式，左下角出現"INSERT")，並貼上設定  
[mysqld]  
server-id = 1  [兩台MySQL編號不能相同]
transaction-isolation = READ-COMMITTED  
>\#Disabling symbolic-links is recommended to prevent assorted security risks;  
>\#to do so, uncomment this line:  
>\#symbolic-links = 0  
key_buffer_size = 32M  
max_allowed_packet = 32M  
thread_stack = 256K  
thread_cache_size = 64  
query_cache_limit = 8M  
query_cache_size = 64M  
query_cache_type = 1  
max_connections = 550  
>\#expire_logs_days = 10  
>\#max_binlog_size = 100M  
>\#log_bin should be on a disk with enough free space. Replace   '/var/lib/mysql/mysql_binary_log' with an appropriate path for your system  
>\#and chown the specified folder to the mysql user.  
log_bin=/var/lib/mysql/mysql_binary_log  
>\#For MySQL version 5.1.8 or later. For older versions, reference MySQL  documentation for configuration help.  
default-storage-engine=innodb  
binlog_format = mixed  
read_buffer_size = 2M  
read_rnd_buffer_size = 16M  
sort_buffer_size = 8M  
join_buffer_size = 8M  
>\#InnoDB settings  
innodb_file_per_table = 1  
innodb_flush_log_at_trx_commit  = 2  
innodb_log_buffer_size = 64M  
innodb_buffer_pool_size = 4G  
innodb_thread_concurrency = 8  
innodb_flush_method = O_DIRECT  
innodb_log_file_size = 512M  
[mysqld_safe]  
log-error=/var/log/mysqld.log  
pid-file=/var/run/mysqld/mysqld.pid  
sql_mode=STRICT_ALL_TABLES  

![db_1](https://i.imgur.com/SHUUNsy.png)
![db_2](https://i.imgur.com/WiQqHR1.png)
![db_3](https://i.imgur.com/kf2EVxY.png)

3.6.啟動MySQL Server，指令如下：
>sudo service mysql start

3.7.設定ＭySQL管理員密碼，安裝時有設定的話這邊會用到，指令如下：
>sudo /usr/bin/mysql_secure_installation  
**Policy強度建議設為0**  

![db_root](https://i.imgur.com/rmU2a7X.png)
![db_root_2](https://i.imgur.com/uUgKxQJ.png)
![db_root_3](https://i.imgur.com/UqFY2y9.png)

3.8.安裝MySQL JDBC驅動程式，針對Cloudera Manager Server用，指令如下：
>sudo apt-get install libmysql-java

3.9.確認MySQL是否正常，指令如下：
>mysql -uroot -proot123 (登入MySQL)
mysql> show databases; (檢查是否正常查詢)

![db_check](https://i.imgur.com/GYGYUbg.png)

>mysql> show engines; (檢查MySQL是否為InnoDB模式)

![db_check_2](https://i.imgur.com/TUJFczg.png)

>mysql> select host,user from mysql.user; (檢查是否有設定過的帳號)

![db_check_3](https://i.imgur.com/UMfGl13.png)

3.10.建立MySQL複寫專用帳號，指令如下：  
>mysql> CREATE USER 'btsync'@'10.0.40.102' IDENTIFIED BY 'btsync!@#';  
mysql> CREATE USER 'btsync'@'10.0.40.101' IDENTIFIED BY 'btsync!@#';  
mysql> GRANT replication slave ON *.* TO 'btsync'@'10.0.40.102';  
mysql> GRANT replication slave ON *.* TO 'btsync'@'10.0.40.101';  
mysql> FLUSH PRIVILEGES;  
檢查是否建立成功  
mysql> select host,user from mysql.user;  

![create_account](https://i.imgur.com/Y1y9iYQ.png)

3.11.資料庫備份與移轉，指令如下：  
>mysqldump -u root -p --all-databases --events > /opt/all_mysql_db.sql  
scp -p /opt/all_mysql_db.sql bdadmin@Cloudera02:/opt/ (將資料庫備份移轉到第二台上面)  
ssh bdadmin@10.0.40.102 (登入到第二台上，進行資料庫安裝動作至3.9即可)  

3.12.資料庫第二台建置完成，將第一台的資料寫入，指令如下：
>mysql -u root -p --default-character-set=utf8 < /opt/all_mysql_db.sql

![create_second_db](https://i.imgur.com/ItS5atx.png)

3.13.檢查第二台資料庫：
>mysql -uroot -proot123 
select host,user from mysql.user;

![second_db](https://i.imgur.com/qyCgx4s.png)

* * *

# 4. 建置MySQL Replication(Master-Master)

4.1. 登入第一台MySQL
>mysql -uroot -proot123  
mysql> show master status; (查詢Master資訊)  

![Master_info_1](https://i.imgur.com/OKbqEv0.png)

>設定Ｍaster 資料庫為第二台MySQL (master_log_file &ß master_log_pos 請根據第二台Master資訊填寫)  
mysql> change master to master_host='10.0.40.102',master_port=3306,  master_user='btsync',master_password='btsync!@#',   master_log_file='mysql_binary_log.000002',master_log_pos=784436;  

>啟動Slave  
mysql> start slave;  
檢查Slave資訊以及狀態  
mysql> show slave status \G; (正確設定後，無提示任何錯誤訊息，便讓第一台與第二台同步成功)  

![Master_info_1](https://i.imgur.com/q5a0NuJ.png)
![Master_info_2](https://i.imgur.com/VBphEWn.png)
![Master_info_3](https://i.imgur.com/ZSwHC99.png)

4.2. 登入第二台MySQL
>mysql -uroot -proot123
mysql> show master status \G; (查詢Master資訊)

![Slave_info_1](https://i.imgur.com/dExfWVo.png)

>設定Ｍaster 資料庫為第一台MySQL (master_log_file &ß master_log_pos 請根據第一台Master資訊填寫)  
mysql> change master to master_host='10.0.40.101',master_port=3306,  master_user='btsync',master_password='btsync!@#',  master_log_file='mysql_binary_log.000002',master_log_pos=771;  

>啟動Slave  
mysql> start slave;  
檢查Slave資訊以及狀態  
mysql> show slave status \G; (正確設定後，無提示任何錯誤訊息，便讓第二台與第一台同步成功)  

![Slave_Info](https://i.imgur.com/6I82Qua.png)
![Slave_info_1](https://i.imgur.com/RWZHgDt.png)
![Slave_info_2](https://i.imgur.com/ISAk5UC.png)

此時MySQL Replication(Master-Master)完成


* * *

# 5. 建置Cloudera Manager需要的資料庫

5.1. 於Cloudera01主機登入，指令如下：
>ssh bdadmin@10.0.40.101  
mysql -uroot -proot123  
mysql> create database amon DEFAULT CHARSET utf8; (Activity Monitor)  
mysql> create database rman DEFAULT CHARSET utf8; (Report Manager)  
mysql> create database metastore DEFAULT CHARSET utf8; (Hive Metastore Server)  
mysql> create database oozie DEFAULT CHARSET utf8; (Oozie Metastore Server)  

5.2. 於Cloudera02主機登入，指令如下：
>ssh bdadmin@10.0.40.102  
mysql -uroot -proot123  
mysql> show databases; (驗證Cloudera01建立的DB是否有複製過來，有的話Replication建置成功)  

![db](https://i.imgur.com/FHenoMC.png)

5.3. 授權Cloudera帳號可存取Cloudera所需的資料庫，指令如下：  
>mysql> grant all on *.* to 'root'@'10.0.40.101' identified by 'root1234' with grant  option;  
mysql> grant all on *.* to 'root'@'10.0.40.102' identified by 'root1234' with grant option;  

>mysql> grant all on metastore.* TO 'hive'@'10.0.40.101' identified by 'hive!@#$';  

>mysql> grant all on metastore.* TO 'hive'@'10.0.40.102' identified by 'hive!@#$';  

>mysql> grant all on amon.* TO 'amon'@'10.0.40.101' identified by 'amon!@#$';  

>mysql> grant all on rman.* TO 'rman'@'10.0.40.101' identified by 'rman!@#$';  
mysql> grant all privileges on oozie.* to 'oozie'@'localhost' identified by 'oozie!@#';  
mysql> grant all privileges on oozie.* to 'oozie'@'10.0.40.101' identified by 'oozie!@#';  
mysql> grant all privileges on oozie.* to 'oozie'@'10.0.40.102' identified by 'oozie!@#';  
mysql> FLUSH PRIVILEGES;  

5.4. 於Cloudera01主機登入，指令如下：
>ssh bdadmin@10.0.40.101  
mysql -uroot -proot123  
mysql> select host,user from mysql.user; (檢查使用者是否新增成功)  

![user_account](https://i.imgur.com/Nc3q2eN.png)

5.5. 於預計安裝Hue Server的主機建立相關資料庫(本次安裝在Cloudera02上)
>ssh bdadmin@10.0.40.102  
mysql -uroot -proot123  
mysql> create database hue default character set utf8 default collate utf8_general_ci; (Hue Server)  
mysql> grant all on hue.* to 'hue'@'10.0.40.102' identified by 'hue!@#$%';  
mysql> select * from information_schema.schemata; (檢查是否建立成功且為UTF8格式)  

![Hue_Server_1](https://i.imgur.com/wtfDWKn.png)

5.6. 將mysql-JDBC放置在Hive以及Oozie相關目錄(兩台資料庫主機上)
>dpkg -L libmysql-java

![JDBC_1](https://i.imgur.com/giUSm0i.png)

>將JDBC放置在Hive以及Oozie目錄內  
(Hive)  
sudo cp -p /usr/share/java/mysql-connector-java-5.1.38.jar   /opt/cloudera/parcels/CDH-5.15.0-1.cdh5.15.0.p0.21/lib/hive/lib/  

>(Oozie)  
sudo cp -p /usr/share/java/mysql-connector-java-5.1.38.jar   
/opt/cloudera/parcels/CDH-5.15.0-1.cdh5.15.0.p0.21/lib/oozie/lib  

* * *

# 6. 建置Cloudera Manager

建置Cloudera Manager 載點於第一台上
   
6.1. 根據OS版本，至Cloudera 官網找對應的載點，步驟如下：  
>ssh bdadmin@10.0.40.101   
lsb_release -a （查詢OS版本）  

![OS_Version](https://i.imgur.com/UgmT4ni.png)


6.2. 至官方網站查找相對應載點 (所有主機都要做)  
>從載點網址找到Repo File欄位對應的OS版本，下載後將內容複製起來，之後會貼在cloudera-manager.list檔案內  
[Cloudera 載點](https://www.cloudera.com/documentation/enterprise/release-notes/topics/cm_vd.html)  
於底下目錄 /etc/apt/sources.list.d/ 新增 cloudera-manager.list 檔案  
touch /etc/apt/sources.list.d/cloudera-manager.list  
sudo vim etc/apt/sources.list.d/cloudera-manager.list  
按一下鍵盤i(進入編輯模式，左下角出現"INSERT")，並貼上設定  
\# Packages for Cloudera Manager, Version 5, on Ubuntu 16.04 amd64       
deb [arch=amd64] http://archive.cloudera.com/cm5/ubuntu/xenial/amd64/cm xenial-cm5 contrib  
deb-src http://archive.cloudera.com/cm5/ubuntu/xenial/amd64/cm xenial-cm5 contrib  

![Cloudera_List](https://i.imgur.com/k98Q3k1.png)

>sudo apt-get update  
如果發生金鑰問題請執行以下指令：  
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 327574EE02A818DD  
（327574EE02A818DD 請根據報錯的KEY填寫）  

![Error_Log](https://i.imgur.com/4gi8Ly2.png)

>之後再重新更新一次即可  

6.3. 安裝Cloudera Manager 
>sudo apt-get install oracle-j2sdk1.7 (安裝Java JDK 所有主機)  
sudo apt-get install cloudera-manager-daemons cloudera-manager-server (安裝Manager Server 一台即可)  

6.4. 設定Cloudera Manager Database  
於安裝Manager Server的主機上執行以下指令：  
>sudo /usr/share/cmf/schema/scm_prepare_database.sh mysql -h 10.0.40.101 -uroot -proot1234 --scm-host 10.0.40.101 scm scm scm12345  

![Cloudera_DB](https://i.imgur.com/DssL6o8.png)

6.5. 啟動Cloudera Manager
>sudo systemctl start cloudera-scm-server.service   
檢查是否有誤  
sudo systemctl status cloudera-scm-server.service  

![Cloudera_Manager](https://i.imgur.com/yZV6j3H.png)

6.6. 登入Cloudera Manager Admin Console
打開習慣的瀏覽器，輸入Cloudera Manager主機的IP，指令如下：
>http://10.0.40.101:7180/  
預設帳號及密碼：admin  

![CM_Admin_Console](https://i.imgur.com/Bp6Jggx.png)

>進入安裝畫面，如下：  
打勾之後點擊Continue  

![CM_!](https://i.imgur.com/DjhRhDZ.png)

>選取相關安裝方式，此次使用免費版Cloudera Express，點擊後選Continue

![CM_3](https://i.imgur.com/80jNcrL.png)

>確認選取的方式是否正確，確認無誤後點擊Continue

![CM_4](https://i.imgur.com/6Jjrwi0.png)

>設定要安裝的主機IP後點擊Search  
10.0.40.101,10.0.40.102,10.0.40.103,10.0.40.104 (IP之間用逗號隔開)  

![CM_5](https://i.imgur.com/RY2VCtL.png)

>顯示設定搜尋的所有主機，確認無誤後點擊Continue

![CM_6](https://i.imgur.com/ZJGaegb.png)

>確認版本後，點擊Continue

![CM_7](https://i.imgur.com/ZsrGT5o.png)

>打勾JDK7後，點擊Continue

![CM_8](https://i.imgur.com/SIeDUuG.png)

>不使用Single User Mode，直接點擊Continue

![CM_9](https://i.imgur.com/m5RmNYb.png)

>使用root或使用sudo指令不用輸入密碼的帳號，並且輸入帳號的密碼，完成後點擊Continue

![CM_10](https://i.imgur.com/RUcbttF.png)

>Cluster安裝中

![CM_11](https://i.imgur.com/f3MG9tO.png)

>安裝完成，Continue變成藍色，點擊Continue

![CM_12](https://i.imgur.com/NTREOCN.png)

>Cloudera自身檢測驗證安裝是否完成

![CM_13](https://i.imgur.com/5Yle7wi.png)

>安裝完成後點擊Continue

![CM_14](https://i.imgur.com/c642wPj.png)

>檢查所有套件全打勾代表正常安裝結束點擊Finish

![CM_15](https://i.imgur.com/3iLij7D.png)

>開始安裝Hadoop Ecosystem，選擇Custom Services，點擊Continue

![CM_16](https://i.imgur.com/EYjD3Fs.png)

>配置Hadoop Ecosystem到規劃好的主機上

![CM_17](https://i.imgur.com/dMpCh2f.png)

>設定Hadoop Ecosystem存取的資料庫，相關資訊參考目錄第五點

![CM_18](https://i.imgur.com/0EisKyP.png)
![CM_19](https://i.imgur.com/bBdtjf3.png)

>服務相關設定，檢查Data Directory是否正確以及設定Kudu，點擊Continue  
Kudu Master WAL Directory：/var/lib/kudu/master  
Kudu Master Data Directories：/var/lib/kudu/master  
Kudu Tablet Server WAL Directory：/var/lib/kudu/tserver  
Kudu Tablet Server Data Directories：/var/lib/kudu/tserver  

![CM_20](https://i.imgur.com/6BSyfhz.png)
![CM_21](https://i.imgur.com/NjZpBXL.png)
![CM_22](https://i.imgur.com/VVXxP5t.png)

>安裝完成，Continue變成藍色，點擊Continue

![CM_23](https://i.imgur.com/VU9cS0G.png)
![CM_24](https://i.imgur.com/ZfSdsT6.png)

>安裝結束，點擊Finish

![CM_25](https://i.imgur.com/jdenSwp.png)

* * *

# 7. 啟用HDFS HA機制
透過Cloudera Manager頁面，點擊HDFS
![HDFS_HA](https://i.imgur.com/bLmFKLJ.png)
點擊Actions下拉式選單，選取Enable High Availability
![HDFS_HA_1](https://i.imgur.com/j5aJKOM.png)
替Nameservice取名或使用預設，點擊Continue
![HDFS_HA_2](https://i.imgur.com/QtU7LWA.png)
選擇第二台NameNode的主機以及同步NameNode Metadata的JournalNode三台，點擊Continue
![HDFS_HA_3](https://i.imgur.com/V8CI6iA.png)
![HDFS_HA_4](https://i.imgur.com/vcRRasF.png)
設定NameNode Metadata的目錄，點擊Continue
![HDFS_HA_5](https://i.imgur.com/KT6qUML.png)
設定開始，正常情況下Formate The Name Directory會失敗，點擊Continue
![HDFS_HA_6](https://i.imgur.com/lWjD3OV.png)
![HDFS_HA_7](https://i.imgur.com/hcGnP2J.png)
設定完成，點擊Finish
![HDFS_HA_8](https://i.imgur.com/C0iIBzJ.png)

* * *

# 8. 啟用Hive To Use HDFS HA
透過Cloudera Manager頁面，依序關閉Hue，Oozie，Impala，Hive  
點擊服務旁邊的下拉式選單，選擇Stop  
![Hive_HA](https://i.imgur.com/YXJtqxK.png)
透過Cloudera Manager頁面，點擊Hive，點擊Actions下拉式選單，選取Update Hive Matastore NameNodes  
![Hive_HA_1](https://i.imgur.com/dq08pfD.png)
選取Update Hive Matastore NameNodes
![Hive_HA_2](https://i.imgur.com/N0I5ehG.png)
完成後點擊Close
![Hive_HA_3](https://i.imgur.com/gMqAu7S.png)
回到Cloudera Manager頁面，依序啟動Hive，Impala，Ooziw，Hue

設定Hive Metastore HA，透過Cloudera Manager頁面，點擊Hive，點擊Configuration
![Hive_HA_4](https://i.imgur.com/D2rrVj9.png)
於左邊Scope選取Hive Metastore Server，找到Hive Metastore Delegation Token Store
![Hive_HA_5](https://i.imgur.com/oX6e7Uu.png)
選擇org.apache.hadoop.hive.thrift.DBTokenStore，點擊Save Changes
![Hive_HA_6](https://i.imgur.com/PCQtuCW.png)
透過Cloudera Manager頁面，點選設定變更圖示，點擊Restart Stale Services，Restart Now
![Hive_HA_7](https://i.imgur.com/hXQdpuj.png)
![Hive_HA_8](https://i.imgur.com/aux6bIx.png)
![Hive_HA_9](https://i.imgur.com/ZEdlLWx.png)
完成後，點擊Finish
![Hive_HA_10](https://i.imgur.com/CU6E3cO.png)

* * *

# 9. 啟用YARN HA
透過Cloudera Manager頁面，點擊YARN，點擊Actions下拉式選單，選取Enable High Availability
![YARN_HA](https://i.imgur.com/dp0dkhz.png)
選擇想安裝的主機，點擊Continue
![YARN_HA_1](https://i.imgur.com/EQHTLnG.png)
完成後，點擊Finish
![YARN_HA_2](https://i.imgur.com/sHM7elm.png)

* * *

# 10. HAProxy + Keepalived 達到 Load Balance + HA
安裝HAProxy，指令如下(選兩台主機安裝)：
>sudo apt-get install haproxy  
備份HAProxy設定檔，指令如下：  
sudo cp -p /etc/haproxy/haproxy.cfg
/etc/haproxy/haproxy.cfg.backup  
編輯設定檔，指令如下  
sudo vim /etc/haproxy/haproxy.cfg  
貼入以下設定： 
global  
    \# To have these messages end up in    /var/log/haproxy.log you will  
    \# need to:  
    \#  
    \# 1) configure syslog to accept network log events.    This is done 
    \#    by adding the '-r' option to the SYSLOGD_OPTIONS in  
    \#    /etc/sysconfig/syslog  
    \# 
    \# 2) configure local2 events to go to the  /var/log/haproxy.log 
    \#   file. A line like the following can be added to 
    \#   /etc/sysconfig/syslog 
    \# 
    \#    local2.*                        /var/log/haproxy.log 
    \# 
    log         127.0.0.1 local0  
    log         127.0.0.1 local1 notice  
    log         127.0.0.1 local2 warning  
    chroot      /var/lib/haproxy   
    pidfile     /var/run/haproxy.pid   
    maxconn     4000  
    user        haproxy  
    group       haproxy
    daemon

>   \# turn on stats unix socket  
\#stats socket /var/lib/haproxy/stats  
  
>\#---------------------------------------------------------------------  
>\# common defaults that all the 'listen' and 'backend' sections will  
\# use if not designated in their block  
\#  
\# You might need to adjust timing values to prevent  timeouts.  
\#  
\# The timeout values should be dependant on how you use the cluster  
\# and how long your queries run.  
\#---------------------------------------------------------------------  
defaults  
    mode                    http  
    log                     global  
    option                  httplog  
    option                  dontlognull  
    option http-server-close  
    option forwardfor       except 127.0.0.0/8  
    option                  redispatch  
    retries                 3  
    maxconn                 3000  
    timeout connect 5000  
    timeout client 3600s  
    timeout server 3600s  

>\#  
\# This sets up the admin page for HA Proxy at port 25002. 
\#  
listen stats   
bind 安裝主機IP:25002  
\#http://安裝主機IP:25002/haproxy?stats  
    balance  
    mode http  
    stats enable  
    stats auth admin:V5vz6am3&E  

>\# This is the setup for Impala. Impala client connect to load_balancer_host:25003.  
\# HAProxy will balance connections among the list of servers listed below.  
\# The list of Impalad is listening at port 21000 for beeswax (impala-shell) or original ODBC driver.  
\# For JDBC or ODBC version 2.x driver, use port 21050 instead of 21000.  
listen impala   
bind 10.0.40.120:25003  
\#10.0.40.120 此為VIP  
    mode tcp  
    option tcplog   
    balance leastconn  

 >  server cloudera03 10.0.40.103:21000 check  
    server cloudera04 10.0.40.104:21000 check   
    

>\# Setup for Hue or other JDBC-enabled applications.  
\# In particular, Hue requires sticky sessions.  
\# The application connects to load_balancer_host:21051,  and HAProxy balances  
\# connections to the associated hosts, where Impala   listens for JDBC  
\# requests on port 21050.  
listen impalajdbc   
bind 10.0.40.120:21051  
\#10.0.40.120 此為VIP  
    mode tcp  
    option tcplog  
    balance source  
    server cloudera03 10.0.40.103:21050 check  
    server cloudera04 10.0.40.104:21050 check  

安裝Keepalived，指令如下(與HAProxy兩台同主機)：  
>sudo apt-get install keepalived  
編輯Keepalived設定檔，指令如下：  
sudo vim /etc/keepalived/keepalived.conf  
global_defs {  
router_id LVS_DEVEL  
}  

>/#HAProxy Health Check  
vrrp_script chk_haproxy {  
script "/etc/keepalived/check_haproxy.sh"  
interval 2  
}  


>vrrp_instance VI_1 {  
state MASTER [此參數兩台主機設置不同]  
interface enp6s0f0  
virtual_router_id 51  
priority 101 [此參數兩台主機設置不同]  
advert_int 1  
authentication {  
auth_type PASS  
auth_pass 1111  
}  

>virtual_ipaddress {  
10.0.40.120  
}  

>track_script {  
chk_haproxy  
}  


>}  
兩台主機配置如下：  

![HAProxy_Check_1](https://i.imgur.com/PppYcVm.png)
![HAProxy_Check_2](https://i.imgur.com/qrsrnus.png)

編輯檢查HAProxy狀態的程式，指令如下：  
>sudo vim /etc/keepalived/check_haproxy.sh  
\#!/bin/bash   

>num=\`ps -C haproxy --no-header |wc -l`  

>if [ $num -eq 0 ]  
then  
/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg  
sleep 3   
if [ \`ps -C haproxy --no-header |wc -l` -eq 0 ]  
then  
sudo /etc/init.d/keepalived stop  
fi   
fi  

![HAProxy_Check](https://i.imgur.com/NbG3Fev.png)

給檢查的程式，執行的權限，指令如下： 
>sudo chmod +x /etc/keepalived/check_haproxy.sh

設定HAProxy Log，指令如下：
>sudo vim /etc/rsyslog.conf  
拿掉兩個註解並且新增一行  
\# provides UDP syslog reception  
\#module(load="imudp")  
\#input(type="imudp" port="514")  
$AllowedSender UDP, 127.0.0.1  

![HAProxy_Log](https://i.imgur.com/CbkjxtL.png)

>sudo vim /etc/rsyslog.d/50-default.conf  
新增設定  
*.*;auth,authpriv.none;local2.none		-/var/log/syslog  
local2.*                        /var/log/haproxy.log  

![HAProxy_Log_1](https://i.imgur.com/3WJo2KG.png)

重啟HAproxy，Keepalived以及rsyslog，指令如下：  
>sudo systemctl restart haproxy.service   
sudo systemctl restart keepalived.service  
sudo systemctl restart rsyslog.service 

檢查是否有成功抓到虛擬IP(安裝的主機個別檢查)，指令如下：
>ip a s

![HAPROXY](https://i.imgur.com/59DyosH.png)
