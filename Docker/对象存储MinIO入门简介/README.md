# 对象存储 MinIO 入门

## 常见的对象存储方式对比

- 文件直接落在应用所在机器的磁盘
  - 好处：实现快，几乎没有额外组件
  - 限制：磁盘满了就要停机搬家，多机之间也不能共享这一份文件
- 分布式文件系统
  - 好处：加节点就能扩容量
  - 限制：接口和运维都更重，做一个完整的上传下载还要处理副本和故障
- NFS
  - 好处：挂载之后程序仍按普通目录读写
  - 限制：要会做导出、权限和网络隔离，单点故障时所有客户端一起受影响
- 云厂商的对象存储
  - 好处：上传、权限、CDN 都是现成的，不用自己值守磁盘
  - 限制：按量计费，数据出云还要看流量价格

## 为何不采取 FastDFS 进行文件存储

FastDFS 能做分布式文件，落地时常见三个绊脚石：

- 缺少一份持续更新的正式文档，步骤散落在各处，版本对不上就失败
- 至少要分别起跟踪节点和存储节点，容器数量和启动顺序都多
- 权限模型简单，暴露在公网时不好做细粒度的访问控制

对象存储用桶和对象这两级来组织文件，HTTP 接口也更接近现在的应用写法，所以下面改用 MinIO。

## 对象存储 MinIO

MinIO 是一套开源的对象存储服务，兼容常见的 S3 接口，可以装在普通服务器上。官方基准在高配硬件上给出过很高的吞吐，具体数字随版本和磁盘变化，把它理解成“适合当数据湖或图片视频底库”即可。Spark、Presto 一类计算引擎可以把这里当成数据来源，也可以在一部分场景里代替 HDFS 存放对象。

它按 Apache 2.0 发布，单机就能起，多盘、多节点时再扩成分布式。中文文档入口：https://docs.min.io/community/minio-object-store/index.html

## 下载

用容器安装。先拉镜像，再准备数据和配置目录：

```bash
docker pull minio/minio:RELEASE.2024-10-13T13-34-11Z
mkdir -p /srv/objstore/data /srv/objstore/config
```

数据和配置分开挂，是为了以后只备份数据目录，或者只重置配置。

## 启动容器

前台启动方便看日志。口令只作练习，上线前换成足够长的随机串。新版本用根用户变量，不再使用旧的 `MINIO_ACCESS_KEY` 这一对名字。

```bash
docker run --name objstore \
  -p 19000:9000 \
  -e MINIO_ROOT_USER=labadmin \
  -e MINIO_ROOT_PASSWORD='lab-secret-88421' \
  -v /srv/objstore/data:/data \
  -v /srv/objstore/config:/root/.minio \
  minio/minio:RELEASE.2024-10-13T13-34-11Z server /data --console-address ":9001"
```

日志里会出现两行地址，API 在 9000，控制台在 9001。容器内的 9001 还没映射时，浏览器进不去，需要补上端口。

放到后台并同时打开控制台：

```bash
docker run -d --name objstore \
  --restart unless-stopped \
  -p 19000:9000 -p 19001:9001 \
  -e MINIO_ROOT_USER=labadmin \
  -e MINIO_ROOT_PASSWORD='lab-secret-88421' \
  -v /srv/objstore/data:/data \
  -v /srv/objstore/config:/root/.minio \
  minio/minio:RELEASE.2024-10-13T13-34-11Z server /data --console-address ":9001"
```

登录要用到的两项：

- `MINIO_ROOT_USER`：根用户名
- `MINIO_ROOT_PASSWORD`：根口令

云主机上记得在安全组放行 19000 和 19001，并且不要对公网开放弱口令。

## 访问

控制台：

```text
http://10.8.20.31:19001
```

用上面的用户名和口令进入。API 端口 `19000` 留给 SDK，不要和网页端口混用。

## 创建 bucket

桶可以理解成独立的顶层目录，权限和生命周期都按桶设置。在控制台左下角的创建按钮里选 Create Bucket，名称填 `media-lab`。创建后侧边栏会出现这个桶。名称一旦定下，应用配置里要与它完全一致。

## 上传文件

选中 `media-lab`，再点上传，选择本地文件。完成后对象列表里能看到文件名、大小和修改时间。到这一步，磁盘上的 `/srv/objstore/data` 里已经有对应对象，容器删掉只要卷还在，文件就还在。

## Spring Boot 整合 MinIO

Java 客户端的写法以当前版本文档为准。下面用 8.5.x 的构建器风格，避免新旧构造方法混在一起。

### 修改权限

SDK 写入之前，桶策略要允许对应的访问密钥。练习阶段可以在 `media-lab` 的策略里给指定前缀读写。对匿名公开读要单独评估，图片若含隐私，应改用带过期时间的预签名链接，而不是把整桶设成公开。

### 添加依赖

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.12</version>
</dependency>
```

### 添加 application.yml

```yaml
server:
  port: 8091
spring:
  application:
    name: media-uploader

objstore:
  endpoint: http://10.8.20.31:19000
  access-key: labadmin
  secret-key: lab-secret-88421
  bucket: media-lab
```

`endpoint` 填 API 端口，不是控制台端口。

### 添加配置文件

把客户端放进容器，业务代码只注入这一个 Bean。

```java
@Configuration
public class ObjectStoreConfig {
    @Value("${objstore.endpoint}")
    private String endpoint;
    @Value("${objstore.access-key}")
    private String accessKey;
    @Value("${objstore.secret-key}")
    private String secretKey;

    @Bean
    public MinioClient objectClient() {
        return MinioClient.builder()
                .endpoint(endpoint)
                .credentials(accessKey, secretKey)
                .build();
    }
}
```

### 编写 Controller

```java
@RestController
public class MediaController {
    @Autowired
    private MinioClient objectClient;

    @PostMapping("/upload")
    public String upload(@RequestParam("data") MultipartFile data) throws Exception {
        String objectName = data.getOriginalFilename();
        objectClient.putObject(
                PutObjectArgs.builder()
                        .bucket("media-lab")
                        .object(objectName)
                        .stream(data.getInputStream(), data.getSize(), -1)
                        .contentType(data.getContentType())
                        .build());
        return objectName;
    }

    @PostMapping("/download")
    public String download(@RequestParam("name") String name) throws Exception {
        return objectClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                        .method(Method.GET)
                        .bucket("media-lab")
                        .object(name)
                        .expiry(7, TimeUnit.DAYS)
                        .build());
    }
}
```

上传返回对象名，下载返回一段七天内有效的地址。文件名应再做清洗，避免带有 `../` 之类的路径。

### 测试图片上传

启动应用后，用接口工具发 `POST /upload`。请求体选 form-data，字段名 `data`，类型为文件。请求头由工具根据 form-data 自动带上：

```text
Content-Type: multipart/form-data
```

返回对象名后，刷新控制台的 `media-lab`，应能看到刚传上去的文件。预签名地址可以直接在浏览器打开。若桶没有公开读，下面这种不带签名的路径会失败，这是预期行为：

```text
http://10.8.20.31:19000/media-lab/cover-2048.png
```

## 业务系统接入 MinIO

应用侧通常要填五项，名称随系统不同，含义固定：

- 对外访问域名：用户浏览器加载文件时用的地址，可以是反代后的域名
- 连接地址：SDK 使用的 endpoint，对应上面的 API 端口
- 访问密钥：根用户或单独建的访问密钥
- 秘密密钥：与访问密钥成对
- 桶名称：必须已经在控制台里建好，例如 `media-lab`

填完后到控制台确认桶存在，并把策略调成应用账号可读写。匿名可读只开给确实要公开的前缀。

保存配置后，在后台做一次图片上传。页面能显示图片，同时控制台对象列表里出现同一文件，才算接入完成。上传失败时先核对 endpoint 端口、时钟和密钥，再看应用日志里的 HTTP 状态码。
