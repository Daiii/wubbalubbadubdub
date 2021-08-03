##### kafka 


----------


### 1 . ip&port list
> ### producer.sh
> | project |       ip        | port |
> | ------- | :-------------: | ---: |
> | dev     |   10.211.95.9   | 9092 |
> | st      | 192.168.177.192 | 5811 |
> | sit     | 192.168.12.211  | 9092 |
> | xncs    | 192.168.177.35  | 9092 |

---

> ### consumer.sh
> | porject |       ip        | port |
> | ------- | :-------------: | ---: |
> | dev     |   10.211.95.9   | 6830 |
> | st      | 192.168.177.192 | 6830 |
> | sit     | 192.168.12.211  | 6830 |
> | xncs    | 192.168.177.35  | 6830 |

---

### 2 . producer.sh
> ##### 1 . 发送MQ消息
> ##### 2 . ```shell bin/kafka-console-producer.sh --broker-list ip:port --topic Topic ```

---

### 3 . consumer.sh
> ##### 1 . 接收MQ消息
> ##### 2 . ```shell bin/kafka-console-consumer.sh --zookeeper ip:port --topic Topic ```

---

### 4 . topic list
> ##### 1 . Topic列表
> ##### 2 . ``` ./kafka-topics.sh --list --zookeeper 192.168.12.211:6830 ```

---

### 5 . Merge Into SQL
```SQL
MERGE INTO ${tablename} T1
		USING (SELECT #{msisdn} AS MSISDN FROM dual) T2
		ON (T1.MSISDN = T2.MSISDN)
		WHEN MATCHED THEN
		  UPDATE
		     SET T1.EXPS    = T1.EXPS + #{exps,
		         jdbcType   = NUMERIC},
		         UPDATETIME = SYSDATE
		   WHERE T1.MSISDN = #{msisdn}
		WHEN NOT MATCHED THEN
		  INSERT
		    (ID, MSISDN, EXPS, LEVELID, CREATETIME, UPDATETIME)
		  VALUES
		    (#{uid}, #{msisdn}, #{exps, jdbcType = NUMERIC}, 1, SYSDATE, SYSDATE)
```

---

```java
public static void main(String[] args){
    System.out.println("Hello World");
}
```

---

```shell
kafka-console-consumer.sh --zookeeper 192.168.17.36:6830,192.168.17.44:6830,192.168.17.89:6830 --topic ReadDuration --from-beginning | grep -a 42034068626
```

---
```shell
cat *6720170511* | awk -F'\\|'0 '{count[$3]++;costtime[$3]+=$6;if($7!=200){error[$3]++;};$3}END{for(action in count){printf("%s,%d,%d,%d,%.2f\n",action,costtime[action]/count[action],count[action],error[action],(count[action]-error[action])/count[action]*100)}}' | sort -t ',' -k 2nr
```

---
```shell
cat *4C20180416*.txt | awk -F'|' '{a_array[$3]+=$7;b_array[$3]++} END{for (i in a_array) print i":"a_array[i]":"b_array[i]":"a_array[i]/b_array[i]}' 
```
```shell
cat *4C20180824* | awk -F'\\|' '{count[$3]++;costtime[$3]+=$7;if($8!=200){error[$3]++;};$3}END{for(action in count){printf("%s,%d,%d,%d,%.2f\n",action,costtime[action]/count[action],count[action],error[action],(count[action]-error[action])/count[action]*100)}}' | sort -t ',' -k 2nr
```

---

```shell
zcat 20170515.tar.gz | awk -F'\\|' '{count[$3]++;costtime[$3]+=$6;if($7!=200){error[$3]++;};$3}END{for(action in count){printf("%s,%d,%d,%d,%.2f\n",action,costtime[action]/count[action],count[action],error[action],(count[action]-error[action])/count[action]*100)}}' | sort -t ',' -k 2nr 
```

---

```shell
./kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zkconnect 192.168.17.36:6830 --group sns.read.duration
```

---

```shell
top -p 17315 -H
```


---


```shell
tcpdump -s 0 -i any port 7070  -w 013220.cap
```

---

```
eclipse mat
```

---

```shell
for i in {0..9}
do
        #echo $i
        redis-cli -n 1 keys sns.execute.campaign_$i* |  xargs redis-cli -n 1 del
        sleep 0.5
done
```


---


```shell
netstat -na | grep ESTAB | grep 7280 | wc -l
```


---


```java
private ExecutorService poolExecutor = new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors(),
            Runtime.getRuntime().availableProcessors(), 1, TimeUnit.SECONDS, new LinkedBlockingDeque<Runnable>(), new ThreadFactory() {
        private final AtomicInteger atomicInteger = new AtomicInteger();

        @Override
        public Thread newThread(Runnable r) {
            return new Thread(r, "ThreadPool name :" + atomicInteger.getAndIncrement());
        }
    });
```

---


```java
/**
 * 单列默认
 
 */
public class Singleton {
    private Singleton(){}
    
    private static class SingletonHolder {
        static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```
