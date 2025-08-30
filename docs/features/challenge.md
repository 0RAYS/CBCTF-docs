---
sidebar_position: 2
---

# 题目类型

CBCTF 支持多种题目类型，包含 问答题（选择题），静态题，动态附件题，容器题

## 问答题

![question.png](img/question.png)

## 静态题

此时，题目附件，flag（需为 `static{}`） 均为固定不变

![static.png](img/static.png)

## 动态附件题

此时，题目附件，flag 均可为动态生成，需要出题人自行编写符合要求的 docker 镜像，CBCTF 将会在每次部署题目时，启动对应的容器，并执行附件生成脚本

![dynamic.png](img/dynamic.png)

## 容器题

此时，题目附件固定，flag 可为动态生成，出题人编写 `docker-compose.yaml` 即可配置单一容器或多容器题目

![container.png](img/container.png)
