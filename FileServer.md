# 该项目实现的功能

# 系统架构

![structure](/Users/chenlei/Downloads/filestore-server/doc/structure.png)



![microservice_interact_archi](/Users/chenlei/Downloads/filestore-server/doc/microservice_interact_archi.png)

# 数据库设计
1. tbl_file  自增主键  filehash设置唯一索引
2. tbl_user  自增主键  username设置唯一索引
3. tbl_user_token  自增主键  username设置唯一索引
4. tbl_user_file  自增主键， username设置普通索引， 唯一所以（filehash, username）； 

设计分表问题：
水平分，垂直分
自增主键冲突了怎么办，
可以利用redis生成唯一全局唯一id，或者利用uuid，雪花算法。

# 用户注册

用户注册需要提供用户名和密码，提交post表单，后台获取到username和password然后对password用sha1加密，保存到user_tbl中。

为什么保存密码需要加密：!

如果数据库丢失，明文保存的话用户账号不安全

还有没有其他更安全的做法：

可以为每个用户生成随机的salt值，然后利用sha1加密

# 用户登陆

1. 校验密码是否正确
2. 生成token
3. 获取用户信息（username， signup time）
4. 获取用户文件列表 select from user_file
4. 获取下载地址和上传地址



# 加密算法的比较

					  MD5				SHA1
	代表      信息摘要    安全哈希算法
	消息长度   128				160
	安全      较差        中等
	速度		  快速        慢

# 上传文件

1. 普通上传

   流程：客户端利用http的multipart/form-data提交文件 ---> 后台利用gin框架的FormFile读取文件 --->  构建文件元信息（文件名，本地存储位置，filehash，上传时间等）---> 创建本地文件保存 --->判读是否保存在阿里云的oss上 ---> 判断是否开启异步处理

2. 分块上传

   1. 客户端请求分块上传的初始化接口（传递文件的大小【前端可以通过file.size获得】和【sha1加密】hash值），服务器根据用户上传的信息计算出分多少个文件块，以及上传文件块的UploadID【用户名+时间戳】，并在redis中利用hash保存该用户的初始化信息（分块的数量，文件hash，文件大小等）
   2. 客户端得到响应后，开始分块上传，同时将每个分块的编号发送给服务器。服务器在本地创建一个零时文件（文件名为每个分块的编号）接收传来的文件数据并将分块编号放入redis中。
   3. 客户端发送完完成后，会请求合并的接口。服务器会从redis中读取出已经接收到文件数量，如果发生缺失，则会通知客户端分块丢失并且告诉客户端分块丢失的序号。

3. 秒传

   在上传时，服务器对文件进行哈希运算选择的是sha1，并检查数据库中该文件是否存在。如果存在则直接将文件信息及路径保存在用户表中，否则就正常上传。

## 采用的什么方式？
文件上传采用的是的http协议，psot提交表单，使用的是multipart/form-data参数
在前端中  <form id="upForm" action="#" method="post" enctype="multipart/form-data"> 然后利用ajax异步请求

<img src="/Users/chenlei/Library/Application Support/typora-user-images/image-20211212114437680.png" alt="image-20211212114437680" style="zoom:67%;" />

server端：利用FormFile从http请求中读取文件内容

c.Request.FormFile("file")返回文件中的第一个文件信息的io.rediser和文件的head信息

<img src="/Users/chenlei/Library/Application Support/typora-user-images/image-20211212124325968.png" alt="image-20211212124325968" style="zoom:67%;" />

<img src="/Users/chenlei/Library/Application Support/typora-user-images/image-20211211111903647.png" alt="image-20211211111903647" style="zoom:67%;" />

请求数据

![image-20211211111956877](/Users/chenlei/Library/Application Support/typora-user-images/image-20211211111956877.png)

name项的值对应form表单中input的属性名。filename为上传文件名

在multipart/form-data包含的每一个上床文件都会有content-type；默认类型为text/plain。该项选根据input框中的内容制定。

## 可以上传文件的协议

FTP，HTTP，socket
1. 其中FTP对于上传大文件有绝对的优势
2. http适合小文件的传输

## http的四种请求参数
1. formdata
http请求中的multipart/form-data，会将表单的数据处理为一条消息，以标签为单元，用分隔符分开。multipart既可以上传键值对，也可以上传文件。当上传的字段是文件时，会有content-Type来说明文件类型。content-Disposition：用来描述信息。同时需要有一个boundary来说明边界
1. x-www-form-urlencoded，会将表单内的数据转换为键值对。不可以上传文件。
2. raw：可以上传任意格式的文本，可以上传text，json，xml，html
3. binary：只可以上传二进制。

## 跨域问题

跨域资源共享CORS，使用额外的HTTP头来告诉浏览器，让运行在一个原服务器的web应用被准许访问来自不同源服务器上指定的资源。

当一个资源从该资源本身所在的服务器不同的域、协议或者端口请求一个资源时，资源会发起一个跨域HTTP请求。

```go

func Cors() gin.HandlerFunc {
	return func(c *gin.Context) {
		method := c.Request.Method

		c.Header("Access-Control-Allow-Origin", "*")
		c.Header("Access-Control-Allow-Headers", "Content-Type,AccessToken,X-CSRF-Token, Authorization, Token")
		c.Header("Access-Control-Allow-Methods", "POST, GET, OPTIONS, PUT, DELETE,UPDATE")      //服务器支持的所有跨域请求的方
		c.Header("Access-Control-Expose-Headers", "Content-Length, Access-Control-Allow-Origin, Access-Control-Allow-Headers, Content-Type")
		c.Header("Access-Control-Allow-Credentials", "true")

		//放行所有OPTIONS方法
		if method == "OPTIONS" {
			c.AbortWithStatus(http.StatusNoContent)
		}
		// 处理请求
		c.Next()
	}
}
```



# 文件下载

1. 普通下载

发送http中的content-type为application/octet-stream以及文件下载的url给前端，前端通过ajax提交下载；hidden的iframe提交

通过post请求

![image-20211212114032959](/Users/chenlei/Library/Application Support/typora-user-images/image-20211212114032959.png)

![image-20211212112428631](/Users/chenlei/Library/Application Support/typora-user-images/image-20211212112428631.png)

1. 多goroutine下载
2. 断点下载 

# 异步传输

RabbitMQ的routing

在上传时如果开启异步传输则利用RabbitMQ进行

1.发送给rabbitmq的消息包括文件hash，本地存储路径，目的存储路径（oss的路径）和存储类型。

2.并将数据转为json格式

3.发布消息

  rabbitMq所做的确保是：只要把消息投递到broker中，那么就确保这个消息会送达到消费者手里。

  a. 消息成功的发出去了

  b. 保证broker成功额度收到了消息

  c. 生产者收到了broker的确认应答

  d. 消息补偿机制，当前三步都失败了，做一个补偿重发机制

  e. 如果第四步失败次数过多，那么说明系统出现问题，要进行人为干预

消息发送的确认机制可以采用confirm和事务。事务是同步的，而confirm是异步的，在没有收到回应时，会一直发送消息。



## 消息发布

![image-20211212134738607](/Users/chenlei/Library/Application Support/typora-user-images/image-20211212134738607.png)



<img src="/Users/chenlei/Library/Application Support/typora-user-images/image-20211212134807375.png" style="zoom:50%;" />

```go
var notifyClose chan *amqp.Error
var notifyConfirm chan amqp.Confirmation

// 注册确认机制
err = channel.Confirm(false)
	if err != nil {
		log.Println("this.channel.confirm", err)
		return false
	}
// 注册一个监听发布的channel，当消息发布成功为ack， 返回一个deliver
notifyConfirm = channel.NotifyPublish(make(chan amqp.Confirmation))
```



```go

// 三次重传的发消息
func (producer *Producer) Push(data []byte) error {
	if !producer.isConnected {
		return errors.New("failed to push push: not connected")
	}
	var currentTime = 0
	for {
		err := producer.UnsafePush(data)
    // 判断发送消息失败
		if err != nil {
			producer.logger.Println("Push failed. Retrying...")
			currentTime += 1
			if currentTime < resendTime {
				continue
			}else {
				return err
			}
		}
    // 如果发送消息没有失败，检测是否能到达broker
		ticker := time.NewTicker(resendDelay)
		select {
		case confirm := <-producer.notifyConfirm:
			if confirm.Ack {
				producer.logger.Println("Push confirmed!")
				return nil
			}
		case <- ticker.C:
		}
		producer.logger.Println("Push didn't confirm. Retrying...")
	}
}

// 发送出去，不管是否接受的到
func (producer *Producer) UnsafePush(data []byte) error {
	if !producer.isConnected {
		return errNotConnected
	}
	return producer.channel.Publish(
		"",         // Exchange
		producer.name, // Routing key
		false,      // Mandatory
		false,      // Immediate
		amqp.Publishing{
			DeliveryMode: 2,
			ContentType:  "application/json",
			Body:        data,
			Timestamp:  time.Now(),
		},
	)
}
```



**Q&A**

消息的发布：与rabbitmq服务器建立一个channel，然后将文件的一些信息，例如文件hash，当前保存地址和目的地址发送到rabbitmq中。

Q：如何确保消息能够发送成功和不丢失

先判断消息是否发送成功，在判断是否发送到broker

rabbitmq提供了事务发布和确认机制。事务机制是一个同步的过程，而确认机制是一个异步的过程。在接收到一个失败时，能够一直发送消息。我们选择了rabbitmq的confirm机制。

发送成功了的话，开启了channel.confirm(false)机制，来进行手动确认ack。同时注册一个channel.NotifyPublish(make(chan amqp.Confirmation, 1))来监听信道。这个函数会返回一个confirmation的通道，包含一个devilderTag和ack, 我们可以判断ack是否为true；如果是调用channel.publish发布消息失败，我们设置了三次重发机制。由于网络问题，可能ack发送延迟了，所以我们利用for-select来监听返回的ack和一个ticker定时器。如果一个设定的周期没有接收到ack则重新发送消息。

## 消息消费

<img src="/Users/chenlei/Library/Application Support/typora-user-images/image-20211212134919468.png" alt="image-20211212134919468" style="zoom:67%;" />

 

利用channel.consume( )来获得消息，consume返回一个通道，开启一个goroutine从通道中获取消息，进行消费。

如何确保消息消费：

我们将consume的autoack设置为false，因为autoack是自动回复ack，消费成功我们需要手动ack，调用msg.Ack(false)。当broker收到消息后会将消息溢出。如果消费失败，我们需要用msg.Nack或者Reject来告诉broker消费失败，并且可以决定是否重新加入队列尾部。

消息重复消费：

因为发布到broker的消息会有一个全局消息ID，我们可以利用这个ID搭配redis缓存，来判断消息是否消费了，如果消费了则回复broker一个reject(false)将该消息移除队列。因为redis的key有天然的幂等性。

如何保证rabbitmq顺序消费

因为rabbitmq投递消息时是有序的，所以我们可以每一个queue对应一个consume。

# gin

<img src="/Users/chenlei/Downloads/filestore-server/doc/about_gin.png" alt="about_gin" style="zoom:80%;" />

# go-micro

首先用micro new一个微服务，然后编写protobuf，然后利用protoc生成micro和go的文件。然后就实现micro中的服务端的接口。最后就是注册微服务。



微服务的划分按照功能模块划分为了用户模块登录注册，用户信息查询等；上传文件模块；下载文件模块；和异步转移模块

好处：可以解耦服务，便于单个服务的重新部署上线，提高系统的容灾性

坏处：出现错误，排查成本上升了。


# consul