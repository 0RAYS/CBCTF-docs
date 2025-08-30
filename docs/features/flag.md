---
sidebar_position: 1
---

# 动态Flag

CBCTF 自带对动态附件题目和容器题目 flag 分发的支持，支持多个 flag 注入同时注入

## 配置规则

flag 生成规则有如下三种，当采用符合格式 flag 时，将会自动替换前缀为比赛所设置的前缀，一下所有示例前缀均为 `CBCTF`

1. 静态 flag

   静态 flag 适用于所有类型题目，flag 内容固定不变

   ```text
   static{this_is_a_static_flag}
   ```
   
   当被添加至比赛时，flag 将会变为

   ```text
   CBCTF{this_is_a_static_flag}
   ```

2. 动态 flag

   动态 flag 适用于所有类型题目，flag 内容基于模板随机生成，保持可读性，**请注意：flag 的长度可能会发生变化，请确保题目不会因为长度变化导致问题**

   ```text
   dynamic{this_is_a_dynamic_flag}
   ```

   当被添加至比赛时，flag 将会变为

   ```text
   CBCTF{ThiliS-iS-4_Dyn4MIC_FLaG}
   ```

3. UUID flag

   UUID flag 适用于所有类型题目，flag 内容为标准 UUID 格式

   ```text
   uuid{}
   ```

   当被添加至比赛时，flag 将会变为

   ```text
   CBCTF{550e8400-e29b-41d4-a716-446655440000}
   ```

## 注入方式

针对动态附件题目与容器题目，使用两种不同的方式进行 flag 注入

### 动态附件题目

动态附件题目的 flag，将有平台在生成器容器中，自动调用 `/root/run.sh` 脚本进行注入，该脚本需要出题人进行编写。调用该脚本时执行的命令如下
    
   ```bash
   # `team_id` 为队伍 ID，类型 int
   /root/run.sh team_id base64(base64(flag1),base64(flag2),...)
   ```
   
   示例：
   ```bash
   # team_id: 1
   # flag1:   CBCTF{flag1}
   # flag2:   CBCTF{flag2}
   /root/run.sh 1 UTBKRFZFWjdabXhoWnpGOSxRMEpEVkVaN1pteGhaeko5
   ```
   
### 容器题目

容器题目的 flag，将由平台在容器启动时，自动注入至容器内，支持两种注入方式：

1. 依据环境变量进行注入，环境变量的名称由出题人编写题目 `docker-compose.yaml` 时指定，**必须以 `FLAG_` 作为前缀**。示例如下：

   ```yaml
   services:
     web:
       image: nginx:latest
       environment:
         - FLAG_1=dynamic{this_is_a_dynamic_flag}
         - FLAG_2=static{this_is_a_static_flag}
         - FLAG_3=uuid{}
       ports:
         - "80:80"
   ```

2. 依据文件进行注入，文件路径由出题人编写题目 `docker-compose.yaml` 时指定。示例如下：

   ```yaml
   services:
     web:
       image: nginx:latest
       volumes:
         - FLAG_1:/flags/flag1.txt
         - FLAG_2:/flags/flag2.txt
       ports:
         - "80:80"

   volumes:
     FLAG_1:
       labels:
         - value=uuid{}
     FLAG_2:
       labels:
         - value=dynamic{this_is_a_dynamic_flag}
   ```

## flag 生成节点

队伍的 flag 并非每次启动容器或下载附件时进行随机，只有当题目进行初始化或重置题目时才会随机生成
