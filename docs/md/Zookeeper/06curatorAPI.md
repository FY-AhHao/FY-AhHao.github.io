## curator API

### 常用方法

[源码地址](https://github.com/FY-AhHao/learning-code/tree/main/zookeeperDemo/src/main/java/com/fy/zookeeper/case2)

引入依赖

```xml
<dependency>
  <groupId>org.apache.zookeeper</groupId>
  <artifactId>zookeeper</artifactId>
  <version>3.6.3</version>
</dependency>
<dependency>
  <groupId>org.apache.curator</groupId>
  <artifactId>curator-framework</artifactId>
  <version>5.2.1</version>
</dependency>
<dependency>
  <groupId>org.apache.curator</groupId>
  <artifactId>curator-recipes</artifactId>
  <version>5.2.1</version>
</dependency>
<dependency>
  <groupId>org.apache.curator</groupId>
  <artifactId>curator-client</artifactId>
  <version>5.2.1</version>
</dependency>
```

```java
public class CuratorAPI {

    private Logger logger = LoggerFactory.getLogger(CuratorAPI.class);
    private static final String connectString = "192.168.31.132:2181,192.168.31.98:2181,192.168.31.223:2181";
    private static final int sessionTimeout = 2000;
    private static final int connectionTimeout = 2000;

    /**
     * 获取连接
     *
     * @return
     */
    public CuratorFramework getConnection() {
        ExponentialBackoffRetry retryPolicy = new ExponentialBackoffRetry(1000, 3);
        return CuratorFrameworkFactory.newClient(connectString, sessionTimeout, connectionTimeout, retryPolicy);
    }

    /**
     * 获取孩子节点
     *
     * @param path
     */
    public void getChildren(String path) {
        CuratorFramework connection = getConnection();
        try {
            connection.start();
            List<String> children = connection.getChildren().forPath(path);
            logger.info("孩子列表：{}", children);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connection.close();
        }
    }

    /**
     * 创建节点
     *
     * @param path
     * @param value
     */
    public void create(String path, byte[] value) {
        CuratorFramework connection = getConnection();
        try {
            connection.start();
            String s = connection.create()
                    .creatingParentsIfNeeded()
                    .withMode(CreateMode.PERSISTENT)
                    .forPath(path, value);
            logger.info("创建结果：{}", s);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connection.close();
        }
    }

    /**
     * 获取节点值
     *
     * @param path
     */
    public void getValue(String path) {
        CuratorFramework connection = getConnection();
        try {
            connection.start();
            byte[] bytes = connection.getData().forPath(path);
            logger.info("节点值：{}", new String(bytes));
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connection.close();
        }
    }

    /**
     * 设置节点值
     *
     * @param path
     * @param value
     */
    public void setValue(String path, byte[] value) {
        CuratorFramework connection = getConnection();
        try {
            connection.start();
            connection.setData().forPath(path, value);
            logger.info("设置成功");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connection.close();
        }
    }

    /**
     * @param path
     * @param value
     */
    public void setValueAsync(String path, byte[] value) {
        CuratorFramework connection = getConnection();
        try {
            connection.start();
            connection.setData()
                    .inBackground((curatorFramework, curatorEvent) -> {
                        logger.info("type:{},code:{}",curatorEvent.getType(),curatorEvent.getResultCode());
                    })
                    .forPath(path, value);
            logger.info("设置成功");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connection.close();
        }
    }

    /**
     * 删除节点
     *
     * @param path
     */
    public void delete(String path) {
        CuratorFramework connection = getConnection();
        try {
            connection.start();
            connection.delete().forPath(path);
            logger.info("删除成功");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connection.close();
        }
    }

    /**
     * 节点是否存在
     *
     * @param path
     */
    public void exists(String path) {
        CuratorFramework connection = getConnection();
        try {
            connection.start();
            Stat exists = connection.checkExists().forPath(path);
            logger.info("{} -- {}", path, exists == null ? "不存在" : "存在");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connection.close();
        }
    }
}
```

### 事件监听

[源码地址](https://github.com/FY-AhHao/learning-code/tree/main/zookeeperDemo/src/main/java/com/fy/zookeeper/case3)

简单监听，通过调用方法在控制台上看到监听只执行了一次

```java
public void simpleWatcher(String path){

        CuratorFramework connection = getConnection();
        try {
            connection.start();

            Stat stat = connection.checkExists().forPath(path);
            if (stat == null) {
                connection.create().forPath(path,"init data".getBytes());
            }

            byte[] bytes = connection.getData()
                    .usingWatcher((Watcher) (watchedEvent) -> {
                        logger.info("监听到数据发生变化，type:{},state:{},path:{}",watchedEvent.getType(),watchedEvent.getState(),watchedEvent.getPath());
                    })
                    .forPath(path);
            logger.info("初始节点数据：{}", new String(bytes));


            connection.setData().forPath(path,"first update data".getBytes());
            connection.setData().forPath(path,"second update data".getBytes());

            connection.delete().forPath(path);

            Thread.sleep(Long.MAX_VALUE);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connection.close();
        }

    }
```

缓存监听

TODO

## 参考

[尚硅谷-zookeeper视频](https://www.bilibili.com/video/BV1to4y1C7gw?spm_id_from=333.999.0.0)

[菜鸟教程-zookeeper](https://www.runoob.com/w3cnote/zookeeper-tutorial.html)

[zookeeper官网](https://zookeeper.apache.org/)

[curator官网](https://curator.apache.org)