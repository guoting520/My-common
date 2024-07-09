## Common公共库的内容

* errInfo —— 错误信息，通过结构体获取错误码和错误内容
* gozero —— gozero模板引用的函数改写
* mqtt —— mqtt接口
* config.go —— 公共的配置或者const 定义
* tools.go —— 工具函数
* types.go —— 公共结构体

## kafka使用
* kafka.go - 封装kafka接收消息  
1. 需要在rpc/etc/xxx.yaml中增加  
```text
Kafka:
  Brokers:
    - 10.51.37.211:9092

```
2. 在rpc/internal/config/config.go中增加  
```text
type Config struct {
	Kafka struct {
		Brokers []string
	}
}
```
3. 增加rpc/internal/logic/kafka.go文件  
```text
var (
  group = "physicalModel" // 消费的分组名
  topics = []string{"v1.physicalModel", "v1.attributeUpload"} // 消费的主题
)

func KqConsumer(c config.Config, svc *svc.ServiceContext) error {
	handler := &respKqHandler{
		c:   c,
		svc: svc,
	}
	err := svc.Kafka.KqConsumer(group, topics, handler)
	if err != nil {
		return err
	}

	go func() {
		err := svc.Kafka.Wait()
		if err != nil {
			logx.Error(err)
		}
	}()
	return nil
}

type respKqHandler struct {
	c   config.Config
	svc *svc.ServiceContext
}

func (*respKqHandler) Setup(_ sarama.ConsumerGroupSession) error {
	return nil
}
func (*respKqHandler) Cleanup(_ sarama.ConsumerGroupSession) error {
	return nil
}
func (h *respKqHandler) ConsumeClaim(s sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
	for msg := range claim.Messages() {
		logx.Info(msg.Topic, string(msg.Key))
		// 处理收到的消息
		var err error
		switch msg.Topic {
		case topics[0]: // 消费v1.physicalModel主题的消息
			err = h.consumerPhysicalModel(msg)
		case topics[1]: // 消费v1.attributeUpload主题的消息
			err = h.consumerAttributeUpload(msg)
		}
		if err != nil {
			logx.Error(err)
		}

		s.MarkMessage(msg, "")
	}
	return nil
}
```
4. 在rpc/xxx.go的主程序中增加  
```text
    defer s.Stop()
    // 调用kafka消费逻辑
    err := logic.KqConsumer(c, ctx)
    if err != nil {
        panic(err)
    }
```

## 通过eureka调用中台服务
