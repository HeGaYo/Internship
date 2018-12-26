nfvi更新文档操作手册-变更记录
20181219变更记录
- 变更版本：20181219_pre_release

- 变更内容
 - 新增备盘池功能，对igw, bgw, cast, dhcp进行提前备盘
 - 新增数据库表 backup_volumes

- 变更步骤
1. 配置文件新增

  ```
  [DEFAULT]
  ...
  volume_queue=volume

  [volumes]
  concurrency=2

  [volumes_backup]
  backup_nums_igw_bgw=10
  backup_nums_dhcp_cast=2
  volume_backup_types=igw,bgw,dhcp,cast
  backup_frequency_minute = 10
  check_state_frequency_minute = 5
  create_volume_time_interval_second = 20

  [auth_args]
  auth_project=admin
  auth_user=admin
  auth_passwd=admin
  ```


3. 更新数据库，新增数据库表。

  ```sql
  USE nfvi;

  CREATE TABLE backup_volumes (
      id INT NOT NULL AUTO_INCREMENT ,
      volume_id       VARCHAR (64),
      name            VARCHAR (64),
      type            VARCHAR (64),
      status          VARCHAR (64),
      snapshot        VARCHAR (64),
      allocated       TINYINT,
      create_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      modify_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
      delete_at       TIMESTAMP NULL DEFAULT NULL,
      deleted         TINYINT,
      PRIMARY KEY (id)
  );
  ```
3. 停止原来的服务，拉取新的服务。更新脚本`restart_docker.sh`
  ```bash
  #!/usr/bin/env bash

  tag="20181219_pre_release"
  docker stop nfvi-worker
  docker stop nfvi-controller
  docker stop nfvi-api
  docker stop nfvi-volumes
  docker rm nfvi-worker
  docker rm nfvi-controller
  docker rm nfvi-api
  docker rm nfvi-volumes
  mkdir -p  /var/log/nfvi/
  docker run -d --name=nfvi-api --net=host -v /home/conf/nfvi/nfvi.conf:/etc/nfvi/nfvi.conf -v /var/log/nfvi/:/var/log/nfvi/ dockerhub.nie.netease.com/scheduler/nfvi:${tag} --module api --config-file=/etc/nfvi/nfvi.conf --log-file=/var/log/nfvi/nfvi-api.log
  docker run -d --name=nfvi-controller --net=host -v /home/conf/nfvi/nfvi.conf:/etc/nfvi/nfvi.conf -v /var/log/nfvi/:/var/log/nfvi/ dockerhub.nie.netease.com/scheduler/nfvi:${tag}  --module controller --config-file=/etc/nfvi/nfvi.conf --log-file=/var/log/nfvi/nfvi-controller.log
  docker run -d --name=nfvi-worker --net=host -v /home/conf/nfvi/nfvi.conf:/etc/nfvi/nfvi.conf -v /var/log/nfvi_test/:/var/log/nfvi/ dockerhub.nie.netease.com/scheduler/nfvi:${tag} --module worker --config-file=/etc/nfvi/nfvi.conf --log-file=/var/log/nfvi/nfvi-worker.log
  docker run -d --name=nfvi-volumes --net=host -v /home/conf/nfvi/nfvi.conf:/etc/nfvi/nfvi.conf -v /var/log/nfvi/:/var/log/nfvi/ dockerhub.nie.netease.com/scheduler/nfvi:${tag} --module volumes --config-file=/etc/nfvi/nfvi.conf --log-file=/var/log/nfvi/nfvi-volumes.log
  ```
4. 运行更新脚本
```
bash restart_docker.sh
```
5. 找QA进行测试
