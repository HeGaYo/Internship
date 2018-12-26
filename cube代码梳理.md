
## 代码阅读
1. skyline/view_v2.py:160
   用户发起创建server的POST请求
2. skyline.cld.server.ServerService#create_server
  调用创建server的函数。
  - 检查参数
  - get_quota
  - 根据数量，构建server_list
    - 数据库Server表插入
    - send_api_task(SERVER, server.id, server.args)
3. skyline/cld/base.py:224
4. skyline/cld/base.py:240
name=API_TASK_NAME
api_args = [resource_type, resource_id, resource_args.get('args')]
5. skyline.controller.cld_scheduler
6. exectuion, execution.start(),
  - get_next_task_args()
  - get_resource_args()
  - send_task(...)
7. worker注册任务, register_server_task(self)
    skyline.command.server.InstanceCommand
8. skyline.command.server.InstanceCommand#create_volume
  - 获取创建参数
  - 如果server存在就直接跳过，返回成功创建server的result
  - 如果ebs_type不存在，也报错。ebs_type是什么
  - skyline.common.util.server.ServerUtil#create_volume_if_absent，创建三种类型的volume，包括sys, data, swap
9. skyline.common.util.server.ServerUtil#create_volume_if_absent
  - 判断是否存在volume_id，如果存在，就直接返回voluem_id，不用继续创建
  - 创建volume，然后返回result['volume']['id']
10. skyline/common/util/server.py:181
  根据类型(sys为一类, data与swap为一类)的不同生成volume_data
  通过CINDER post请求创建volume。
11. skyline/common/util/server.py:151
  volume_type(necessary), snapshot_id(necessary) and volume_size(necessary)


```python
skyline/cld/server.py:96
如何获取snapshot_id
@classmethod
def get_os_by_name(cls, name, ebs_type):
    ebs_type = ebs_type.lower()
    _os = None
    if name:
        _os = current_session.query(Os).filter_by(name=name, type=ebs_type).first()
    return _os

if _os and _os.snapshot_id:
    server_args['snapshot_id'] = _os.snapshot_id


data_volume_id = ServerUtil.create_volume_if_absent(
            cld_api,
            hostname,
            'data',
            size=server_args.get("ebs_size", DEFAULT_SYS_SIZE),
            volume_type=data_volume_type)


@classmethod
def create_volume_if_absent(cls, cld_api, hostname, volume_category,
                            **kwargs):
    volume_name = "%s_%s" % (hostname, volume_category)
    volume_id = ServerUtil.get_volume_by_name(cld_api, volume_name)

    if volume_id is not None:
        return volume_id
    result = cls.create_volume(cld_api, hostname, volume_category,
                               **kwargs)
    return result['volume']['id']


@classmethod
def create_volume(cls, cld_api, hostname, volume_partition, **kwargs):
    if volume_partition == 'sys':
        volume_data = cls.get_volume_data(hostname, volume_partition,
                                          **kwargs)
    elif volume_partition == 'data' or volume_partition == 'swap':
        volume_data = cls.get_volume_data(hostname, volume_partition,
                                          **kwargs)
    else:
        raise WorkArgsError(
            "volume_partition %s is not support" % volume_partition)

    url = "volumes"
    return cld_api.post(CINDER, url, volume_data)


@classmethod
def get_volume_data(cls, hostname, volume_partition, **kwargs):
    volume_size = kwargs.get('size')
    snapshot_id = kwargs.get('snapshot_id')
    volume_type = kwargs.get('volume_type')

    if snapshot_id is None and volume_size is None:
        raise WorkArgsError("create_volume need size or snapshot_id "
                            "is not None")
    if volume_type is None:
        raise WorkArgsError("create_volume need volume_type is not None")

    if volume_partition == 'data':
        size = volume_size
    elif volume_partition == 'swap':
        size = volume_size
    else:
        size = None
    volume_data = {
        "volume": {
            "size": size,
            "multiattach ": False,
            "snapshot_id": snapshot_id,
            "name": "%s_%s" % (hostname, volume_partition),
            "volume_type": volume_type,
            "metadata": {},
        },
    }
    return volume_data
```

## cube备盘池方案
1. 只需要备系统盘吧
   |操作| 周期|
   |---|---|
   backup_volumes

   check_volume_state


2. 修改`create_volume_if_absent`的逻辑
创建volume的逻辑：
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


volume_data = {
            "volume": {
                "size": size,
                "multiattach ": False,
                "snapshot_id": snapshot_id,
                "name": "%s_%s" % (hostname, volume_partition),
                "volume_type": volume_type,
                "metadata": {},
            },
        }

url = "volumes"
return cld_api.post(CINDER, url, volume_data)
