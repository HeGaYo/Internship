问题相关issue参考：scheduler/cld-api-server#79
nfvi备盘池issue参考：scheduler/cld-api-server#134
## cube中备盘池方案设计
目前提供虚机的流程是：
  - 基于 cinder snapshot 创建一个系统盘
  - 创建 swap，data 盘
  - 将以上三块盘通过 device-mapping 挂进去

创建cinder volume的时候可能会因为cinder的配置`rbd_flatten_volume_from_snapshot=True`导致创盘很慢，或者进程挂掉，影响了整个虚机创建的过程。为了加速这个过程，可以先提前预备好一批available的sys盘，到时创建请求来了之后，直接在备盘池中选择一个备盘即可，然后拉起server即可。这样子整个过程会比较快速。

我们可以参考之前在nfvi实现备盘池的方案scheduler/cld-api-server#134，做一定的修改，移植到cube中。

1. 方案简述
  - 新增volume-worker，使用beat来执行celery的周期任务。
    - def backup_volume()
      - step1: 身份认证，获取token，tenant id
      - step2：从配置文件中获取备盘的配置信息.这里我们根据type(sas. ssd)以及os_name(windows，debians)来创建不同的盘。
      - step3: 检查数据库backup_volumes，看这次备盘操作,哪几种类型的盘需要备，需要备多少个。用两个参数来过滤计数(snapshot_type, status)
      - step4: 获取备盘的两个参数volume_type和snapshot_id。通过cinder list以及os表来查询。
      - step5：create_volume(cld_api, vol_data), 通过post请求Cinder API，创盘
      - step6: 将创盘的记录加入到数据库中
        - 根据返回值中的volume_id，查询数据库是否有条目
        - 如果有，就更新相关的信息。name, status, allocated
        - 如果没有，创建一条新的条目。id, name, type, snapshot, allocated

    - check_volume_state
      - 检查状态为creating的所有volume状态变化。获取所有状态为creating的volume的volume_id
      - 查询volume在cinder中的状态
        - 如果为available，更新本地的数据库表volume的状态
        - 如果为creating，说明还没创建完，下次还需查看这个volume的状态
        - 如果为其他状态，一律删除本地数据库中对应的volume条目。
  - 原来创建server的流程修改
    - 在`create_volume_if_absent`中，添加从备盘池选择的逻辑，根据volume_type和os_name从备盘池中选择有效的备盘，并且将备盘池中的数据删除。当备盘池空的时候，我们走原来的创建流程。

    ```python
    @classmethod
    def create_volume_if_absent(cls, cld_api, hostname, volume_category,
                                **kwargs):
        volume_name = "%s_%s" % (hostname, volume_category)
        volume_id = ServerUtil.get_volume_by_name(cld_api, volume_name)

        ## 增加从备盘池中查询volume
        ## if exist in backup_volumes,
        ##   check_state
             ## if ok,
                  ## rename the volume， 赋值volume_id
        ## else volume_id = None

        if volume_id is not None:
            return volume_id
        result = cls.create_volume(cld_api, hostname, volume_category,
                                   **kwargs)
        return result['volume']['id']
    ```
    - 删除server的流程无需改变

2. 数据库设计

|属性|含义|备注|
|---|---|---|
|id   |自增id   |   |
|volume_id   |volume的UUID，与cinder中的volume id相同  |   |
|type   |volume_type，与cinder中的volume type相同   | ssd/ sas  |
|snapshot_id   |对应的snapshot_id   |   |
| os  |os类型   |   |
|status   |volume状态   | creating/available  |
|created_at   |创建时间   |   |
|updated_at   |更新时间  |   |

3. 方案对比，如何获取snapshot_id
正常逻辑中获取snapshot_id

```
def get_os_by_name(cls, name, ebs_type):
    ebs_type = ebs_type.lower()
    _os = None
    if name:
        _os = current_session.query(Os).filter_by(name=name, type=ebs_type).first()
    return _os

if _os and _os.snapshot_id:
    server_args['snapshot_id'] = _os.snapshot_id
```

创盘需要的信息json格式，如何获取volume_type。首先是通过os表查询对应的volume-id，然后再通过这个volume_id去查询cinder，看这个volume对应的volume_type，再赋值。
```python
volume_data = {
            "volume": {
                "size": size,
                "multiattach ": False,
                "snapshot_id": snapshot_id,
                "name": "backup_volume",
                "volume_type": volume_type,
                "metadata": {},
            },
        }
```
![](assets/markdown-img-paste-2018122115264569.png)
cinder
 list中volume_type是由ceph池+volume_type组成。

备盘之前要获取snapshot_id, volume_type
### 方案1
从os表中读取os_name对应的
![](assets/markdown-img-paste-20181221152908415.png)
os表怎么来的呢
sync_os：(由管理员更新镜像后，手动触发)
1. 清空os表
2. 读取cinder的snapshot-list，获取所有的snapshot
3. 根据snapshot的名称进行解析。然后命名规则匹配，是我们要的那个snapshot_id，我们就插入新的数据到os表中。更新os表。

### 方案2
直接从cinder list中根据命名规则去取最新的volume_id，然后再解析，获取到他的volume_type和snapshot_id，然后发出post请求创盘。

| 方案简述| 优点| 缺点|
|---|---|---|
| 方案1  |简单，方便，直接读取数据库表就可以获取   | 一旦底层的镜像更新，导致备盘池中的备盘用的镜像不是最新的，如果直接删除备盘池，代价很大  |
| 方案2  | 不管管理员是否忘记sync_os，能够保证创建备盘时用的是最新的镜像  | 比较麻烦  |

缺点：可能备盘使用了旧的镜像，这样会与数据库os的不一致。->影响是什么？(镜像不一致)

需要备盘的镜像不存在怎么办
- os表无法获取，但cinder list可以获取
- os表、cinder list无法获取(无法备盘)


project归属问题
cld_api 根据project、


------
创盘流程修改：
create_volume_if_absent
非系统盘走原来的流程。
如果是系统盘：
从cinder list中根据volume_name来获取这个volume(GET请求)
  - 如果存在，不用创建，直接返回volume_id
  - 如果不存在，则查询备盘池表
      - 如果有多余的备盘，进行备盘转移cinder_transfer、cinder_accept，返回volume_id
      - 如果没有多余的备盘，调用create_volume(...)方法来创盘，返回volume_id



sync_os的时候，需要将备盘数据清空
