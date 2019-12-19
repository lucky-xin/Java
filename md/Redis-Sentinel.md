# bash redis-start.sh启动shell脚本待服务都启动

# 执行
```shell script
docker exec -it redis_master bash -c "redis-cli -p 6379 -a Data*2019* info replication"
```
查看redis主节点信息，输出如下：
```text
# Replication
role:master
connected_slaves:0
master_replid:656f6a293635adb4f97f6da3883869f3b5eb4abf
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```

# 执行
```shell script
docker exec -it redis_slave1 bash -c "redis-cli -p 6380 -a Data*2019* info replication"
```
查看redis从节点redis_slave1信息，输出如下：
```text
# Replication
role:slave
master_host:192.168.1.7 # 主节点ip
master_port:6379        # 主节点端口
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:2977
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:dbb42c525c5c7ef1991dcfa01ee37bdea7feb4a8
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:2977
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:2977
```
# 执行
```shell script
docker exec -it redis_slave2 bash -c "redis-cli -p 6381 -a Data*2019* info replication"
```
查看redis从节点redis_slave2信息，输出如下：
```text
# Replication
role:slave
master_host:192.168.1.7 # 主节点ip
master_port:6379        # 主节点端口
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_repl_offset:26091
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:dbb42c525c5c7ef1991dcfa01ee37bdea7feb4a8
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:26091
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:26091
```
# 此时主节点为redis_master ip为192.168.1.7 端口为6379

执行
```text
docker exec -it redis_sentinel1 redis-cli -p 26379 info
```
输出如下：
```text
# Server
redis_version:5.0.7
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:7359662505fc6f11
redis_mode:sentinel
os:Linux 4.19.76-linuxkit x86_64
arch_bits:64
multiplexing_api:epoll
atomicvar_api:atomic-builtin
gcc_version:8.3.0
process_id:1
run_id:293964ba2d67ca583863a38a1d8082bf1a8509c8
tcp_port:26379
uptime_in_seconds:159
uptime_in_days:0
hz:15
configured_hz:10
lru_clock:16437751
executable:/data/redis-server
config_file:/etc/redis/sentinel.conf

# Clients
connected_clients:3
client_recent_max_input_buffer:2
client_recent_max_output_buffer:0
blocked_clients:0

# CPU
used_cpu_sys:0.426013
used_cpu_user:0.222006
used_cpu_sys_children:0.000000
used_cpu_user_children:0.000000

# Stats
total_connections_received:3
total_commands_processed:458
instantaneous_ops_per_sec:2
total_net_input_bytes:25790
total_net_output_bytes:2743
instantaneous_input_kbps:0.11
instantaneous_output_kbps:0.02
rejected_connections:0
sync_full:0
sync_partial_ok:0
sync_partial_err:0
expired_keys:0
expired_stale_perc:0.00
expired_time_cap_reached_count:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:0
migrate_cached_sockets:0
slave_expires_tracked_keys:0
active_defrag_hits:0
active_defrag_misses:0
active_defrag_key_hits:0
active_defrag_key_misses:0

# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
# redis 主节点信息
master0:name=redis_master,status=ok,address=192.168.1.7:6379,slaves=2,sentinels=3
```
redis_sentinel2，redis_sentinel3信息类似
# SpringBoot Sentinel 模式连接
## bootstrap.yml配置
```yaml
server:
  port: ${SERVER_PORT:19012}
  max-http-header-size: 30000

spring:
  main:
    allow-bean-definition-overriding: true
  application:
    name: @project.artifactId@
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_HOST:datainsights-register}:${NACOS_PORT:8848}
      config:
        server-addr: ${spring.cloud.nacos.discovery.server-addr}
        file-extension: yml
        shared-dataids: application-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}
  profiles:
    active: dev
  # Redis Sentinel模式连接
  redis:
    sentinel:
      master: redis_master
      nodes: 192.168.1.7:26379,192.168.1.7:26380,192.168.1.7:26381
    password: Data*2019*
```
## pom.xml中添加如下配置
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
## 代码配置
```java
/***
 * @Description: Redis Sentinel 配置
 * @Auther: Luchaoxin
 * @Date: 2019/12/19
 * @version : V1.0
 */
@Slf4j
@Configuration
@EnableAutoConfiguration
public class RedisConfig {
    
    @Bean
    @ConfigurationProperties(prefix = "spring.redis")
    public JedisPoolConfig getRedisConfig() {
        JedisPoolConfig config = new JedisPoolConfig();
        return config;
    }

    @Bean
    public RedisSentinelConfiguration sentinelConfiguration(RedisProperties redisProperties) {
        RedisSentinelConfiguration redisSentinelConfiguration = new RedisSentinelConfiguration();
        //配置matser的名称
        redisSentinelConfiguration.master(redisProperties.getSentinel().getMaster());
        //配置redis的哨兵sentinel
        Set<RedisNode> redisNodeSet = new HashSet<>();
        redisProperties.getSentinel().getNodes().forEach(x -> {
            String[] array = x.split(":");
            redisNodeSet.add(new RedisNode(array[0], Integer.parseInt(array[1])));
        });
        log.info("redisNodeSet -->" + redisNodeSet);
        redisSentinelConfiguration.setSentinels(redisNodeSet);
        redisSentinelConfiguration.setPassword(redisProperties.getPassword());
        return redisSentinelConfiguration;
    }

    @Bean
    public JedisConnectionFactory jedisConnectionFactory(JedisPoolConfig jedisPoolConfig, RedisSentinelConfiguration sentinelConfig) {
        JedisConnectionFactory jedisConnectionFactory = new JedisConnectionFactory(sentinelConfig, jedisPoolConfig);
        return jedisConnectionFactory;
    }

}
```
## SpringBoot启动日志如下：
```text
2019-12-19 09:58:48.110 INFO  192.168.1.7 4714 c.a.c.s.SentinelWebAutoConfiguration#sentinelFilter   [Sentinel Starter] register Sentinel CommonFilter with urlPatterns: [/*]. 
2019-12-19 09:58:49.185 INFO  192.168.1.7 4714 b.d.s.q.c.m.c.RedisConfig#sentinelConfiguration   redisNodeSet -->[192.168.1.7:26379, 192.168.1.7:26381, 192.168.1.7:26380] 
2019-12-19 09:59:33.332 INFO  192.168.1.7 4714 r.c.j.JedisSentinelPool#initSentinels   Trying to find master from available Sentinels... 
2019-12-19 09:59:43.143 INFO  192.168.1.7 4714 r.c.j.JedisSentinelPool#initSentinels   Redis master running at 192.168.1.7:6379, starting Sentinel listeners... 
2019-12-19 09:59:56.326 INFO  192.168.1.7 4714 r.c.j.JedisSentinelPool#initPool   Created JedisPool to master at 192.168.1.7:6379 
```

查看任意Sentinel 日志
```shell script
docker logs -f --tail 200 redis_sentinel2
```
日志输出如下，可知此时Redis Master ip为 192.168.1.7 port 为6379
```text
1:X 19 Dec 2019 01:24:41.477 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:X 19 Dec 2019 01:24:41.477 # Redis version=5.0.7, bits=64, commit=00000000, modified=0, pid=1, just started
1:X 19 Dec 2019 01:24:41.477 # Configuration loaded
1:X 19 Dec 2019 01:24:41.478 * Running mode=sentinel, port=26380.
1:X 19 Dec 2019 01:24:41.478 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:X 19 Dec 2019 01:24:41.489 # Sentinel ID is d73806d2351cbcfc0235d49c8d9f2b373cf0e0f7
1:X 19 Dec 2019 01:24:41.489 # +monitor master redis_master 192.168.1.7 6379 quorum 2
1:X 19 Dec 2019 01:24:41.495 * +slave slave 172.20.0.1:6380 172.20.0.1 6380 @ redis_master 192.168.1.7 6379
1:X 19 Dec 2019 01:24:42.987 * +sentinel sentinel 3d9ef79aeca294ee19121fb5ce301b24b96cac53 172.20.0.2 26379 @ redis_master 192.168.1.7 6379
1:X 19 Dec 2019 01:24:43.461 * +sentinel sentinel 199cf7a87ce712959a7a837a4de67f2c72112d85 172.20.0.5 26381 @ redis_master 192.168.1.7 6379
1:X 19 Dec 2019 01:24:51.547 * +slave slave 172.20.0.1:6381 172.20.0.1 6381 @ redis_master 192.168.1.7 6379
```

## SpringBoot 设置值
```java
/**
 * @author Luchaoxin
 * @version V 1.0
 * @Description: TODO
 * @date 2019-02-14 10:13
 */
@RunWith(SpringRunner.class)
@SpringBootTest(classes = SmartQrCodeManagerApplication.class)
public class ApplicationTest {

    @Autowired
    private RedisTemplate redisTemplate;


    @Test
    public void test() throws IOException {
        // 设置序列化为字符串
        redisTemplate.setValueSerializer(new StringRedisSerializer());
        redisTemplate.opsForValue().set("xin", "2020");
        Object val = redisTemplate.opsForValue().get("xin");
        System.out.println(val);
    }

}
```
# 去redis g各个节点查询 key为xin的值

### 主节点执行如下命令
```shell script
docker exec -it redis_master bash -c "redis-cli -p 6379 -a Data*2019* GET xin"
```
输出如下
```text
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
"2020"
```

### 从节点1执行如下命令
```shell script
docker exec -it redis_slave1 bash -c "redis-cli -p 6380 -a Data*2019* GET xin"
```
输出如下
```text
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
"2020"

### 从节点2执行如下命令
```shell script
docker exec -it redis_slave2 bash -c "redis-cli -p 6381 -a Data*2019* GET xin"
```
输出如下
```text
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
"2020"
```

# Redis 所有写操作数据同步正常，节点工作正常

# 手动停止Redis主节点
```shell script
docker stop redis_master
```
此时
# 执行
```shell script
docker exec -it redis_slave2 bash -c "redis-cli -p 6381 -a Data*2019* info replication"
```
输出如下，可知此时主节点已经切换到端口为6381的节点
```text
# Replication
role:master
connected_slaves:1
slave0:ip=172.20.0.1,port=6380,state=online,offset=934569,lag=1
master_replid:fa806f030f950b19a823c2064c6cb2a3d3d840e1
master_replid2:dbb42c525c5c7ef1991dcfa01ee37bdea7feb4a8
master_repl_offset:934708
second_repl_offset:863041
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:934708
```

# 执行
```shell script
docker exec -it redis_slave1 bash -c "redis-cli -p 6380 -a Data*2019* info replication"
```
输出如下，可知此时主节点已经切换到端口为6381的节点
```text
# Replication
role:slave
master_host:172.20.0.1
master_port:6381
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:946051
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:fa806f030f950b19a823c2064c6cb2a3d3d840e1
master_replid2:dbb42c525c5c7ef1991dcfa01ee37bdea7feb4a8
master_repl_offset:946051
second_repl_offset:863041
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:946051
```
# SpringBoot 日志输出如下
![](https://github.com/lucky-xin/Learning/blob/gh-pages/image/SpringBoot-Sentinel.png)
# Sentinel log
![](https://github.com/lucky-xin/Learning/blob/gh-pages/image/Sentinel-Log.png)

