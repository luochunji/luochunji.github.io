---
title: zookeeper客户端-java api简单使用
date: 2018-03-23 21:30:27
tags:
- zookeeper
- zkjavaApi
categories:
- zookeeper
---

参照zookeeper的官方文档，简单学习如何通过java api操作zookeeper。

<!-- more -->

### 引入依赖包

```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.8</version>
</dependency>
```

### 客户端实例创建

```java
public class ZookeeperInstance {
    //zk集群的IP:prot，用逗号分隔
    private static final String CONNECT_STRING = "192.168.96.129:2181,192.168.96.130:2181," +
            "192.168.96.131:2181,192.168.96.132:2181";
    
    //计数器
    private static CountDownLatch latch = new CountDownLatch(1);

    public static ZooKeeper getInstance() {
        ZooKeeper zooKeeper = null;
        try {
            //传入集群ip:port配置，超时时间，监听事件
            zooKeeper = new ZooKeeper(CONNECT_STRING, 5000, watchedEvent -> {
                //连接成功时的事件状态为SyncConnected
                if (watchedEvent.getState() == Watcher.Event.KeeperState.SyncConnected) {
                    System.out.println("zk状态:" + watchedEvent.getState());
                    latch.countDown();
                }
            });
            //等待客户端链接成功
            latch.await();
            System.out.println("zk启动成功");
        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return zooKeeper;
    }
}
```

客户端的几种连接状态：

```java
Disconnected(0),//未连接
SyncConnected(3),//已连接
AuthFailed(4),//认证失败
ConnectedReadOnly(5),//只读连接
SaslAuthenticated(6),//已认证
Expired(-112);//已过期
```

### Znode的增删改查

```java
public class Operator {

    private ZooKeeper zooKeeper;
    private OperatorListener listener = new OperatorListener();
    private List<ACL> acls;
    private Stat stat = new Stat();

    public Operator() {
        this.zooKeeper = ZookeeperInstance.getInstance();
        //权限初始化
        this.acls = new ArrayList<>();
        ACL acl = new ACL(ZooDefs.Perms.ALL, new Id("world", "anyone"));
        acls.add(acl);

    }

    /**
     * 创建节点
     * @param path 节点路径
     * @param value 节点值
     * @throws KeeperException
     * @throws InterruptedException
     */
    private void createZnode(String path,String value) throws KeeperException, InterruptedException {
        zooKeeper.create(path, value.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        TimeUnit.SECONDS.sleep(1);

    }

    /**
     * 删除节点
     * @param path 节点路径
     * @throws KeeperException
     * @throws InterruptedException
     */
    private void deleteZnode(String path) throws KeeperException, InterruptedException {
        zooKeeper.delete(path, -1);
    }

    /**
     * 更新节点
     * @param path 节点路径
     * @param value 节点值
     * @throws KeeperException
     * @throws InterruptedException
     */
    private void updateZnode(String path, String value) throws KeeperException, InterruptedException {
        zooKeeper.setData(path, value.getBytes(), -1);
    }

    /**
     * 获取节点
     * @param path 节点路径
     * @throws KeeperException
     * @throws InterruptedException
     */
    private void getNodeData(String path) throws KeeperException, InterruptedException {
        byte[] bytes = zooKeeper.getData(path, listener, stat);
        System.out.printf("节点:%s,数据:%s\n", path, new String(bytes));
    }

    /**
     * 节点是否存在
     * @param path 节点路径
     * @throws KeeperException
     * @throws InterruptedException
     */
    private void exists(String path) throws KeeperException, InterruptedException {
        Stat s = zooKeeper.exists(path, listener);
        System.out.println(null == s ? "节点" + path + "不存在\n" : "节点" + path + "存在\n");
    }

    /**
     * 获取子节点
     * @param path 节点路径
     * @throws KeeperException
     * @throws InterruptedException
     */
    private void getChild(String path) throws KeeperException, InterruptedException {
        List<String> childNodeList = zooKeeper.getChildren(path,listener);
        System.out.println("子节点列表："+ Arrays.toString(childNodeList.toArray()));
    }


    public static void main(String[] args) throws KeeperException, InterruptedException {
        Operator operator = new Operator();
        String path = "/myNode";
        String child = path +"/moNode-1";
        //新建节点
        operator.exists(path);
        operator.createZnode(path,"123");
        operator.getNodeData(path);

        //新建子节点
        operator.getChild(path);
        operator.createZnode(child,"123");
        operator.getNodeData(path);

        operator.updateZnode(path, "456");
        operator.getNodeData(path);

        operator.deleteZnode(child);
        operator.deleteZnode(path);
        operator.exists(path);

    }

}
```

创建节点时，节点模式4种可选

```java
    PERSISTENT, //持久节点
    PERSISTENT_SEQUENTIAL,//持久有序节点
    EPHEMERAL,//临时节点
    EPHEMERAL_SEQUENTIAL;//临时有序节点
```

### Znode的监听

```java
/**
     * 监听类，继承Watcher，zk api提供getData(),exists()和getChildren()
     * 三个方法对Znode进行监听
     *
     */
    class OperatorListener implements Watcher {

        @Override
        public void process(WatchedEvent watchedEvent) {
            switch (watchedEvent.getType()) {
                case None:
                    System.out.println("连接成功");
                    break;
                case NodeCreated:
                    System.out.println("创建节点:"+watchedEvent.getPath());
                    break;
                case NodeDeleted:
                    System.out.println("删除节点:"+watchedEvent.getPath());
                    break;
                case NodeDataChanged:
                    try {
                        System.out.println("修改节点:"+watchedEvent.getPath());
                        getNodeData(watchedEvent.getPath());
                    } catch (KeeperException e) {
                        e.printStackTrace();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    break;
                case NodeChildrenChanged:
                    System.out.println("创建子节点:"+watchedEvent.getPath());
                    break;
                default:
                    break;
            }
        }
    }
```

事件的几种状态

```java
None,//连接成功的初始状态
NodeCreated,//节点创建时触发
NodeDeleted,//节点删除时触发
NodeDataChanged,//节点数据更新时触发
NodeChildrenChanged;//子节点变化时触发
```