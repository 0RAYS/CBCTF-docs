---
sidebar_position: 4
---

# 附件生成

## 静态附件

[问答题](/docs/features/challenge#问答题)，[静态题](/docs/features/challenge#静态题)，[容器题](/docs/features/challenge#容器题) 均支持上传静态附件供给选手下载

## 动态附件

由出题人编写 docker 镜像，CBCTF 将会在每次部署题目时，启动对应的容器，并执行指定脚本，生成的附件将会打包供选手下载

示例题目：[dynamic-attachment](https://github.com/0RAYS/CBCTF/tree/main/example/dynamic)

### 生成步骤

1. 平台生成动态 flag，获取队伍 ID
   
   - 以队伍 ID 作为附件唯一标识
   - 将一（多）个 flag 进行 base64 编码确保 flag 内容不会因为特殊字符导致脚本执行失败

2. 平台启动容器

   - 替代镜像默认的 entrypoint，执行 `sleep infinity` 保活
   - 容器静默运行等待生成附件指令

3. 平台上传 `generator.zip` 至容器内 `/root` 目录（可选：仅当出题人在平台上传 generator.zip 时执行此步骤。目的是当以来环境相同，仅执行脚本不同时的镜像构建工作量）

   - 解压 `generator.zip` 至 `/root` 目录

4. 平台执行 `/root/run.sh` 生成附件

   - 该脚本由出题人编写，脚本内需要包含生成附件的逻辑
   - 平台执行的命令如下：
      ```bash
      /root/run.sh team_id base64(base64(flag1),base64(flag2),...)
      # 例如
      # team_id: 1
      # flag1:   CBCTF{flag1}
      # flag2:   CBCTF{flag2}
      /root/run.sh 1 UTBKRFZFWjdabXhoWnpGOSxRMEpEVkVaN1pteGhaeko5
      ```
   - **附件生成位置必须为 `/root/mnt/attachments/{team_id}.zip`**

5. 定时重启容器

## 注意事项

1. 容器须包含 `sleep` `unzip` 命令

2. 附件生成路径固定为 `/root/mnt/attachments/{team_id}.zip`

3. 生成脚本入口固定为 `/root/run.sh`

4. flag 长度可能会发生变化，请确保题目不会因为长度变化导致问题

5. 平台只负责执行 `/root/run.sh` 脚本与复制 `/root/mnt/attachments/{team_id}.zip`，其余所有操作均由出题人编写 docker 完成，包括：附件生成，压缩附件等
