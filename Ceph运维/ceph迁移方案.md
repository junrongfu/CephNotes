今年2021-06-15日将ceph的服务器进行了迁移，将ceph的数据从老的机器迁移到了新的机器上，不同于无状态的迁移，对数据的迁移时间周期是很长的在这个过程中很可能会出现异常，为了保证线上程序的稳定运行，必须让业务无感并且对采用整个系统负载最小的方案来迁移，所以制定了如下方案

## 迁移方案

------

1. 首先将新的节点的所有OSD初始化完毕，然后将这些OSD **加入到另一棵osd tree下** (这里简称原先的集群的osd tree叫做old_tree，新建的包含新OSD的 osd tree 叫做 new_tree)，这样部署完毕后，不会对原有集群有任何影响，也不会涉及到数据迁移的问题，此时新的OSD下还没有保存数据。
2. 导出 CRUSHMAP，编辑，添加三条 CRUSH rule：
   - crush rule 0 (原先默认生成的): 从 old_tree 下选出size副本（这里size=2)。
   - crush rule 1 (新生成的第一条): 从 old_tree 下选出两副本。对于副本数为2的集群来说，crush_rule_0 和 crush_rule_1 选出的两副本是一样的。
   - crush rule 2 (新生成的第二条): 从 old_tree 下选出两副本，再从 new_tree 下选出两副本。由**原理简介第二段**可知， 由于 old_tree 下面的 OSD结构不变也没有增加，所以 crush_rule_0 和 crush_rule_1 选出的前两副本是**一样的**。
   - crush rule 3 (新生成的第三条）: 从 new_tree 下选出两副本。 由于crush_rule_1 的第二次选择为选出 new_tree下的前两副本，这和 crush_rule_2 选出两副本（也是前两副本）其实是**一样的**。
3. 注入新的CRUSHMAP后，我们做如下操作：
   - 将所有pool(当然建议一个pool一个pool来，后面类似) 的 CRUSH RULE 从 crush_rule_0 设置为 crush_rule_1 ，此时所有PG均保持active+clean，状态没有任何变化。
   - 将所有pool的副本数设置为4， 由于此时各个 pool 的CRUSH RULE 均为 crush_rule_1 ，而这个 RULE 只能选出两副本，剩下两副本不会被选出，所以此时所有PG状态在重新 peer 之后，变为 active + undersized + degraded，由于不会生成新的三四副本，所以集群没有任何数据迁移(backfill) 动作，此步骤耗时短暂。
   - 将所有pool的 CRUSH RULE 从 crush_rule_1 设置为 crush_rule_2，此时上一条动作中没有选出的三四副本会从 new_tree 下选出，并且原先的两副本不会发生任何迁移，整个过程宏观来看就是在 old_tree -> new_tree 单向数据复制克隆生成了新的两副本，而旧的两副本没有移动。此时生成新的两副本耗时较长，取决于磁盘性能带宽数据量等，可能需要几天到一周的时间，所有数据恢复完毕后，集群所有PG变为 active+clean 状态。
   - 将所有pool的 CRUSH RULE 从 crush_rule_2 设置为 crush_rule_3，此时所有PG状态会变为 active+remapped，发生的另一个动作是，原先四副本的PG的主副本是在 old_tree 上的某一个OSD上的，现在这个PG的主副本变为 new_tree下的选出的第一个副本，也就是发生了主副本的切换。比如原先 PG 1.0 => [osd.a, osd.b, osd.c, osd.d] 在这步骤之后会变成 PG 1.0 => [osd.c, osd.d]。原先的第三副本也就是new_tree下的第一副本升级为主副本。
   - 将所有pool的副本数设置为2，此时PG状态会很快变为 active+clean，然后在OSD层开始删除 old_tree 下的所有数据。此时数据已经全部迁移到新的OSD上。

## 注意事项

------



由于本次数据迁移是在生产环境上执行的，所以没有直接执行将数据从旧节点直接 `mv` 到新节点，而是选择了执行步骤较为复杂的上面的方案，先 `scp` 到新节点，再 `rm` 掉旧节点的数据。并且每一个步骤都是可以快速回退到上一步的状态的，对于生产环境的操作，是比较友好的。在本次方案测试过程中，遇到了如下的一些问题，需要引起充分的注意：

- Ceph 版本不一致： 由于旧的节点的 Ceph 版本为 0.94.5 ，而新节点安装了较新版本的 10.2.7， 在副本 2=>4 的过程中Peer是正常的，而将池的crush_ruleset 设置为3 ，也就是将新节点的 PG 升级为主副本后，PG由新节点向旧节点发生Peer，此时会一直卡住，PG始终卡在了 `remapped + peering`，导致该 pool 无法IO。在将**新旧节点 Ceph 版本一致**后(旧节点升级，新节点降级)，此现象得以消除。 为了确保此现象不会在实际操作中发生，应该在变更之前，新建一个测试pool，对其写入部分数据，再执行所有数据迁移指令，查看此过程是否顺畅，确认无误后，再对生产pool进行操作！
- 新节点 OSD 的重启问题： 如果没有添加了 `osd_crush_update_on_start` 这个配置参数，那么当新节点的OSD重启后，会自动添加到默认的 `root=default` 下，然后立刻产生数据迁移，因此需要添加这个参数，保证OSD始终位于我们人为指定的节点下，并不受重启影响。这是个基本的Ceph运维常识，但是一旦遗忘了，可能造成较为严重的影响。更不用说防火墙时钟这些配置了。
- 集群性能降低：总体来说，在副本从2克隆为4这段时间(约2-3天，取决于集群数据量）内，集群的实际IO表现降低到变更前的 25%->80% 左右，时间越往后表现越接近变更前，这虽然不会导致客户端的IO阻塞，但从客户反馈来看，可以感知到较为明显的卡顿。因此克隆时间应该选择业务量较低的节假日等。
- 新节点IP不`cluster_network`范围内：这个比较好解决，只需要增大部署目录`ceph.conf`内的`cluster_network` 或者 `public_network` 的掩码范围即可，不需要修改旧节点的，当然网络还是要通的。
- 变更的回退：对于生产环境来说，尽管执行步骤几乎是严谨不会出错的，但是难免会遇到意外情况，就比如上面的版本不一致导致的 peer 卡住现象。因此我们需要制定完善的回退步骤，在意外发生的时候，能够快速将环境回退到上一步集群正常的状况，而不是在意外发生时惊出一身冷汗，双手颤抖得敲指令。。。所以，下面的表格给出了这个变更操作的每一步的回退步骤：

## 迁移指令

------

| 变更步骤                                           | 实际影响                                           | 回退指令                                              | 回退影响                                                     |
| -------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------ |
| ceph osd pool set $pool_name crush_rule $tmp_rule1 | 无                                                 | ceph osd pool set $pool_name crush_rule $default_rule | 无                                                           |
| ceph osd pool set $pool_name size 4                | PG由active+clean，变为 active+undersized+degraded  | ceph osd pool set $pool_name size 2                   | PG很快恢复active+clean                                       |
| ceph osd pool set $pool_name crush_rule $tmp_rule2 | PG 开始 backfill，需要较长时间                     | ceph osd pool set $pool_name crush_rule $tmp_rule1    | PG很快恢复到active+undersized+degraded，并删除在新节点生成的数据。 |
| ceph osd pool set $pool_name crush_rule $tmp_rule3 | PG 很快变为 active+remapped。                      | ceph osd pool set $pool_name crush_rule $tmp_rule2    | PG很快恢复到 active+clean                                    |
| ceph osd pool set $pool_name size 2                | PG 很快变为 active+clean，并且后台删除旧节点数据。 | ceph osd pool set $pool_name size 4                   | PG 变为active+remapped。                                     |