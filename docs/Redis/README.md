# Redis 

## Redis Single Install

-   Docker 

    ```bash
    # 拉取镜像
    docker pull redis:5.0.4
    # 启动镜像并指定密码
    docker run --name redis -d -p 6379:6379 redis --requirepass "password"
    # 登录 Redis 容器
    docker exec -it redis bash
    # 启动客户端
    redic-cli -a password
    ```

## Redis Cluster Install

-   Env

    -   Mac OS
    -   Docker 

-   Install

    -   docker 文件下执行 `docker-compose build`,然后执行 `docker-compose up`.

        `* Background AOF rewrite finished successfully` 表示集群启动成功.

-   Link

    [Mac上最简单明了的利用Docker搭建Redis集群](https://juejin.im/post/5cbd3c435188250a8b7cf55e)       

## Lettuce 

[Lettuce](https://lettuce.io/core/release/reference/#overview)

[Lettuce GitHub](https://github.com/lettuce-io/lettuce-core/wiki/About-Lettuce)



