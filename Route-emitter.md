# route-emitter 部分(初版)
下面开始从route-emitter部分开始简要源码分析，如果后期发现问题，择后在改 <br />

route-emitter一共有1个组件组成
-----------------------------------
说明：<br />
用来接收nats端的路由注册和反注册信息，然后通过这些信息注册或更新路由表<br />

### Route-emitter组件分析
先看它的启动参数：<br />

		/var/vcap/packages/route_emitter/bin/route-emitter \
		-consulCluster=http://127.0.0.1:8500 \
		-natsAddresses=10.244.0.6:4222 \
		-natsUsername=nats \
		-natsPassword=nats \
		-diegoAPIURL=http://:@receptor.service.cf.internal:8887 \
		-debugAddr=0.0.0.0:17009 \
		-syncInterval=60s \
		-logLevel=debug
		
直接看核心：watcher/watcher.go<br />

这里watcher的对象主要有两个，一个是<strong>DesiredLrp</strong>,一个是<strong>ActualLrp</strong><br />

//事件类型 和tps里定义的类似，不过这些事件都是从nats获取，事件源不一样<br />

		func (watcher *Watcher) handleEvent(logger lager.Logger, event receptor.Event) {
			switch event := event.(type) {
			//应用清单建立事件
			case receptor.DesiredLRPCreatedEvent:
				watcher.handleDesiredCreate(logger, event.DesiredLRPResponse)
			//应用清单变化事件
			case receptor.DesiredLRPChangedEvent:
				watcher.handleDesiredUpdate(logger, event.Before, event.After)
			//应用清单移除事件，其实就是整个应用被删除
			case receptor.DesiredLRPRemovedEvent:
				watcher.handleDesiredDelete(logger, event.DesiredLRPResponse)
			//应用运行实例创建事件
			case receptor.ActualLRPCreatedEvent:
				watcher.handleActualCreate(logger, event.ActualLRPResponse)
			//应用运行实例变化事件
			case receptor.ActualLRPChangedEvent:
				watcher.handleActualUpdate(logger, event.Before, event.After)
			//应用运行实例被移除事件
			case receptor.ActualLRPRemovedEvent:
				watcher.handleActualDelete(logger, event.ActualLRPResponse)
			default:
				logger.Info("did-not-handle-unrecognizable-event", lager.Data{"event-type": event.EventType()})
			}
		}

--->定位到func (watcher *Watcher) Run(signals <-chan os.Signal, ready chan<- struct{}) error <br />

首先定义了两个通道：<br />

		eventChan := make(chan receptor.Event)
		syncEndChan := make(chan syncEndEvent)
		
		var eventSource atomic.Value
		var stopEventSource int32
	
后面来到核心代码块：<br />

		//同步事件
		for {
			select {
			case <-watcher.syncEvents.Sync:
				if syncing == false {
					logger := watcher.logger.Session("sync")
					logger.Info("starting")
					syncing = true

					if !startedEventSource {
						startedEventSource = true
						startEventSource() 
					}

					cachedEvents = make(map[string]receptor.Event)
					go watcher.sync(logger, syncEndChan)
				}

			case syncEnd := <-syncEndChan:
				watcher.completeSync(syncEnd, cachedEvents)
				cachedEvents = nil
				syncing = false
				syncEnd.logger.Info("complete")

			case <-watcher.syncEvents.Emit:
				logger := watcher.logger.Session("emit")
				watcher.emit(logger)

			case event := <-eventChan:
				if syncing {
					watcher.logger.Info("caching-event", lager.Data{
						"type": event.EventType(),
					})

					cachedEvents[event.Key()] = event
				} else {
					watcher.handleEvent(watcher.logger, event)
				}

			case <-signals:
				watcher.logger.Info("stopping")
				atomic.StoreInt32(&stopEventSource, 1)
				if es := eventSource.Load(); es != nil {
					err := es.(receptor.EventSource).Close()
					if err != nil {
						watcher.logger.Error("failed-closing-event-source", err)
					}
				}
				return nil
			}
		}
	
--->跟到startedEventSource = true startEventSource()，这里是个匿名函数<br />
startEventSource := func(){}跟进去看看里面的具体信息：<br />
	
		接着执行了一个goroutine,里面有
		不断的从受体接收信息，并将其存储起来：
		es, err = watcher.receptorClient.SubscribeToEvents()
		eventSource.Store(es)
		
		在采用一层循环，将event放入eventChan通道里
		if event != nil {
			eventChan <- event
		}
		
		执行完startEventSource()后，将受体里的事件都存储起来：
		cachedEvents = make(map[string]receptor.Event)
		最后执行sync同步方法
		go watcher.sync(logger, syncEndChan)
		进入到sync方法中去：
		var runningActualLRPs []receptor.ActualLRPResponse
		var getActualLRPsErr error
		var desiredLRPs []receptor.DesiredLRPResponse
		var getDesiredLRPsErr error
		从这里可以看到定义了desiredLRPs和ActualLRPs而且是running的
		进入runnintActualLRPs
		go func() {
			defer wg.Done()

			logger.Debug("getting-actual-lrps")
			actualLRPResponses, err := watcher.receptorClient.ActualLRPs()
			if err != nil {
				logger.Error("failed-getting-actual-lrps", err)
				getActualLRPsErr = err
				return
			}
			logger.Debug("succeeded-getting-actual-lrps", lager.Data{"num-actual-responses": len(actualLRPResponses)})

			runningActualLRPs = make([]receptor.ActualLRPResponse, 0, len(actualLRPResponses))
			for _, actualLRPResponse := range actualLRPResponses {
				if actualLRPResponse.State == receptor.ActualLRPStateRunning {
					runningActualLRPs = append(runningActualLRPs, actualLRPResponse)
				}
			}
		}()
		
很明显的看到它是通过执行一个匿名goroutine函数，通过receptorClient.ActualLRPs()获取actualLRPs，继而判断其state是否是running，最后将running状态的actualLRP加入到runningActualLRPs中<br />
	
--->再继续看desiredLRPs<br />

		go func() {
			defer wg.Done()

			logger.Debug("getting-desired-lrps")
			desiredLRPResponses, err := watcher.receptorClient.DesiredLRPs()
			if err != nil {
				logger.Error("failed-getting-desired-lrps", err)
				getDesiredLRPsErr = err
				return
			}
			logger.Debug("succeeded-getting-desired-lrps", lager.Data{"num-desired-responses": len(desiredLRPResponses)})

			desiredLRPs = make([]receptor.DesiredLRPResponse, 0, len(desiredLRPResponses))
			for _, desiredLRPResponse := range desiredLRPResponses {
				desiredLRPs = append(desiredLRPs, desiredLRPResponse)
			}
		}()
		
这里也很明显通过receptorClient.DesiredLRPs获取所有的desiredLRPs，然后遍历放入到desiredLRPs中<br />
	
最后生成一份newTable<br />

		newTable := routing_table.NewTempTable(
			routing_table.RoutesByRoutingKeyFromDesireds(desiredLRPs),
			routing_table.EndpointsByRoutingKeyFromActuals(runningActualLRPs),
		)
	
我们来看一下desiredLRPs 的handler的日志，这里主要展现了一个app实例被扩展的情形 private_key太长被我砍掉了<br />
		{
		"timestamp": "1441088698.029039621", 
		"source": "route-emitter", 
		"message": "route-emitter.watcher.handling-desired-update.starting", 
		"log_level": 1, 
		"data": {
		"after": {
		  "ports": [
			8080, 
			2222
		  ], 
		  "process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
		  "routes": {
			"cf-router": [
			  {
				"hostnames": [
				  "helloDocker.10.244.0.34.xip.io"
				], 
				"port": 8080
			  }
			], 
			"diego-ssh": {
			  "container_port": 2222, 
			  "host_fingerprint": "b8:d0:bf:b8:1f:06:d0:1a:d2:7c:ea:91:b5:70:43:fc", 
			  "private_key": ""
			}
		  }
		}, 
		"before": {
		  "ports": [
			8080, 
			2222
		  ], 
		  "process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
		  "routes": {
			"cf-router": [
			  {
				"hostnames": [
				  "helloDocker.10.244.0.34.xip.io"
				], 
				"port": 8080
			  }
			], 
			"diego-ssh": {
			  "container_port": 2222, 
			  "host_fingerprint": "b8:d0:bf:b8:1f:06:d0:1a:d2:7c:ea:91:b5:70:43:fc", 
			  "private_key": ""
			}
		  }
		}, 
		"session": "5.2695"
		}
		}

这里可以看到有两张路由表的前后对比，如果只是修改实例个数，一般不会有什么变化<br />
下面一般还有 不过里面内容都差不多<br />

	   route-emitter.watcher.handling-desired-update.emitting-messages 
	   route-emitter.watcher.handling-desired-update.complet
   
等一会，清单部署到cells里还需要一点事件来触发实例也就是actualLRPs,注意观察before和after<br />
   
		{
		"timestamp": "1441088706.596357346", 
		"source": "route-emitter", 
		"message": "route-emitter.watcher.handling-actual-update.starting", 
		"log_level": 1, 
		"data": {
		"after": {
		  "address": "10.244.16.138", 
		  "cell-id": "cell_z1-0", 
		  "domain": "cf-apps", 
		  "evacuating": false, 
		  "index": 1, 
		  "instance-guid": "21811b3c-a37e-4277-5878-c646a7213bf0", 
		  "ports": [
			{
			  "container_port": 8080, 
			  "host_port": 60002
			}, 
			{
			  "container_port": 2222, 
			  "host_port": 60003
			}
		  ], 
		  "process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6"
		}, 
		"before": {
		  "address": "", 
		  "cell-id": "cell_z1-0", 
		  "domain": "cf-apps", 
		  "evacuating": false, 
		  "index": 1, 
		  "instance-guid": "21811b3c-a37e-4277-5878-c646a7213bf0", 
		  "ports": null, 
		  "process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6"
		}, 
		"session": "5.2699"
		}
		}
   
这时我们就能看到实例在容器中是怎么映射的了，这也就是diego在这两个LRPs的精妙处理<br />

		{
		"timestamp": "1441088706.596627951", 
		"source": "route-emitter", 
		"message": "route-emitter.watcher.handling-actual-update.emitting-messages", 
		"log_level": 0, 
		"data": {
		"after": {
		  "address": "10.244.16.138", 
		  "cell-id": "cell_z1-0", 
		  "domain": "cf-apps", 
		  "evacuating": false, 
		  "index": 1, 
		  "instance-guid": "21811b3c-a37e-4277-5878-c646a7213bf0", 
		  "ports": [
			{
			  "container_port": 8080, 
			  "host_port": 60002
			}, 
			{
			  "container_port": 2222, 
			  "host_port": 60003
			}
		  ], 
		  "process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6"
		}, 
		"before": {
		  "address": "", 
		  "cell-id": "cell_z1-0", 
		  "domain": "cf-apps", 
		  "evacuating": false, 
		  "index": 1, 
		  "instance-guid": "21811b3c-a37e-4277-5878-c646a7213bf0", 
		  "ports": null, 
		  "process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6"
		}, 
		"messages": {
		  "RegistrationMessages": [
			{
			  "host": "10.244.16.138", 
			  "port": 60002, 
			  "uris": [
				"helloDocker.10.244.0.34.xip.io"
			  ], 
			  "app": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
			  "private_instance_id": "21811b3c-a37e-4277-5878-c646a7213bf0"
			}
		  ], 
		  "UnregistrationMessages": null
		}, 
		"session": "5.2699"
		}
		}
		
看一下nats的注册信息：<br />

		{
		"timestamp": "1441088706.596804857", 
		"source": "route-emitter", 
		"message": "route-emitter.nats-emitter.emit", 
		"log_level": 0, 
		"data": {
		"message": {
		  "host": "10.244.16.138", 
		  "port": 60002, 
		  "uris": [
			"helloDocker.10.244.0.34.xip.io"
		  ], 
		  "app": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
		  "private_instance_id": "21811b3c-a37e-4277-5878-c646a7213bf0"
		}, 
		"session": "3", 
		"subject": "router.register"
		}
		}

直到看到route-emitter.watcher.handling-actual-update.complete 这个实例被完整的创建成功<br />
	
--->接下来进如到watcher部分，更新路由表<br />
		{
		"timestamp": "1441088716.608674765", 
		"source": "route-emitter", 
		"message": "route-emitter.watcher.emit.emitting-messages", 
		"log_level": 0, 
		"data": {
		"messages": {
		  "RegistrationMessages": [
			{
			  "host": "10.244.16.138", 
			  "port": 60000, 
			  "uris": [
				"helloDocker.10.244.0.34.xip.io"
			  ], 
			  "app": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
			  "private_instance_id": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe"
			}, 
			{
			  "host": "10.244.16.138", 
			  "port": 60002, 
			  "uris": [
				"helloDocker.10.244.0.34.xip.io"
			  ], 
			  "app": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
			  "private_instance_id": "21811b3c-a37e-4277-5878-c646a7213bf0"
			}
		  ], 
		  "UnregistrationMessages": null
		}, 
		"session": "5.2701"
		}
		}

		{"timestamp":"1441088716.608885765","source":"route-emitter","message":"route-emitter.nats-emitter.emit","log_level":0,"data":{"message":{"host":"10.244.16.138","port":60000,"uris":["helloDocker.10.244.0.34.xip.io"],"app":"c181be6b-d89c-469c-b79c-4cab1e99b8de","private_instance_id":"fc4a83cd-5ed9-40fb-51bf-1679beb41bfe"},"session":"3","subject":"router.register"}}
		{"timestamp":"1441088716.609278917","source":"route-emitter","message":"route-emitter.nats-emitter.emit","log_level":0,"data":{"message":{"host":"10.244.16.138","port":60002,"uris":["helloDocker.10.244.0.34.xip.io"],"app":"c181be6b-d89c-469c-b79c-4cab1e99b8de","private_instance_id":"21811b3c-a37e-4277-5878-c646a7213bf0"},"session":"3","subject":"router.register"}}

再次看到watcher 已经发现actuallrps已经变成2了<br />

		{
			"timestamp": "1441088716.663050652", 
			"source": "route-emitter", 
			"message": "route-emitter.watcher.sync.succeeded-getting-actual-lrps", 
			"log_level": 0, 
			"data": {
				"num-actual-responses": 2, 
				"session": "5.2700"
			}
		}
	
而desiredlrps还是1，说明都是正常的<br />

		{
			"timestamp": "1441088716.664330006", 
			"source": "route-emitter", 
			"message": "route-emitter.watcher.sync.succeeded-getting-desired-lrps", 
			"log_level": 0, 
			"data": {
				"num-desired-responses": 1, 
				"session": "5.2700"
			}
		}
	
如果想了解touteTable请看到routing_table这一部分的代码