bucket 分片问题：

对象存储中如果当前bucket中的对象数超过了分片的最大对象数，会触发自动分片，在自动分片过程中会导致无法读写。如果卡住时间比较久请求队列过大导致rgw堵住影响全局性问题
处理目标：出现卡住问题时在不关闭自动分片的情况下，只影响单一分片不影响全局，控制分片时间能够在夜间低峰期运行
关键结果1：bucket分片期间，丢掉到此bucket的所有请求，保证全局的可用性
关键结果2：bucket分片的时间控制在低峰时间段，减小因bucket分片丢请求产生对业务的影响

处理方案：
\1. 对bucket进行分片检查，如果bucket需要分片则检查当前时间，把分片放到夜间或周末
分片检查使用"radosgw-admin bucket limit check"先获取所有的bucket然后检查每个分片的最大对象数，如果超过设置的值则以当前分片数据的2倍进行重新分片

\2. 开始分片前对bucket设置PUT deny策略，然后执行"radosgw-admin bucket reshard --bucket= --num-shards="触发分片"

\3. 分片完成后继续检查下一个bucket

\4. bucket分片检查每天凌晨检查一次

bucket分片流程说明：

1. 定时任务每天23点开始执行radosgw-admin bucket limit check获取所有bucket的信息
2. 逐个检查bucket中"objects_per_shard"的值，如果超过10万，则需要分片，否则遍历下一个bucket
3. 检查当前bucket配置的策略，如果已配置策略则先把当前策略保存到s3上；如果没有策略则生成空策略上传到s3中为后面分片完成后的恢复做准备
4. 在现有的bucket策略上添加put deny规则，并set到bucket上
5. 执行radosgw-admin bucket reshard --bucket= --num-shards="当前分片数*2"，开始分片
6. 上一步分片完成之后会从s3上获取老的策略配置set到bucket上。继续检查下一个bucket是否需要分片，如果所有的bucket都遍历完成则本次分片结束
7. 由于每天23点开始遍历所有bucket，每次只对一个bucket进行分片，直到遍历完成



深度扫描主要配置说明

| 配置项                     | 含义                                                         | 影响程度 |
| -------------------------- | ------------------------------------------------------------ | -------- |
| osd_max_scrubs             | 单个osd上同时扫描pg的数量                                    | 大       |
| osd_scrub_chunk_min        | 每个chunky包含的最小对象数，ceph以chunky为单位对一组对象进行扫描 | 小       |
| osd_scrub_chunk_max        | 每个chunky包含的最大对象数，ceph以chunky为单位对一组对象进行扫描 | 小       |
| osd_scrub_sleep            | 每个chunky之前sleep的时间，此配置sleep期间会释放占用的线程，降低深度扫描的速率 | 大       |
| osd_scrub_priority         | 执行深度扫描调度的优先权，如果调整的过高会导致客户端请求一直得不到响应 | 大       |
| osd_deep_scrub_keys        | 深度扫描期间一次对每个对象key的读取个数，存放索引的对象包含的key比较大，调整后影响比较大 | 小       |
| osd_debug_deep_scrub_sleep | 在deep scrub IO期间注入sleep以使其更容易被抢占（此项会诱导抢占io资源，不会释放工作线程，会同时影响深度扫描的速率和客户端请求） | 大       |

深度扫描调整案例：

问题：正式集群索引盘深度扫描导致客户端慢请求[ARBASE-880](http://172.20.200.191:8080/browse/ARBASE-880)

原因：深度扫描期间导致客户端请求无法及时处理产生慢请求

处理方式：

1. 调整osd_scrub_sleep 配置，在chunky之间sleep，降低深度扫描的速率
2. 如果需要降低单个chunky的扫描时间，调整osd_deep_scrub_keys,osd_scrub_chunk_min,osd_scrub_chunk_max

动态重平衡配置调整参数主要涉及以下参数：

osd_recovery_max_active

osd_recovery_max_single_start

osd_recovery_sleep

详细处理流程：

1. 通过ceph pg ls | grep -E "backfill|recovery"检查当前pg是否正常重平衡
2. 如果没有重平衡的pg则尝试把配置调整最小，如果已经最小则不再调整
3. 如果当前有重平衡的pg，则开始重平衡操作
4. 通过ceph osd perf获取所有osd的延迟数据，根据延迟数据确定调整方案
5. 重平衡配置的调整有三种情况：一、如果延迟数据大于最大阈值500，则立即把配置调整到最小，如果在(240，500)超过三次则往下调整，如果在区间（160~240）则不需要调整，如果小于160则按照指定步长上调，阈值和步长可以通过配置动态调整并实时生效；当前步骤是在区间中不需要调整配置
6. 根据延迟数据和配置需要调大或调小重平衡配置
7. 配置调整完成后需要更新调整的日志缓存，此日志缓存记录上一次调整方案，如超过最大阈值的次数。配置调整之后需要更新此缓存

配置说明如下：

```
`#动态调整间隔（重平衡配置调整完成sleep 3秒后继续下次调整） sleepTime=3 #休盘期间最小调整阈值（osd perf的数据和此阈值对比，如果在休盘期间小于此阈值则继续按照步长上调，大于此值小于maxTradeOutAdjustThreshold则无需调整） minTradeOutAdjustThreshold=160 #休盘期间最大调整阈值（osd perf的数据和此阈值对比，如果在休盘期间大于此阈值且小于maxOsdPerfThreshold超过三次则开始下调，如果大于minTradeOutAdjustThreshold小于此值则无需调整） maxTradeOutAdjustThreshold=240 #盘中最小调整阈值(9~15)（同minTradeOutAdjustThreshold，此值在9~15点生效） minTradeInAdjustThreshold=120 #盘中最大调整阈值(9~15)（同maxTradeOutAdjustThreshold，此值在9~15点生效） maxTradeInAdjustThreshold=200 #最大重平衡配置,即Osd_recovery_max_active和Osd_recovery_max_single_start的最大上限，Osd_recovery_sleep最大为1 maxRebalanceCfg=100 #Osd_recovery_sleep向上调整的步长 sleepAdjustStep =0.05 #Osd_recovery_max_active和Osd_recovery_max_single_start向上调整的步长 recoverAdjustStep=10 #日志开关 logLevel=Info #osd perf最高阈值，如果超过此阈值立即调到最低（最低配置：osd_recovery_max_active=1，osd_recovery_max_single_start=1，osd_recovery_sleep=1） maxOsdPerfThreshold=500`
```



| 配置                          | 说明                                                         | 影响 |
| ----------------------------- | ------------------------------------------------------------ | ---- |
| osd_max_backfills             | 单个osd的并发backfill的pg数                                  | 大   |
| osd_recovery_max_active       | 每个OSD上同时进行的所有PG的恢复操作数，此值在与osd_recovery_max_single_start之间取最小值作为一次入队恢复的操作数 | 大   |
| osd_recovery_max_single_start | OSD为一个PG启动恢复操作数，可以结合sleep配置控制恢复的速率   | 大   |
| osd_recovery_sleep            | 恢复操作出队列后先Sleep一段时间，拉长两个Recovery的时间间隔  | 大   |
| osd_recovery_sleep_hdd        | hdd类型的osd在恢复节点的sleep时间，如果osd_recovery_sleep不为0此配置无效 | 大   |
| osd_recovery_sleep_hybrid     | 混合类型的osd在恢复节点的sleep时间，如果osd_recovery_sleep不为0此配置无效 | 大   |
| osd_recovery_sleep_ssd        | ssd类型的osd在恢复节点的sleep时间，如果osd_recovery_sleep不为0此配置无效 | 大   |

当前ceph在回填pg过程中对集群的可用性影响比较大，需要根据可用性的指标数据调整回填的进度。
方案：
\1. 从小到大按照步长快速调大，如果perf的数据在min~max区间则停止上调，如果连续3次大于max，进入步骤2
\2. 把配置参数调为当前的一半，继续监控perf数据
\3. perf数据连续3次大于max转2，否则4，
\4. 小幅度调大配置，如果perf数据小于min继续上调直到设定的上限，在此区间则停止调整，如果大于min转2

测试点：
\1. 验证步长配置更新(ok)
\2. 验证回填的环境判断,已经是默认的不用填，其他情况需要再回填(ok)
\3. 验证osd取配置的功能（ok）
\4. 验证修改调整数据(ok)
\5. 验证set数据(ok)
\6. 验证取perf数据(ok)
\7. 验证计算osd状态(ok)
\8. 验证上调(ok)
\9. 验证下调(ok)
\10. 验证恒定操作(ok)