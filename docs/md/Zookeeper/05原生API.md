## 原生api

### 常用方法

[源码地址]()

引入依赖

```xml
<dependency>
  <groupId>org.apache.zookeeper</groupId>
  <artifactId>zookeeper</artifactId>
  <version>3.6.3</version>
</dependency>
```

```java
/**
 * @description: 原生api
 * @author：AhHao
 * @date: 2022/3/20
 */
public class NativeAPI {

    private Logger logger = LoggerFactory.getLogger(NativeAPI.class);
    private static final String connectString = "192.168.31.132:2181,192.168.31.98:2181,192.168.31.223:2181";
    private static final int sessionTimeout = 2000;
    private final CountDownLatch countDownLatch = new CountDownLatch(1);
    private ZooKeeper zooKeeper;

    /**
     * 获取连接
     */
    public void getConnection() {
        try {
            zooKeeper = new ZooKeeper(connectString, sessionTimeout, event -> {
                if (Watcher.Event.KeeperState.SyncConnected == event.getState()) {
                    //连接成功，放行
                    countDownLatch.countDown();
                }
            });
            //阻塞等待连接
            countDownLatch.await();

            logger.info("连接成功");
        } catch (IOException | InterruptedException e) {
            e.printStackTrace();
        }
    }

    /**
     * 获取孩子节点
     *
     * @param path
     */
    public void getChildren(String path) {
        try {
            List<String> children = zooKeeper.getChildren(path, false);
            logger.info("孩子节点：{}", children);
        } catch (KeeperException | InterruptedException e) {
            e.printStackTrace();
        }
    }

    /**
     * 创建节点
     *
     * @param path
     * @param value
     */
    public void create(String path, byte[] value) {
        String s = null;
        try {
            s = zooKeeper.create(path, value, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            logger.info("创建结果：{}", s);
        } catch (KeeperException | InterruptedException e) {
            e.printStackTrace();
        }
    }

    /**
     * 获取节点值
     *
     * @param path
     */
    public void getValue(String path) {
        try {
            byte[] data = zooKeeper.getData(path, false, null);
            String val = new String(data);
            logger.info("节点值：{}", val);
        } catch (KeeperException | InterruptedException e) {
            e.printStackTrace();
        }
    }

    /**
     * 设置节点值
     *
     * @param path
     * @param value
     */
    public void setValue(String path, byte[] value) {
        try {
            zooKeeper.setData(path, value, -1);
            logger.info("设置成功");
        } catch (KeeperException | InterruptedException e) {
            e.printStackTrace();
        }
    }


    /**
     * 删除节点
     *
     * @param path
     */
    public void delete(String path) {
        try {
            zooKeeper.delete(path, -1);
            logger.info("删除成功");
        } catch (InterruptedException | KeeperException e) {
            e.printStackTrace();
        }
    }

    /**
     * 节点是否存在
     *
     * @param path
     */
    public void exists(String path) {
        try {
            Stat exists = zooKeeper.exists(path, false);
            logger.info("{} -- {}", path, exists == null ? "不存在" : "存在");
        } catch (KeeperException | InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

### 服务注册发现demo

[源码地址]()

模拟服务的注册与发现，先启动客户端，一开始客户端打印的服务端列表是空的，启动服务端，观察客户端控制台，发现能够打印出刚刚注册的服务端

客户端

```java
public class DistributeClient {

    private Logger logger = LoggerFactory.getLogger(DistributeServer.class);
    private static final String connectString = "192.168.31.132:2181,192.168.31.98:2181,192.168.31.223:2181";
    private static final int sessionTimeout = 2000;
    private final CountDownLatch countDownLatch = new CountDownLatch(1);
    private ZooKeeper zooKeeper;
    private static final String parentNode = "/servers";

    /**
     * 获取连接
     * @return
     * @throws IOException
     */
    public void getConnection(){
        try {
            zooKeeper = new ZooKeeper(connectString, sessionTimeout, watchedEvent -> {
                if (Watcher.Event.KeeperState.SyncConnected == watchedEvent.getState()) {

                    try {
                        Stat exists = zooKeeper.exists(parentNode, false);
                        if (exists == null) {
                            //创建根节点
                            zooKeeper.create(parentNode, null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
                        }
                    } catch (KeeperException | InterruptedException e) {
                        e.printStackTrace();
                    }

                    //连接成功，放行
                    countDownLatch.countDown();
                }
                logger.info("watchedEvent: type:{},state:{}",watchedEvent.getType(),watchedEvent.getState());
                //再次启动监听
                getServerList();
            });

            //阻塞等待连接
            countDownLatch.await();
            logger.info("客户端获取连接成功");

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 获取服务端地址
     */
    public void getServerList(){
        try {
            //获取服务节点，开启监听
            List<String>  children = zooKeeper.getChildren(parentNode, true);
            //存储服务节点值
            List<String> hosts = new ArrayList<>(children.size());
            for (String child : children) {
                byte[] data = zooKeeper.getData(parentNode + "/" + child, false, null);
                hosts.add(new String(data));
            }
            logger.info("服务端地址：{}",hosts);
        } catch (KeeperException | InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws Exception {
        DistributeClient distributeClient = new DistributeClient();
        distributeClient.getConnection();
        distributeClient.getServerList();

        //模拟服务运行中
        Thread.sleep(Long.MAX_VALUE);
    }
}
```

服务端

```java
public class DistributeServer {

    private Logger logger = LoggerFactory.getLogger(DistributeServer.class);
    private static final String connectString = "192.168.31.132:2181,192.168.31.98:2181,192.168.31.223:2181";
    private static final int sessionTimeout = 2000;
    private final CountDownLatch countDownLatch = new CountDownLatch(1);
    private ZooKeeper zooKeeper;
    private static final String parentNode = "/servers";

    /**
     * 获取连接
     * @return
     * @throws IOException
     */
    public void getConnection(){
        try {
            zooKeeper = new ZooKeeper(connectString, sessionTimeout, watchedEvent -> {
                if (Watcher.Event.KeeperState.SyncConnected == watchedEvent.getState()) {

                    try {
                        Stat exists = zooKeeper.exists(parentNode, false);
                        if (exists == null) {
                            //创建根节点
                            zooKeeper.create(parentNode, null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
                        }
                    } catch (KeeperException | InterruptedException e) {
                        e.printStackTrace();
                    }

                    //连接成功，放行
                    countDownLatch.countDown();
                }
                logger.info("watchedEvent: type:{},state:{}",watchedEvent.getType(),watchedEvent.getState());
            });

            //阻塞等待连接
            countDownLatch.await();
            logger.info("服务端获取连接成功");

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 注册服务
     * @param host
     * @throws Exception
     */
    public void registerServer(String host) throws Exception{
        //创建短暂顺序节点
        String s = zooKeeper.create(parentNode + "/server", host.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        logger.info("{} {} 已注册 ",host,s);
    }


    public static void main(String[] args) throws Exception {
        DistributeServer distributeServer = new DistributeServer();
        distributeServer.getConnection();
        distributeServer.registerServer("node1");

        //模拟服务运行中
        Thread.sleep(Long.MAX_VALUE);
    }
}
```

## 参考

[尚硅谷-zookeeper视频](https://www.bilibili.com/video/BV1to4y1C7gw?spm_id_from=333.999.0.0)

[菜鸟教程-zookeeper](https://www.runoob.com/w3cnote/zookeeper-tutorial.html)

[zookeeper官网](https://zookeeper.apache.org/)

