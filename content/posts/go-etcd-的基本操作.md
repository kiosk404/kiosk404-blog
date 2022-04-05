---
title: etcd 的基本入门
author: kiosk
tags:
  - etcd
categories:
  - devops
date: 2020-05-30 21:01:00
---
**etcd** 是一个强一致性的分布式键值存储系统，可以提供可靠的分布式集群的数据访问方式。

etcd is a strongly consistent, distributed key-value store that provides a reliable way to store data that needs to be accessed by a distributed system or cluster of machines.

refer: https://etcd.io/

<!--more-->

----------------------------------------

<img src="https://img1.kiosk007.top/static/images/etcd/etcd.png" />

# etcd简介

etcd 诞生于 CoreOS 公司，它最初是用于解决集群管理系统中 OS 升级的**分布式并发控制**、**服务发现**、**集群状态存储** 以及 **配置文件的存储与分发**等问题。基于此，etcd 被设计为提供高可用、强一致的小型 keyvalue 数据存储服务。

项目当前隶属于 CNCF 基金会，被 AWS、Google、Microsoft、Alibaba 等大型互联网公司广泛使用。

# 核心特性

- 数据存储在集群中的高可用K-V存储
- 允许应用实时监听存储中的K-V的变化
- 能够容忍单点故障，能够应对网络分区

> 分布式的容灾策略是基于 [鸽巢理论](https://www.cnblogs.com/ECJTUACM-873284962/p/7215942.html) ，假设一个班级有60人，我将一个秘密告诉31人，那么随便在这个班级挑出30人来，肯定至少有一个人指导这个秘密。

**etcd与Raft的关系**

Raft是强一致性的集群日志同步算法，etcd是一个分布式KV存储，etcd利用raft算法在集群中同步key-value 。etcd集群需要2N+1个节点。

<img src="https://img1.kiosk007.top/static/images/etcd/raft_a_consensus_algorithm_for_replicated_logs_1440-21.jpg" />
日志一旦由 leader节点 复制到了大多数 follower节点 即可完成提交。
上图可以看到7个节点的集群，log index 为9 的数据已经同步给 a,c,d 所以index 9 属于整个集群，index为11 的几个数据leader 不认，所以这个数据不属于集群。

>`Raft 可以保证，给客户端承诺过的请求一定是不会丢失的`
>`各个节点的数据一定是最终一致的` 

[raft 算法英文动画演示](http://thesecretlivesofdata.com/raft/)

# 交互协议

- HTTP 基于JSON请求，例如 curl，简单通用。
- SDK 内置GRPC协议，性能高效

# etcd 功能

refer： [etcd 文档演示](https://etcd.io/docs/v3.4.0/demo/)

- put、get、del 操作
    数据库普通的增加，查找，删除操作，如果想要更新，依旧使用put。

- watch 监听

   支持监听某一key的变化，有变化即产生回调，另外还支持查找key 的历史版本。

- lease 租约
    
   申请定时器，举例：申请一个TTL为10s的租约 lease，当 put 一个 key 时，携带该租约，当TTL到期时，key也会被删除，一个 lease id 可以关联多个key，也就是租约到期，多个key 可以被删除，想要防止被删除，可以用keepalive定期续租。
   
- txn 事物、loack 分布式锁
   etcd 支持将多个请求包装到一个事务中，或者添加一个分布式锁。

# 安装部署

- 安装etcd

``` bash
➜  wget https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz
➜  tar -xf etcd-v3.4.9-linux-amd64.tar.gz
➜  cd etcd-v3.4.9-linux-amd64/
➜  sudo cp etcd /usr/local/sbin/
➜  sudo cp etcdctl /usr/local/sbin/
```

- 设置集群信息

``` bash
TOKEN=token-01
CLUSTER_STATE=new
NAME_1=machine-1
NAME_2=machine-2
NAME_3=machine-3
HOST_1=127.0.0.1
HOST_2=127.0.0.1
HOST_3=127.0.0.1
CLUSTER=${NAME_1}=http://${HOST_1}:2381,${NAME_2}=http://${HOST_2}:2382,${NAME_3}=http://${HOST_3}:2383

```

- 启动集群

``` bash
# For machine 1
THIS_NAME=${NAME_1}
THIS_IP=${HOST_1}
etcd --data-dir=data.etcd1 --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2381 --listen-peer-urls http://${THIS_IP}:2381 \
	--advertise-client-urls http://${THIS_IP}:2371 --listen-client-urls http://${THIS_IP}:2371 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

# For machine 2
THIS_NAME=${NAME_2}
THIS_IP=${HOST_2}
etcd --data-dir=data.etcd2 --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2382 --listen-peer-urls http://${THIS_IP}:2382 \
	--advertise-client-urls http://${THIS_IP}:2372 --listen-client-urls http://${THIS_IP}:2372 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

# For machine 3
THIS_NAME=${NAME_3}
THIS_IP=${HOST_3}
etcd --data-dir=data.etcd3 --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2383 --listen-peer-urls http://${THIS_IP}:2383 \
	--advertise-client-urls http://${THIS_IP}:2373 --listen-client-urls http://${THIS_IP}:2373 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
```

- 使用etcdctl 连接到etcd

``` bash
export ETCDCTL_API=3
HOST_1=127.0.0.1
HOST_2=127.0.0.1
HOST_3=127.0.0.1
ENDPOINTS=$HOST_1:2371,$HOST_2:2372,$HOST_3:2373

etcdctl --endpoints=$ENDPOINTS member list
```

- 尝试添加一个key

``` bash
➜  etcdctl --endpoints=$ENDPOINTS put "key" "hello"
OK
```

# golang 操作 etcd

github 官方示例代码：https://github.com/etcd-io/etcd/tree/master/clientv3

使用前先安装 

``` bash
go get go.etcd.io/etcd/clientv3
```
解决Golang1.14 etcd/clientv3报错：etcd undefined: resolver.BuildOption

refer: https://blog.csdn.net/qq_43442524/article/details/104997539

## 连接

``` go
var client *clientv3.Client

func Conn() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"localhost:2371", "localhost:2372", "localhost:2373"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		// handle error!
	}
	defer cli.Close()
	client = cli
}
```

## 简单操作 GET，PUT，DEL

refer：https://godoc.org/go.etcd.io/etcd/clientv3

etcd的操作强大在可以添加 `WithOption` 比如 `WithPrevKV()` 获取前一个Key 和 `WithPrefix()` 遍历

``` go
type EtcdApi struct {
	KV clientv3.KV
}

func (e EtcdApi) PutKey(key string, value string) error {
	ctx, _ := context.WithTimeout(context.Background(),2 * time.Second)

// clientv3.WithPrevKV() 获取到删KV之前的值
	if putResp, err := e.KV.Put(ctx,key,value,clientv3.WithPrevKV()); err != nil {
		return err
	}else {
		if putResp.PrevKv != nil {
			fmt.Println("PrevValue: ", string(putResp.PrevKv.Value))
		}
		return nil
	}
}

func (e EtcdApi) GetKey(key string) (value string, err error) {
	ctx, _ := context.WithTimeout(context.Background(),2 * time.Second)

// 可以遍历目录
	if getResp, err := e.KV.Get(ctx,key,clientv3.WithPrefix());err != nil {
		return "", err
	}else {
		return fmt.Sprintf("%s", getResp.Kvs),err
	}
}

func (e EtcdApi) DelKey(key string) (value string, err error) {
	ctx, _ := context.WithTimeout(context.Background(),2 * time.Second)

	if delResp, err := e.KV.Delete(ctx,key,clientv3.WithPrevKV());err != nil {
		return value,nil
	}else {
		return fmt.Sprintf("%s",delResp.PrevKvs),nil
	}
}


```

## Lease 租期

- 创建租约


``` go
func (e EtcdApi) GrantLease() (lease clientv3.Lease, leaseId clientv3.LeaseID,err error) {
	var leaseGrantResp *clientv3.LeaseGrantResponse
	
	lease = clientv3.NewLease(e.Client)

	// 申请一个10s的租约
	if leaseGrantResp, err = lease.Grant(e.Ctx,10); err != nil {
		return 
	}

	leaseId = leaseGrantResp.ID

	// Put 一个KV, 关联租约，从而实现10s自动过期
	if putResp,err := e.KV.Put(e.Ctx,"/key1/lease","v1",clientv3.WithLease(leaseId)); err != nil {
		return 
	}else {
		fmt.Println(putResp.Header.Revision)
		return 
	}
}
```

- 续租

``` go
func (e *EtcdApi) KeepAlive(leaseId clientv3.LeaseID) {
	// 自动续期
	var keepResp *clientv3.LeaseKeepAliveResponse
	keepRespChan,err := e.Lease.KeepAlive(context.TODO(), leaseId)
	if err != nil {
		log.Fatal("ERROR :", err)
	}

	go func() {
		for {
			select{
			case keepResp = <- keepRespChan:
				if keepRespChan == nil {
					fmt.Println("租约实效")
					goto END
				}else { // 每秒续租一次
					fmt.Println("续租", keepResp.ID)
				}
			}
		}
	END:
		fmt.Println("Return")
	}()
}
```

## Watch 监听

``` go
func (e *EtcdApi) Watch() {
	// 创建一个watcher
	e.Watcher = clientv3.Watcher(e.Client)

	// 30 s 后取消监听
	ctx, cancelFunc := context.WithCancel(context.TODO())
	time.AfterFunc(30*time.Second, func() {
		cancelFunc()
	})

	// 启动监听
	watchRespChan := e.Watcher.Watch(ctx,"/key1/watcher")

	// 处理kv变化事件
	for watchResp := range watchRespChan {
		for _,event := range watchResp.Events {
			switch event.Type {
			case mvccpb.PUT: {
				fmt.Println(" 修改为： ", string(event.Kv.Value), "Revision: ", event.Kv.CreateRevision , event.Kv.ModRevision)
			}
			case mvccpb.DELETE: {
				fmt.Println(" 删除： Revision: ", event.Kv.ModRevision)
			}
			}
		}
	}
}
```

## Operation

可以将上面提到的 `Get`、`Put`、`Delete`等操作抽象成一个Operation。

``` go
func (e EtcdApi) Op() {
	// 创建一个OP：operation
	putOp := clientv3.OpPut("/key1/op","value_op")

	// 执行OP
	opResp, err := e.KV.Do(e.Ctx,putOp)
	if err != nil {
		fmt.Println(err)
		return
	}

	fmt.Println("写入Revision： ", &opResp.Put().Header.Revision)
}
```

## Txn
``` go
// 处理业务
func (e *EtcdApi) DoTxn(leaseId clientv3.LeaseID) {
	// 创建事物
	txn := e.KV.Txn(e.Ctx)
	
	// 定义事物
	txn.If(clientv3.Compare(clientv3.CreateRevision("/key1/lease"),"=","0")).
		Then(clientv3.OpPut("/key1/lease","value1",clientv3.WithLease(leaseId))).
		Else(clientv3.OpGet("/key1/lease"))  //抢锁失败
	
	txnResp,err := txn.Commit()
	if err != nil {
		log.Fatal(err.Error())
	}
	
	// 是否抢到锁
	if !txnResp.Succeeded {
		fmt.Println("抢锁失败,锁被占用：" , string(txnResp.Responses[0].GetResponseRange().Kvs[0].Value))
		return
	}
}
```