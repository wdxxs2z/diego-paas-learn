# Cells 部分(初版)
下面开始从Cells部分开始简要源码分析，如果后期发现问题，择后在改 <br />

Cells不能从严格意义上说有几个组件，比较关键的有4个，加上扩展的5个
-----------------------------------
* rep bbs，auctions组件都要于它通信，同时优势executor的桥梁，进入garden的大门
* garden 容器前端
* executor 执行体
* garden-linux-backend 容器后端
* garden-windows(扩展)

### rep executor组件分析
先看它的启动参数：

		/var/vcap/packages/rep/bin/rep ${etcd_sec_flags} \
		-etcdCluster="https://etcd.service.cf.internal:4001" \
		-bbsAddress=http://bbs.service.cf.internal:8889 \
		-consulCluster=http://127.0.0.1:8500 \
		-receptorTaskHandlerURL=http://receptor.service.cf.internal:1169 \
		-debugAddr=0.0.0.0:17008 \
		-listenAddr=0.0.0.0:1800 \
		-preloadedRootFS cflinuxfs2:/var/vcap/packages/rootfs_cflinuxfs2/rootfs \
		-rootFSProvider docker \
		-cellID=cell_z1-0 \
		-zone=z1 \
		-pollingInterval=30s \
		-evacuationPollingInterval=10s \
		-evacuationTimeout=60s \
		-skipCertVerify=true \
		-gardenNetwork=tcp \
		-gardenAddr=127.0.0.1:7777 \
		-memoryMB=auto \
		-diskMB=auto \
		-containerInodeLimit=200000 \
		-containerMaxCpuShares=1024 \
		-cachePath=$CACHE_DIR \
		-maxCacheSizeInBytes=10000000000\
		-exportNetworkEnvVars=true\
		-healthyMonitoringInterval=30s \
		-unhealthyMonitoringInterval=0.5s \
		-createWorkPoolSize=32 \
		-deleteWorkPoolSize=32 \
		-readWorkPoolSize=64 \
		-metricsWorkPoolSize=8 \
		-healthCheckWorkPoolSize=64 \
		-tempDir=$TMP_DIR \
		-logLevel=debug
		
* rep：
做为进入cells的大门，rep组件起了很关键的作用：</br>
1.首先它代表cell调解所有来自BBS组件的通信，主要是确保BBS组件中Tasks和actuallLrps在容器中的同步，然后是容灾，这个在Converger组件中已经分析到了，主要是容灾实例迁移</br>
2.参与auctions的Tasks和LRPs的请求</br>
3.内部会询问Executor组件，通过这个组件，tasks和lrps才能够真正的创建容器并在容器中运行任务或进程</br>

* Executor：
它本身是一个容器创建和执行的实现，它不会区分Tasks和LRPs，这个组件会将cell中的日志转出来</br>

重点先看一下rep:这个组件有一大堆参数 也是后期需要根据实际应用环境进行调优最多地方的组件，这个是和容器比较相关的配置</br>
https://github.com/cloudfoundry-incubator/rep/blob/6676dde8901ba3c483602e48ff8f88058f285f76/cmd/rep/executor.go</br>

		func executorConfig() executorinit.Configuration {
			return executorinit.Configuration{
				GardenNetwork:               *gardenNetwork, //tcp
				GardenAddr:                  *gardenAddr,//garden端的地址 127.0.0.1:7777
				ContainerOwnerName:          *containerOwnerName,//容器的拥有者
				TempDir:                     *tempDir,
				CachePath:                   *cachePath,
				MaxCacheSizeInBytes:         *maxCacheSizeInBytes,//10000000000
				SkipCertVerify:              *skipCertVerify, //true
				ExportNetworkEnvVars:        *exportNetworkEnvVars, //是否需要网络环境变量 true
				ContainerMaxCpuShares:       *containerMaxCpuShares, //1024
				ContainerInodeLimit:         *containerInodeLimit, //inode的数目200000
				HealthyMonitoringInterval:   *healthyMonitoringInterval, //健康检查30s
				UnhealthyMonitoringInterval: *unhealthyMonitoringInterval,
				HealthCheckWorkPoolSize:     *healthCheckWorkPoolSize,
				CreateWorkPoolSize:          *createWorkPoolSize,
				DeleteWorkPoolSize:          *deleteWorkPoolSize,
				ReadWorkPoolSize:            *readWorkPoolSize,
				MetricsWorkPoolSize:         *metricsWorkPoolSize,
				RegistryPruningInterval:     *registryPruningInterval,
				MemoryMB:                    *memoryMBFlag, //剩下两个参数都是auto
				DiskMB:                      *diskMBFlag,
			}
		}

下面的这个是rep组件本身的一些配置
https://github.com/cloudfoundry-incubator/rep/blob/6676dde8901ba3c483602e48ff8f88058f285f76/cmd/rep/main.go</br>

在没进入容器代码之前，我们之前分析过，每个tasks和LRPs都有相应的auction选择cells来执行</br>
进入到auction_cell_rep部分：</br>

		type AuctionCellRep struct {
			cellID               string
			stackPathMap         rep.StackPathMap
			rootFSProviders      auctiontypes.RootFSProviders
			stack                string
			zone                 string
			generateInstanceGuid func() (string, error)
			bbs                  Bbs.RepBBS
			client               executor.Client
			evacuationReporter   evacuation_context.EvacuationReporter
			logger               lager.Logger
		}
		
里面有cellID，stack，rootFSproviders，zone还有一些需要用的客户端比如bbs，executor.Client</br>

		func rootFSProviders(preloaded rep.StackPathMap, arbitrary []string) auctiontypes.RootFSProviders {
			rootFSProviders := auctiontypes.RootFSProviders{}
			for _, scheme := range arbitrary {
				rootFSProviders[scheme] = auctiontypes.ArbitraryRootFSProvider{}
			}

			stacks := make([]string, 0, len(preloaded))
			for stack, _ := range preloaded {
				stacks = append(stacks, stack)
			}
			rootFSProviders["preloaded"] = auctiontypes.NewFixedSetRootFSProvider(stacks...)

			return rootFSProviders
		}

这一块其实就是已经配置好的：</br>

		-preloadedRootFS cflinuxfs2:/var/vcap/packages/rootfs_cflinuxfs2/rootfs
		-rootFSProvider docker

执行一下cf stacks：</br>

		cflinuxfs2      Cloud Foundry Linux-based filesystem   
		windows2012R2   Windows Server 2012 R2
		
这些都是preloadedRootFS就是预先在部署的时候预装的rootfs</br>

--->然后进入func (a *AuctionCellRep) Perform(work auctiontypes.Work) (auctiontypes.Work, error) 方法</br>
* 先看LRPs部分：</br>

		if len(work.LRPs) > 0 {
			lrpLogger := logger.Session("lrp-allocate-instances")

			containers, lrpAuctionMap, untranslatedLRPs := a.lrpsToContainers(work.LRPs)
			
			//lrpsToContainers(work.LRPs)方法可以获取容器配置，auction集合，未定义的LRPs
			
			if len(untranslatedLRPs) > 0 {
				lrpLogger.Info("failed-to-translate-lrps-to-containers", lager.Data{"num-failed-to-translate": len(untranslatedLRPs)})
				failedWork.LRPs = untranslatedLRPs
			}

			lrpLogger.Info("requesting-container-allocation", lager.Data{"num-requesting-allocation": len(containers)})
			
			//这里开始分配容器
			errMessageMap, err := a.client.AllocateContainers(containers)
			
			if err != nil {
				lrpLogger.Error("failed-requesting-container-allocation", err)
				failedWork.LRPs = work.LRPs
			} else {
				lrpLogger.Info("succeeded-requesting-container-allocation", lager.Data{"num-failed-to-allocate": len(errMessageMap)})
				for guid, lrpStart := range lrpAuctionMap {
					if _, found := errMessageMap[guid]; found {
						failedWork.LRPs = append(failedWork.LRPs, lrpStart)
					}
				}
			}
		}

看到这个函数：rpsToContainers</br>
首先它会遍历所有的LRPs,然后根据lrp的信息构造出每一个容器的资源等信息</br>

		containerGuid := rep.LRPContainerGuid(lrpStart.DesiredLRP.ProcessGuid, instanceGuid)
		lrpAuctionMap[containerGuid] = lrpStart

具体看一下这个container是如何构造的：</br>

		container := executor.Container{
			Guid: containerGuid,

			Tags: executor.Tags{
				rep.LifecycleTag:    rep.LRPLifecycle,
				rep.DomainTag:       lrpStart.DesiredLRP.Domain,
				rep.ProcessGuidTag:  lrpStart.DesiredLRP.ProcessGuid,
				rep.InstanceGuidTag: instanceGuid,
				rep.ProcessIndexTag: strconv.Itoa(lrpStart.Index),
			},

			MemoryMB:     int(lrpStart.DesiredLRP.MemoryMb),
			DiskMB:       int(lrpStart.DesiredLRP.DiskMb),
			DiskScope:    diskScope,
			CPUWeight:    uint(lrpStart.DesiredLRP.CpuWeight),
			RootFSPath:   rootFSPath,
			Privileged:   lrpStart.DesiredLRP.Privileged,
			Ports:        a.convertPortMappings(lrpStart.DesiredLRP.Ports),
			StartTimeout: uint(lrpStart.DesiredLRP.StartTimeout),

			LogConfig: executor.LogConfig{
				Guid:       lrpStart.DesiredLRP.LogGuid,
				Index:      lrpStart.Index,
				SourceName: lrpStart.DesiredLRP.LogSource,
			},

			MetricsConfig: executor.MetricsConfig{
				Guid:  lrpStart.DesiredLRP.MetricsGuid,
				Index: lrpStart.Index,
			},

			Setup:   lrpStart.DesiredLRP.Setup,
			Action:  lrpStart.DesiredLRP.Action,
			Monitor: lrpStart.DesiredLRP.Monitor,

			Env: append([]executor.EnvironmentVariable{
				{Name: "INSTANCE_GUID", Value: instanceGuid},
				{Name: "INSTANCE_INDEX", Value: strconv.Itoa(lrpStart.Index)},
				{Name: "CF_INSTANCE_GUID", Value: instanceGuid},
				{Name: "CF_INSTANCE_INDEX", Value: strconv.Itoa(lrpStart.Index)},
			}, executor.EnvironmentVariablesFromModel(lrpStart.DesiredLRP.EnvironmentVariables)...),
			EgressRules: lrpStart.DesiredLRP.EgressRules,
		}

其实就是DesiredLRPs定义的一些清单,因为执行器和rep是密切相关的，所以我们可以看看它是怎么分配这个容器的：</br>
就是上面这个函数</br>

		errMessageMap, err := a.client.AllocateContainers(containers)

* 直接来到https://github.com/cloudfoundry-incubator/executor/blob/6c38b2fe1a175d8074e0e0164bc9841fd023161c/depot/depot.go#L110</br>

		//分配容器前的预备工作
		for _, executorContainer := range executorContainers {
			if executorContainer.CPUWeight > 100 || executorContainer.CPUWeight < 0 {
				logger.Debug("invalid-cpu-weight", lager.Data{
					"guid":      executorContainer.Guid,
					"cpuweight": executorContainer.CPUWeight,
				})
				errMessageMap[executorContainer.Guid] = executor.ErrLimitsInvalid.Error()
				continue
			} else if executorContainer.CPUWeight == 0 {
				//如果是0，则给最高权重100
				executorContainer.CPUWeight = 100
			}

			if executorContainer.Guid == "" {
				logger.Debug("empty-guid")
				errMessageMap[executorContainer.Guid] = executor.ErrGuidNotSpecified.Error()
				continue
			}

			eligibleContainers = append(eligibleContainers, executorContainer)
		}
		
这一段是对container容器定义的cpu优先级权值进行判断，很好理解</br>

然后看到，它先加了一个资源锁：</br>

		c.resourcesLock.Lock()

然后对可分配的容器分配容器资源：</br>

		for _, allocatableContainer := range allocatableContainers {
			if _, err := c.allocationStore.Allocate(logger, allocatableContainer); err != nil {
				logger.Debug(
					"failed-to-allocate-container",
					lager.Data{
						"guid":  allocatableContainer.Guid,
						"error": err.Error(),
					},
				)
				errMessageMap[allocatableContainer.Guid] = err.Error()
			}
		}
		
我们可以继续跟到这个函数：c.allocationStore.Allocate(logger, allocatableContainer)</br>
https://github.com/cloudfoundry-incubator/executor/blob/6c38b2fe1a175d8074e0e0164bc9841fd023161c/depot/allocationstore/allocationstore.go#L49</br>

		func (a *AllocationStore) Allocate(logger lager.Logger, container executor.Container) (executor.Container, error) {
			a.lock.Lock()
			defer a.lock.Unlock()

			if _, err := a.lookup(container.Guid); err == nil {
				logger.Error("failed-allocating-container", err)
				return executor.Container{}, executor.ErrContainerGuidNotAvailable
			}
			logger.Debug("allocating-container", lager.Data{"container": container})

			container.State = executor.StateReserved
			container.AllocatedAt = a.clock.Now().UnixNano()
			a.allocated[container.Guid] = container

			a.eventEmitter.Emit(executor.NewContainerReservedEvent(container))

			return container, nil
		}

容器分配完，我们来看一份创建清单：</br>

		{
		  "timestamp": "1441087134.999691010", 
		  "source": "rep", 
		  "message": "rep.depot-client.allocate-containers.allocating-container", 
		  "log_level": 0, 
		  "data": {
			"container": {
			  "guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-b4dcdbe9337046cc91410d126d4cf861", 
			  "state": "", 
			  "privileged": true, 
			  "memory_mb": 1024, 
			  "disk_mb": 6144, 
			  "cpu_weight": 100, 
			  "tags": {
				"domain": "cf-app-staging", 
				"lifecycle": "task", 
				"result-file": "/tmp/docker-result/result.json"
			  }, 
			  "allocated_at": 0, 
			  "rootfs": "/var/vcap/packages/rootfs_cflinuxfs2/rootfs", 
			  "external_ip": "", 
			  "ports": null, 
			  "log_config": {
				"guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
				"index": 0, 
				"source_name": "STG"
			  }, 
			  "metrics_config": {
				"guid": "", 
				"index": 0
			  }, 
			  "start_timeout": 0, 
			  "setup": null, 
			  "run": {
				"timeout": {
				  "action": {
					"serial": {
					  "actions": [
						{
						  "emit_progress": {
							"action": {
							  "download": {
								"from": "http://file-server.service.cf.internal:8080/v1/static/docker_app_lifecycle/docker_app_lifecycle.tgz", 
								"to": "/tmp/docker_app_lifecycle", 
								"cache_key": "docker-lifecycle", 
								"user": "vcap"
							  }
							}, 
							"start_message": "", 
							"success_message": "", 
							"failure_message_prefix": "Failed to set up docker environment"
						  }
						}, 
						{
						  "emit_progress": {
							"action": {
							  "run": {
								"path": "/tmp/docker_app_lifecycle/builder", 
								"args": [
								  "-outputMetadataJSONFilename", 
								  "/tmp/docker-result/result.json", 
								  "-dockerRef", 
								  "tutum/tomcat:8.0"
								], 
								"env": [
								  {
									"name": "VCAP_APPLICATION", 
									"value": "{"limits":{"mem":256,"disk":1024,"fds":16384},"application_id":"c181be6b-d89c-469c-b79c-4cab1e99b8de","application_version":"6eac776b-f051-4cb0-9614-5a3954d1bfd6","application_name":"helloDocker","version":"6eac776b-f051-4cb0-9614-5a3954d1bfd6","name":"helloDocker","space_name":"cloud-space","space_id":"e73d49d7-e590-42fb-a046-50a9f5c259d5"}"
								  }, 
								  {
									"name": "VCAP_SERVICES", 
									"value": "{}"
								  }, 
								  {
									"name": "MEMORY_LIMIT", 
									"value": "256m"
								  }, 
								  {
									"name": "CF_STACK", 
									"value": "cflinuxfs2"
								  }
								], 
								"resource_limits": {
								  "nofile": 16384
								}, 
								"user": "vcap"
							  }
							}, 
							"start_message": "Staging...", 
							"success_message": "Staging Complete", 
							"failure_message_prefix": "Staging Failed"
						  }
						}
					  ]
					}
				  }, 
				  "timeout": 900000000000
				}
			  }, 
			  "monitor": null, 
			  "run_result": {
				"failed": false, 
				"failure_reason": "", 
				"stopped": false
			  }, 
			  "egress_rules": [
				{
				  "protocol": "all", 
				  "destinations": [
					"0.0.0.0-9.255.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"11.0.0.0-169.253.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"169.255.0.0-172.15.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"172.32.0.0-192.167.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"192.169.0.0-255.255.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "tcp", 
				  "destinations": [
					"0.0.0.0/0"
				  ], 
				  "ports": [
					53
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "udp", 
				  "destinations": [
					"0.0.0.0/0"
				  ], 
				  "ports": [
					53
				  ], 
				  "log": false
				}
			  ]
			}, 
			"session": "4.2079"
		  }
		}

这只是一个**task**行为动作，刚好对应上我们之前看到的stager部分在遇到docker只会运行一次task，就是预建立容器，查看docker image是否被缓存等等。</br>

* 然后就直接送到了LRP中去：这时候已经是一个实例咯</br>

		{
		  "timestamp": "1441087199.705446482", 
		  "source": "rep", 
		  "message": "rep.depot-client.allocate-containers.allocating-container", 
		  "log_level": 0, 
		  "data": {
			"container": {
			  "guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6-fc4a83cd-5ed9-40fb-51bf-1679beb41bfe", 
			  "state": "", 
			  "privileged": false, 
			  "memory_mb": 256, 
			  "disk_mb": 1024, 
			  "cpu_weight": 1, 
			  "tags": {
				"domain": "cf-apps", 
				"instance-guid": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe", 
				"lifecycle": "lrp", 
				"process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
				"process-index": "0"
			  }, 
			  "allocated_at": 0, 
			  "rootfs": "docker:///tutum/tomcat#8.0", 
			  "external_ip": "", 
			  "ports": [
				{
				  "container_port": 8080
				}, 
				{
				  "container_port": 2222
				}
			  ], 
			  "log_config": {
				"guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
				"index": 0, 
				"source_name": "CELL"
			  }, 
			  "metrics_config": {
				"guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
				"index": 0
			  }, 
			  "start_timeout": 60, 
			  "setup": {
				"serial": {
				  "actions": [
					{
					  "download": {
						"from": "http://file-server.service.cf.internal:8080/v1/static/docker_app_lifecycle/docker_app_lifecycle.tgz", 
						"to": "/tmp/lifecycle", 
						"cache_key": "docker-lifecycle", 
						"user": "root"
					  }
					}
				  ]
				}
			  }, 
			  "run": {
				"codependent": {
				  "actions": [
					{
					  "run": {
						"path": "/tmp/lifecycle/launcher", 
						"args": [
						  "app", 
						  "", 
						  "{"cmd":["/run.sh"],"ports":[{"Port":8080,"Protocol":"tcp"}]}"
						], 
						"env": [
						  {
							"name": "VCAP_APPLICATION", 
							"value": "{"limits":{"mem":256,"disk":1024,"fds":16384},"application_id":"c181be6b-d89c-469c-b79c-4cab1e99b8de","application_version":"6eac776b-f051-4cb0-9614-5a3954d1bfd6","application_name":"helloDocker","version":"6eac776b-f051-4cb0-9614-5a3954d1bfd6","name":"helloDocker","space_name":"cloud-space","space_id":"e73d49d7-e590-42fb-a046-50a9f5c259d5"}"
						  }, 
						  {
							"name": "VCAP_SERVICES", 
							"value": "{}"
						  }, 
						  {
							"name": "MEMORY_LIMIT", 
							"value": "256m"
						  }, 
						  {
							"name": "CF_STACK", 
							"value": "cflinuxfs2"
						  }, 
						  {
							"name": "PORT", 
							"value": "8080"
						  }
						], 
						"resource_limits": {
						  "nofile": 16384
						}, 
						"user": "root", 
						"log_source": "APP"
					  }
					}, 
					{
					  "run": {
						"path": "/tmp/lifecycle/diego-sshd", 
						"args": [
						  "-address=0.0.0.0:2222", 
						  "-hostKey=", 
						  "-authorizedKey=ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAgQC3zd/eSXS2peF10a0AHX60XDuCtb/kOtOdY/OE9/4IkJexT3pqMbQvf6Te8PxeGJklSOzvI7gP6x2wCWcO2JezKRXBxh2eMAyTvH4CC/gXnBl50vjmG6prbQEujv92+i9nN4K357AQPMY6WHQt4ZuGJcT87U3Iji6zAjGP/U9M5Q==
		", 
						  "-inheritDaemonEnv", 
						  "-logLevel=fatal"
						], 
						"env": [
						  {
							"name": "VCAP_APPLICATION", 
							"value": "{"limits":{"mem":256,"disk":1024,"fds":16384},"application_id":"c181be6b-d89c-469c-b79c-4cab1e99b8de","application_version":"6eac776b-f051-4cb0-9614-5a3954d1bfd6","application_name":"helloDocker","version":"6eac776b-f051-4cb0-9614-5a3954d1bfd6","name":"helloDocker","space_name":"cloud-space","space_id":"e73d49d7-e590-42fb-a046-50a9f5c259d5"}"
						  }, 
						  {
							"name": "VCAP_SERVICES", 
							"value": "{}"
						  }, 
						  {
							"name": "MEMORY_LIMIT", 
							"value": "256m"
						  }, 
						  {
							"name": "CF_STACK", 
							"value": "cflinuxfs2"
						  }, 
						  {
							"name": "PORT", 
							"value": "8080"
						  }
						], 
						"resource_limits": {
						  "nofile": 16384
						}, 
						"user": "root"
					  }
					}
				  ]
				}
			  }, 
			  "monitor": {
				"timeout": {
				  "action": {
					"run": {
					  "path": "/tmp/lifecycle/healthcheck", 
					  "args": [
						"-port=8080"
					  ], 
					  "resource_limits": {
						"nofile": 1024
					  }, 
					  "user": "root", 
					  "log_source": "HEALTH"
					}
				  }, 
				  "timeout": 30000000000
				}
			  }, 
			  "env": [
				{
				  "name": "INSTANCE_GUID", 
				  "value": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe"
				}, 
				{
				  "name": "INSTANCE_INDEX", 
				  "value": "0"
				}, 
				{
				  "name": "CF_INSTANCE_GUID", 
				  "value": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe"
				}, 
				{
				  "name": "CF_INSTANCE_INDEX", 
				  "value": "0"
				}
			  ], 
			  "run_result": {
				"failed": false, 
				"failure_reason": "", 
				"stopped": false
			  }, 
			  "egress_rules": [
				{
				  "protocol": "tcp", 
				  "destinations": [
					"0.0.0.0/0"
				  ], 
				  "ports": [
					53
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "udp", 
				  "destinations": [
					"0.0.0.0/0"
				  ], 
				  "ports": [
					53
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"0.0.0.0-9.255.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"11.0.0.0-169.253.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"169.255.0.0-172.15.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"172.32.0.0-192.167.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"192.169.0.0-255.255.255.255"
				  ], 
				  "log": false
				}
			  ]
			}, 
			"session": "4.2092"
		  }
		}

注意看上面的区别：</br>

> lifecycle：一个是task，下面这个是lrp，
> rootfs: 一个是rootfs_cflinuxfs2/rootfs，一个是docker:///tutum/tomcat#8.0

其它的不看了，我们从刚才打包stager可以看到一些docker的相关文件：</br>

如docker：</br>

> Client version: 1.6.2
> Client API version: 1.18
> Go version (client): go1.4.2
> Git commit (client): 7c8fca2
> OS/Arch (client): linux/amd64

说明是1.6.2的版本</br>
这里延伸一个知识点，就是如果我们有私服怎么办，不过这个也不用操心diego也给我们做好了</br>
如builder，这个太爽了：</br> 此处从stager处也能看到，只不过在这里更明显</br>

		-cacheDockerImage=false: Caches Docker images to private docker registry
		-dockerDaemonExecutablePath="/tmp/docker_app_lifecycle/docker": path to the 'docker' executable
		-dockerEmail="": Email for pulling from docker registry
		-dockerImageURL="": docker image uri in docker://[registry/][scope/]repository[#tag] format
		-dockerLoginServer="https://index.docker.io/v1/": Docker Login server address
		-dockerPassword="": Password for pulling from docker registry
		-dockerRef="": docker image reference in standard docker string format
		-dockerRegistryHost="": Docker Registry host
		-dockerRegistryIPs=[]: Docker Registry IPs
		-dockerRegistryPort=8080: Docker Registry port
		-dockerUser="": User for pulling from docker registry
		-insecureDockerRegistries=[]: insecure Docker Registry addresses (host:port)
		-outputMetadataJSONFilename="/tmp/result/result.json": filename in which to write the app metadata

还有个**launcher**，一般是做执行docker里定义的脚本使用</br>

还有一个连接sshd的diego-sshd工具：</br>

		diego-sshd
		-address=0.0.0.0:2222，
		-hostKey="",
		-authorizedKey="rsa ---"
		-inheritDaemonEnv
		-logLevel=fatal

最后一个healthcheck</br>

		这个其实在分析之前的源码时候，可以留意一下，健康监测有几种方式，这个是根据port来进行检测的：
		-port=8080
		我们可以在本机上执行一下：
		vagrant@agent-id-bosh-0:~$ ./healthcheck -port 22
		healthcheck passed

* 说完创建容器，还需要了解一下rep是如何把容器启动的：</br>
https://github.com/cloudfoundry-incubator/rep/blob/ab1835570afb992ff756f5d0b3dcb7af1978b639/generator/internal/container_delegate.go#L49</br>

		func (d *containerDelegate) RunContainer(logger lager.Logger, guid string) bool {
			logger.Info("running-container")
			err := d.client.RunContainer(guid)
			if err != nil {
				logInfoOrError(logger, "failed-running-container", err)
				d.DeleteContainer(logger, guid)
				return false
			}
			logger.Info("succeeded-running-container")
			return true
		}

先run，看有没有错，有了则删除容器，然后进到这个runContainer看看它是怎么工作的：</br>
https://github.com/cloudfoundry-incubator/executor/blob/6c38b2fe1a175d8074e0e0164bc9841fd023161c/depot/depot.go#L209</br>

		func (c *client) RunContainer(guid string) error {
			logger := c.logger.Session("run-container", lager.Data{
				"guid": guid,
			})

			logger.Debug("initializing-container")
			//初始化容器
			err := c.allocationStore.Initialize(logger, guid)
			if err != nil {
				logger.Error("failed-initializing-container", err)
				return err
			}
			logger.Debug("succeeded-initializing-container")

			c.creationWorkPool.Submit(func() {
				c.containerLockManager.Lock(guid)
				defer c.containerLockManager.Unlock(guid)

				logger.Debug("looking-up-in-allocation-store")
				container, err := c.allocationStore.Lookup(guid)
				if err != nil {
					logger.Error("failed-looking-up-in-allocation-store", err)
					return
				}
				logger.Debug("succeeded-looking-up-in-allocation-store")

				if container.State != executor.StateInitializing {
					logger.Error("container-state-invalid", err, lager.Data{"state": container.State})
					return
				}

				logger.Info("creating-container-in-garden")
				//在garden中创建容器
				container, err = c.gardenStore.Create(logger, container)
				if err != nil {
					logger.Error("failed-creating-container-in-garden", err)
					c.allocationStore.Fail(logger, guid, ContainerInitializationFailedMessage)
					return
				}
				logger.Info("succeeded-creating-container-in-garden")

				if !c.allocationStore.Deallocate(logger, guid) {
					//如果失败则销毁容器和容器定义
					logger.Info("container-deallocated-during-initialization")

					err = c.gardenStore.Destroy(logger, guid)
					if err != nil {
						logger.Error("failed-to-destroy", err)
					}

					return
				}

				logger.Info("running-container-in-garden")
				//执行运行容器的命令
				err = c.gardenStore.Run(logger, container)
				if err != nil {
					logger.Error("failed-running-container-in-garden", err)
				}
				logger.Info("succeeded-running-container-in-garden")
			})

			return nil
		}

--->前面一直都是初始化工作，跟到容器是怎么被garden创建的：container, err = c.gardenStore.Create(logger, container)</br>
https://github.com/cloudfoundry-incubator/executor/blob/6c38b2fe1a175d8074e0e0164bc9841fd023161c/depot/gardenstore/garden_store.go#L145</br>

		func (store *GardenStore) Create(logger lager.Logger, container executor.Container) (executor.Container, error) {
			if container.State != executor.StateInitializing {
				return executor.Container{}, executor.ErrInvalidTransition
			}
			container.State = executor.StateCreated

			logStreamer := log_streamer.New(
				container.LogConfig.Guid,
				container.LogConfig.SourceName,
				container.LogConfig.Index,
			)

			fmt.Fprintf(logStreamer.Stdout(), "Creating container\n")
			
			//关键，这里才是garden创建容器的入口
			container, err := store.exchanger.CreateInGarden(logger, store.gardenClient, container)
			if err != nil {
				fmt.Fprintf(logStreamer.Stderr(), "Failed to create container\n")
				return executor.Container{}, err
			}

			fmt.Fprintf(logStreamer.Stdout(), "Successfully created container\n")

			return container, nil
		}

* 跟到容器创建的入口：store.exchanger.CreateInGarden(logger, store.gardenClient, container)</br>
不惜一切把代码粘上来：</br>

		func (exchanger exchanger) CreateInGarden(logger lager.Logger, gardenClient GardenClient, executorContainer executor.Container) (executor.Container, error) {
			logger = logger.Session("create-in-garden", lager.Data{"container-guid": executorContainer.Guid})
			
			//容器基本信息
			containerSpec := garden.ContainerSpec{
				Handle:     executorContainer.Guid,
				Privileged: executorContainer.Privileged,
				RootFSPath: executorContainer.RootFSPath,
			}

			//内设置容器并将内存转换成bytes
			if executorContainer.MemoryMB != 0 {
				logger.Debug("setting-up-memory-limits")
				containerSpec.Limits.Memory.LimitInBytes = uint64(executorContainer.MemoryMB * 1024 * 1024)
			}

			logger.Debug("setting-up-disk-limits")
			
			//const DiskLimitScopeExclusive DiskLimitScope = 1
			//磁盘配额限制 这里解释一下，懂运维的可以掠过：
			//dd if=/dev/hda of=/root/image count=1 bs=512 这里是将512byte的数据考到/root/image里去，这里就是所谓的mbr备份
			//if 代表从什么地方，of是写入哪里，bs是输入输出每块的字节数，count代表了要写的块数
			//如果之前做了对某个文件的配额限制，则在超过的时候，将会报错，不再向里写入数据
			gardenScope := garden.DiskLimitScopeExclusive
			
			//DiskLimitScopeTotal DiskLimitScope = 0
			
			if executorContainer.DiskScope == executor.TotalDiskLimit {
				gardenScope = garden.DiskLimitScopeTotal
			}
			
			//一般docker里默认的DiskMB都是1024M
			//ContainerInodeLimit: 200000 inode默认为200000个
			containerSpec.Limits.Disk = garden.DiskLimits{
				ByteHard:  uint64(executorContainer.DiskMB * 1024 * 1024),
				InodeHard: exchanger.containerInodeLimit,
				Scope:     gardenScope,
			}

			logger.Debug("setting-up-cpu-limits")
			
			//ContainerMaxCpuShares: 0 看到这个应该很熟悉，他是cgroup的一个子系统 0为不限制
			//containerMaxCpuShares=1024 不过我们在启动rep的时候已经设定了该值是1024
			//至于权值，我们在分配容器的时候已经采用了默认的100 具体请看上面的AllocateContainers
			containerSpec.Limits.CPU.LimitInShares = uint64(float64(exchanger.containerMaxCPUShares) * float64(executorContainer.CPUWeight) / 100.0)

			logJson, err := json.Marshal(executorContainer.LogConfig)
			if err != nil {
				logger.Error("failed-marshal-log", err)
				return executor.Container{}, err
			}

			metricsConfigJson, err := json.Marshal(executorContainer.MetricsConfig)
			if err != nil {
				logger.Error("failed-marshal-metrics-config", err)
				return executor.Container{}, err
			}

			resultJson, err := json.Marshal(executorContainer.RunResult)
			if err != nil {
				logger.Error("failed-marshal-run-result", err)
				return executor.Container{}, err
			}

			//然后就是一些清单属性
			containerSpec.Properties = garden.Properties{
				ContainerOwnerProperty:         exchanger.containerOwnerName,
				ContainerStateProperty:         string(executorContainer.State),
				ContainerAllocatedAtProperty:   fmt.Sprintf("%d", executorContainer.AllocatedAt),
				ContainerStartTimeoutProperty:  fmt.Sprintf("%d", executorContainer.StartTimeout),
				ContainerRootfsProperty:        executorContainer.RootFSPath,
				ContainerLogProperty:           string(logJson),
				ContainerMetricsConfigProperty: string(metricsConfigJson),
				ContainerResultProperty:        string(resultJson),
				ContainerMemoryMBProperty:      fmt.Sprintf("%d", executorContainer.MemoryMB),
				ContainerDiskMBProperty:        fmt.Sprintf("%d", executorContainer.DiskMB),
				ContainerCPUWeightProperty:     fmt.Sprintf("%d", executorContainer.CPUWeight),
			}

			for name, value := range executorContainer.Tags {
				containerSpec.Properties[tagPropertyPrefix+name] = value
			}

			for _, env := range executorContainer.Env {
				containerSpec.Env = append(containerSpec.Env, env.Name+"="+env.Value)
			}

			for _, securityRule := range executorContainer.EgressRules {
				if err := securityRule.Validate(); err != nil {
					logger.Error("invalid-security-rule", err, lager.Data{"security_group_rule": securityRule})
					return executor.Container{}, executor.ErrInvalidSecurityGroup
				}
			}

			logger.Debug("creating-garden-container")
			gardenContainer, err := gardenClient.Create(containerSpec)
			if err != nil {
				logger.Error("failed-creating-garden-container", err)
				return executor.Container{}, err
			}
			logger.Debug("succeeded-creating-garden-container")

			//设置端口和主机映射端口
			if executorContainer.Ports != nil {
				actualPortMappings := make([]executor.PortMapping, len(executorContainer.Ports))

				logger.Debug("setting-up-ports")
				for i, ports := range executorContainer.Ports {
					actualHostPort, actualContainerPort, err := gardenContainer.NetIn(uint32(ports.HostPort), uint32(ports.ContainerPort))
					if err != nil {
						logger.Error("failed-setting-up-ports", err)
						exchanger.destroyContainer(logger, gardenClient, gardenContainer)
						return executor.Container{}, err
					}

					actualPortMappings[i].ContainerPort = uint16(actualContainerPort)
					actualPortMappings[i].HostPort = uint16(actualHostPort)
				}
				logger.Debug("succeeded-setting-up-ports")

				executorContainer.Ports = actualPortMappings
			}

			//设置安全组 其实是在容器里设置iptables
			//https://github.com/cloudfoundry-incubator/executor/blob/229bbf2af858bc00d14320249a4c16d908435682/depot/gardenstore/exchanger.go#L379
			for _, securityRule := range executorContainer.EgressRules {
				netOutRule, err := securityGroupRuleToNetOutRule(securityRule)
				if err != nil {
					logger.Error("failed-to-build-net-out-rule", err, lager.Data{"security_group_rule": securityRule})
					return executor.Container{}, err
				}

				logger.Debug("setting-up-net-out")
				err = gardenContainer.NetOut(netOutRule)
				if err != nil {
					logger.Error("failed-setting-up-net-out", err, lager.Data{"net-out-rule": netOutRule})
					exchanger.destroyContainer(logger, gardenClient, gardenContainer)
					return executor.Container{}, err
				}
				logger.Debug("succeeded-setting-up-net-out")
			}

			logger.Debug("getting-garden-container-info")
			info, err := gardenContainer.Info()
			if err != nil {
				logger.Error("failed-getting-garden-container-info", err)

				gardenErr := gardenClient.Destroy(gardenContainer.Handle())
				if gardenErr != nil {
					logger.Error("failed-destroy-garden-container", gardenErr)
				}

				return executor.Container{}, err
			}
			logger.Debug("succeeded-getting-garden-container-info")

			//如果设置了externalIp就会有这个，一般为空
			executorContainer.ExternalIP = info.ExternalIP

			return executorContainer, nil
		}

其它的不想分析了，无非就是处理LRPs进程实例和任务之类的，总之都会走到garden那一层。</br>

### Garden组件分析
先看它的启动参数：</br>

		#创建cgroup设备目录 并将设备子系统挂载到cgroup里去
		mkdir /tmp/devices-cgroup
		mount -t cgroup -o $devices_subsytems none /tmp/devices-cgroup
   
		#创建btrfs文件系统，并挂载
		backing_store=/var/vcap/data/garden/garden_graph_backing_store 
		graph_path=/var/vcap/data/garden/btrfs_graph
		mount_point=$graph_path
		loopback_device=$(losetup -f --show $backing_store)
		mkfs.btrfs --nodiscard $loopback_device
		mount -t btrfs $loopback_device $mount_point
   
		/var/vcap/packages/garden-linux/bin/garden-linux \
		-depot=/var/vcap/data/garden/depot \
		-snapshots="${snapshots_path}" \
		-graph=$graph_path \
		-bin=/var/vcap/packages/garden-linux/src/github.com/cloudfoundry-incubator/garden-linux/linux_backend/bin \
		-mtu=1500 \
		-disableQuotas=false \
		-listenNetwork=tcp \
		-listenAddr=0.0.0.0:7777 \
		-denyNetworks=0.0.0.0/0 \
		-allowNetworks= \
		-allowHostAccess=false \
		-debugAddr=0.0.0.0:17013 \
		-rootfs=/var/vcap/packages/busybox \
		-containerGraceTime=5m
	  
系统的rootfses:</br>

		mkdir -p $RUN_DIR
		chown -R vcap:vcap $RUN_DIR

		echo $$ > $PIDFILE

		# Setup rootfs
		ROOTFS_PACKAGE=/var/vcap/packages/rootfs_cflinuxfs2/
		ROOTFS_DIR=$ROOTFS_PACKAGE/rootfs
		if [ ! -d $ROOTFS_DIR ]; then
		  mkdir -p $ROOTFS_DIR
		  tar -pzxf $ROOTFS_PACKAGE/cflinuxfs2.tar.gz -C $ROOTFS_DIR
		fi

### 分析：</br>
Garden:

* create/delete containers
* apply resource limits to containers
* open and attach network ports to containers
* copy files into/out of containers
* run processes within containers, streaming back stdout and stderr data
* annotate containers with arbitrary metadata
* snapshot containers for down-timeless redeploys

看到最后一条，重新部署的时候能建立快照，其它的都是基础的资源隔离还应用</br>

真正干活的是Garden-Linux</br>

还是老习惯，先看routes 这里可以很清晰的看到garden会接收哪些restful的请求</br>

		var Routes = rata.Routes{
			{Path: "/ping", Method: "GET", Name: Ping},
			{Path: "/capacity", Method: "GET", Name: Capacity},

			{Path: "/containers", Method: "GET", Name: List},
			{Path: "/containers", Method: "POST", Name: Create},

			{Path: "/containers/:handle/info", Method: "GET", Name: Info},
			{Path: "/containers/bulk_info", Method: "GET", Name: BulkInfo},
			{Path: "/containers/bulk_metrics", Method: "GET", Name: BulkMetrics},

			{Path: "/containers/:handle", Method: "DELETE", Name: Destroy},
			{Path: "/containers/:handle/stop", Method: "PUT", Name: Stop},

			{Path: "/containers/:handle/files", Method: "PUT", Name: StreamIn},
			{Path: "/containers/:handle/files", Method: "GET", Name: StreamOut},

			{Path: "/containers/:handle/limits/bandwidth", Method: "PUT", Name: LimitBandwidth},
			{Path: "/containers/:handle/limits/bandwidth", Method: "GET", Name: CurrentBandwidthLimits},

			{Path: "/containers/:handle/limits/cpu", Method: "PUT", Name: LimitCPU},
			{Path: "/containers/:handle/limits/cpu", Method: "GET", Name: CurrentCPULimits},

			{Path: "/containers/:handle/limits/disk", Method: "PUT", Name: LimitDisk},
			{Path: "/containers/:handle/limits/disk", Method: "GET", Name: CurrentDiskLimits},

			{Path: "/containers/:handle/limits/memory", Method: "PUT", Name: LimitMemory},
			{Path: "/containers/:handle/limits/memory", Method: "GET", Name: CurrentMemoryLimits},

			{Path: "/containers/:handle/net/in", Method: "POST", Name: NetIn},
			{Path: "/containers/:handle/net/out", Method: "POST", Name: NetOut},

			{Path: "/containers/:handle/processes/:pid/attaches/:streamid/stdout", Method: "GET", Name: Stdout},
			{Path: "/containers/:handle/processes/:pid/attaches/:streamid/stderr", Method: "GET", Name: Stderr},
			{Path: "/containers/:handle/processes", Method: "POST", Name: Run},
			{Path: "/containers/:handle/processes/:pid", Method: "GET", Name: Attach},

			{Path: "/containers/:handle/properties", Method: "GET", Name: Properties},
			{Path: "/containers/:handle/properties/:key", Method: "GET", Name: Property},
			{Path: "/containers/:handle/properties/:key", Method: "PUT", Name: SetProperty},
			{Path: "/containers/:handle/properties/:key", Method: "DELETE", Name: RemoveProperty},

			{Path: "/containers/:handle/metrics", Method: "GET", Name: Metrics},
		}

这里的handle就是之前的container的ID：c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6-fc4a83cd-5ed9-40fb-51bf-1679beb41bfe</br>

我们可以随便执行一下：/info 获取一个容器的info</br>

		{
		  "State": "active", 
		  "Events": [ ], 
		  "HostIP": "10.254.0.2", 
		  "ContainerIP": "10.254.0.1", 
		  "ExternalIP": "10.244.16.138", 
		  "ContainerPath": "/var/vcap/data/garden/depot/vrishcl40k3", 
		  "ProcessIDs": [
			1, 
			2
		  ], 
		  "Properties": {
			"executor:allocated-at": "1441087199719774323", 
			"executor:cpu-weight": "1", 
			"executor:disk-mb": "1024", 
			"executor:log-config": "{"guid":"c181be6b-d89c-469c-b79c-4cab1e99b8de","index":0,"source_name":"CELL"}", 
			"executor:memory-mb": "256", 
			"executor:metrics-config": "{"guid":"c181be6b-d89c-469c-b79c-4cab1e99b8de","index":0}", 
			"executor:owner": "executor", 
			"executor:result": "{"failed":false,"failure_reason":"","stopped":false}", 
			"executor:rootfs": "docker:///tutum/tomcat#8.0", 
			"executor:start-timeout": "60", 
			"executor:state": "running", 
			"tag:domain": "cf-apps", 
			"tag:instance-guid": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe", 
			"tag:lifecycle": "lrp", 
			"tag:process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
			"tag:process-index": "0"
		  }, 
		  "MappedPorts": [
			{
			  "HostPort": 60000, 
			  "ContainerPort": 8080
			}, 
			{
			  "HostPort": 60001, 
			  "ContainerPort": 2222
			}
		  ]
		}

好了，我们来看看容器是怎么创建的：</br>
https://github.com/cloudfoundry-incubator/garden/blob/master/server/request_handling.go#L52</br>

		func (s *GardenServer) handleCreate(w http.ResponseWriter, r *http.Request) {
			var spec garden.ContainerSpec
			if !s.readRequest(&spec, w, r) {
				return
			}

			hLog := s.logger.Session("create", lager.Data{
				"request": containerDebugInfo{
					Handle:     spec.Handle,
					GraceTime:  spec.GraceTime,
					RootFSPath: spec.RootFSPath,
					BindMounts: spec.BindMounts,
					Network:    spec.Network,
					Privileged: spec.Privileged,
					Limits:     spec.Limits,
				},
			})

			if spec.GraceTime == 0 {
				spec.GraceTime = s.containerGraceTime
			}

			hLog.Debug("creating")

			//关键看这个方法，前面的方法都是在构建spec清单也就是task或者DesiredLsp的清单
			container, err := s.backend.Create(spec)
			if err != nil {
				s.writeError(w, err, hLog)
				return
			}

			hLog.Info("created")

			s.bomberman.Strap(container)

			s.writeResponse(w, &struct{ Handle string }{
				Handle: container.Handle(),
			})
		}

然后看到这里：https://github.com/cloudfoundry-incubator/garden/blob/master/client/connection/connection.go#L126</br>

		func (c *connection) Create(spec garden.ContainerSpec) (string, error) {
			res := struct {
				Handle string `json:"handle"`
			}{}

			err := c.do(routes.Create, spec, &res, nil, nil)
			if err != nil {
				return "", err
			}

			return res.Handle, nil
		}

继续,这里就是请求json的解析了，不过实现的很优雅：</br>

		func (c *connection) do(
			handler string,
			req, res interface{},
			params rata.Params,
			query url.Values,
		) error {
			var body io.Reader

			if req != nil {
				buf := new(bytes.Buffer)

				err := transport.WriteMessage(buf, req)
				if err != nil {
					return err
				}

				body = buf
			}

			contentType := ""
			if req != nil {
				contentType = "application/json"
			}

			response, err := c.hijacker.Stream(
				handler,
				body,
				params,
				query,
				contentType,
			)
			if err != nil {
				return err
			}

			defer response.Close()

			return json.NewDecoder(response).Decode(res)
		}

### 继续往garden-linux看吧，这是个庞大，也是整个系统里最核心的部分了

* garden-linux

当garden-server给garden-linux发出建立容器的和运行容器的任务的时候，后端即开始执行相应的操作</br>

其中**garden-server**在启动的过程：</br>

1.初始化docker的Graph驱动：</br>

		dockerGraphDriver, err := graphdriver.New(*graphRoot, nil)
		dockerGraph, err := graph.NewGraph(*graphRoot, dockerGraphDriver)

2.然后挂载btrfs格式化过的文件系统</br>

		graphMountPoint := mountPoint(logger, *graphRoot)

3.设置garden的基本配置</br>

		cake = &layercake.BtrfsCleaningCake{
			Cake:            cake,
			Runner:          runner,
			BtrfsMountPoint: graphMountPoint,
			RemoveAll:       os.RemoveAll,
			Logger:          logger.Session("btrfs-cleanup"),
		}
		
4.设置repository_fetcher</br>

里面有4个参数：dockerRegistry,cake,map[registry.APIVersion]repository_fetcher.VersionedFetcher,repository_fetcher.EndpointPinger{}
这里是根据不同版本的registry api版本摄者fetcher到自己的repository_fetcher里</br>

5.设置uidNamespace,划分uid gid范围</br>

		rootFSNamespacer := &rootfs_provider.UidNamespacer
		
6.更具RootFsProvider设置 rootfs_provider：</br>

		//docker的rootfs
		remoteRootFSProvider, err := rootfs_provider.NewDocker(fmt.Sprintf("docker-remote-%s", cake.DriverName()),
		repoFetcher, cake, rootfs_provider.SimpleVolumeCreator{}, rootFSNamespacer, clock.NewClock())

		//自家warden的rootfs
		localRootFSProvider, err := rootfs_provider.NewDocker(fmt.Sprintf("docker-local-%s", cake.DriverName()),
		&repository_fetcher.Local{
			Cake:              cake,
			DefaultRootFSPath: *rootFSPath,
			IDProvider:        repository_fetcher.LayerIDProvider{},
		}, cake, rootfs_provider.SimpleVolumeCreator{}, rootFSNamespacer, clock.NewClock())
		
		rootFSProviders := map[string]rootfs_provider.RootFSProvider{
			"":       localRootFSProvider,
			"docker": remoteRootFSProvider,
		}
	
7.设置externalIP，实际上是local ip</br>

	ip, err := localip.LocalIP()

8.设置quotaManager 根据graphMountPoint配置属于btrfs的配额规则</br>

		var quotaManager linux_container.QuotaManager = quota_manager.DisabledQuotaManager{}
		if !*disableQuotas {
			quotaManager = &quota_manager.BtrfsQuotaManager{
				Runner:     runner,
				MountPoint: graphMountPoint,
			}
		}

9.配置一些其它资源pool，iptables,mtu，subnetPool等，后面具体分析</br>

10.根据以上的资源分配，设置linux_backed</br>

		backend := linux_backend.New(logger, pool, container_repository.New(), injector, systemInfo, *snapshotsPath, int(*maxContainers))
		err = backend.Setup()

11.如果以上都没有错，则开始启动gardenServer</br>

		graceTime := *containerGraceTime
		gardenServer := server.New(*listenNetwork, *listenAddr, graceTime, backend, logger)
		

---------------------------------------------------------------------------------------------------
官方有两个涉及图：一个是容器创建过程，一个是gardenServer如何跟backed后端进程通信的</br>
![github](http://github.com/wdxxs2z/PictureStore/diego/container creation.png "github")

容器构建其实和上一代的warden没有什么区别，区别在于**AcquirePoolResources**和**AcquireSystemResources**这两个方法</br>

完全是为了适配docker而生,看到resource_pool：</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/master/resource_pool/resource_pool.go</br>

直接去看里面的Acquire,看一下garden是如何配置容器的</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/master/resource_pool/resource_pool.go#L244</br>

		(p *LinuxResourcePool) Acquire(spec garden.ContainerSpec) (linux_backend.LinuxContainerSpec, error)

这里有个参数，garden.ContainerSpec，这个sPec值的是之前我们定义container的一些参数，环境变量和安全组等等</br>

1.分配container id ,container path,depotPath一般为：/var/vcap/data/garden/depot</br>

		containerID：id := <-p.containerIDs，
		containerPath := path.Join(p.depotPath, id)
		
2.开始获取poolResource</br>

		resources, err := p.acquirePoolResources(spec, id)
		https://github.com/cloudfoundry-incubator/garden-linux/blob/master/resource_pool/resource_pool.go#L484

		func (p *LinuxResourcePool) acquirePoolResources(spec garden.ContainerSpec, id string) (*linux_backend.Resources, error) {
			//其实是CELL的IP
			resources := linux_backend.NewResources(0, nil, "", nil, p.externalIP)
			
			//解析spec中的network部分
			subnet, ip, err := parseNetworkSpec(spec.Network)
			if err != nil {
				return nil, fmt.Errorf("create container: invalid network spec: %v", err)
			}

			//根据Privileged判断是否是root权限，如果是uid肯定是0，一般在build镜像的时候这个值一般都为true，uid为root
			if err := p.acquireUID(resources, spec.Privileged); err != nil {
				return nil, err
			}

			//根据subnet和ip设置resources.Network
			//https://github.com/cloudfoundry-incubator/garden-linux/blob/59c89dc849e992f5a5f7531889c493cfd844bc4d/network/subnets/subnets.go#L69
			if resources.Network, err = p.subnetPool.Acquire(subnet, ip); err != nil {
				p.releasePoolResources(resources)
				return nil, err
			}

			return resources, nil
		}

3.分配handleId,如果先前没有分配，则把ID设置成handlerId</br>

		handle := getHandle(spec.Handle, id)

4.设置磁盘配额</br>

		var quota int64 = int64(spec.Limits.Disk.ByteHard)
		if quota == 0 {
			quota = math.MaxInt64
		}

5.设置containerRootFSPath, rootFSEnv，重点是acquireSystemResources</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/master/resource_pool/resource_pool.go#L268</br>

		containerRootFSPath, rootFSEnv, err := p.acquireSystemResources(id, handle, containerPath, spec.RootFSPath, resources, spec.BindMounts, quota, pLog)

**spec.BindMounts**是我们常说的docker volume</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/master/resource_pool/resource_pool.go#L524</br>

1).创建containerPath</br>

		os.MkdirAll(containerPath, 0755)
		
2).设置rootfsUrl docker一般为docker:\\\</br>
3).设置rootfsProviders，docker，warden</br>

		provider, found := p.rootfsProviders[rootfsURL.Scheme]
		
4).根据不同的provider设置不同的rootfsPath</br>

		rootfsPath, rootFSEnvVars, err := provider.ProvideRootFS(pLog.Session("create-rootfs"), id, rootfsURL, resources.RootUID != 0, diskQuota)
		
docker一般会去找自己的layer，这里的layer已经被garden化了，也就是在btrfs目录下</br>
如果是普通的buildpack，则会去加载自家的rootfs：/var/vcap/packages/rootfs_cflinuxfs2/rootfs</br>

5).为当前容器分配一个网桥，这个设置其实是为了方便同一个CELL中的不同容器通信用的，因为实际CIDR为/30的划分，本身就2个IP可用</br>

6).设置网桥名：一般以wb打头 后面是容器ID</br>

7).后面就是一些列的容器创建动作了，从create.sh这个脚本开始，设置环境变量等，然后看到一个方法：</br>

		err = p.writeBindMounts(containerPath, rootfsPath, bindMounts)
		每个containerPath下都有一个lib目录，里面有几个脚本，其实看到后应该很熟悉，在CF的v2版里也有，用来设置cgroup
		hook-parent-before-clone.sh 这里就不分析这了

6.上述所有都创建完成后，开始合并环境变量</br>

		specEnv, err := process.NewEnv(spec.Env)
		spec.Env = rootFSEnv.Merge(specEnv).Array()

7.最终返回一个结构体：</br>

		return linux_backend.LinuxContainerSpec{
			ID:                  id,
			ContainerPath:       containerPath,
			ContainerRootFSPath: containerRootFSPath,
			Resources:           resources,
			Events:              []string{},
			Version:             p.currentContainerVersion,
			State:               linux_backend.StateBorn,

			ContainerSpec: spec,
		}, nil

到这里容器资源部分就已经创建完了。

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
* 现在可以将焦点聚焦在两个地方，一个是garden是如何构建docker镜像的，一个是garden是如何创建网络的</br>

		因为之前都有rootfsProvider：
		type RootFSProvider interface {
			Name() string
			ProvideRootFS(logger lager.Logger, id string, rootfs *url.URL, namespaced bool, quota int64) (mountpoint string, envvar process.Env, err error)
		}

		type Graph interface {
			layercake.Cake
		}

其中下面graph，如果研究过docker源码，应该知道docker有一个graph driver用来构建layer做可堆叠文件系统的驱动，里面有很多实现，如aufs,btrfs，还有自家的DeviceMapper，其实就为了创建优雅的metadata</br>

ProvideRootFS：构建docker image</br>

1.默认设置一个tag := "latest" </br>

2.fetch镜像</br>

		fetchedID, envvars, volumes, err := provider.repoFetcher.Fetch(logger, url, tag, quota)
		
如果想知道是如何fetch并下载镜像的，可以从这里入手：</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/4c869ef07d712cfe007c4ed1f81b576efa640c04/repository_fetcher/remote_v1.go</br>

我们在使用docker的时候，在pull镜像的时候，一般格式是some-repository-name:target 不过严格上来说是docker:///加some-repository-name:target</br>
garden也不例外：</br>

1).获取镜像的metadata</br>

		imgID, endpoints, err := fetcher.fetchImageMetadata(request)
		
2).通过迭代endpoints，获取image,比如我们使用docker时会看到：</br>

		31fa814ba25a: Pulling image (latest) from training/webapp, endpoint: https://reg31fa814ba25a: Pulling dependent layers
		image, err := fetcher.fetchFromEndpoint(request, endpointURL, imgID, request.Logger)
		
跟进去：https://github.com/cloudfoundry-incubator/garden-linux/blob/4c869ef07d712cfe007c4ed1f81b576efa640c04/repository_fetcher/remote_v1.go#L79</br>

一般一个image会分很多层，所以会一层一层的获取，在获取某一层之前会先检查这一层的layer是否已经被缓存过了</br>

		var allLayers []*dockerLayer
		layer, err := fetcher.fetchLayer(request, endpointURL, history[i], remainingQuota, logger)
		allLayers = append(allLayers, layer)
		
继续跟到fetchLayer:</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/4c869ef07d712cfe007c4ed1f81b576efa640c04/repository_fetcher/remote_v1.go#L109</br>

		func (fetcher *RemoteV1Fetcher) fetchLayer(request *FetchRequest, endpointURL string, layerID string, remaining int64, logger lager.Logger) (*dockerLayer, error)
		检查是否被缓存，如果有，则直接返回这个layer，没有的，则通过下面继续获取：
		fetcher.Cake.Get(layercake.DockerImageID(layerID))
		每次下载都会开启计时，然后统计下载完成所用的时间：took，最后你会看到两种状态，一个是downloading和download

3.根据imageId和containerID构建出garden自己的rootfs</br>

		provider.graph.Create(containerID, imageID)

https://github.com/cloudfoundry-incubator/garden-linux/blob/964c92719378f8ef0bdbe60726e2f9ab42b69850/layercake/docker.go#L24</br>

		func (d *Docker) Create(containerID ID, imageID ID) error {
			return d.Register(
				&image.Image{
					ID:     containerID.GraphID(),
					Parent: imageID.GraphID(),
				}, nil)
		}

可以看到其实garden在存储自己的镜像时，一个是containerId,也就是实例ID，还有一个是它会记录一份docker Image的ID</br>

4.如果有volume，则在容器的graph的文件系统里创建出一个volume,这里官方只说，目前只是简单的实现创建，还没有做任何管理。</br>

通过查看日志，我们得知：645c4570fd120f7ed5bed9277886af4797e1874e3d5b571257b8ffdc6596bf9e为最后的imageId,然后我们还会看到一个08on3iof61u，他们都在
/var/vcap/data/garden/btrfs_graph/btrfs/subvolumes/这个目录下，根据分析的源码，也就知道了这个08on3iof61u就是containerID.GraphID()，其实也是存放容器rootPath的地方
/var/vcap/data/garden/depot/08on3iof61u</br>

我们还能看到一个NamespacedLayerID：645c4570fd120f7ed5bed9277886af4797e1874e3d5b571257b8ffdc6596bf9e@0-4294967294-1,1-1-4294967293+0-4294967294-1,1-1-4294967293</br>
		
		func (n NamespacedLayerID) GraphID() string {
			return shaID(n.LayerID + "@" + n.CacheKey)
		}

补一个知识，在构建每一层的时候，都会在这一layer的创建json和size数据，这个和docker相符。</br>
/var/vcap/data/garden/btrfs_graph/{layerId}</br>

		{
		  "id": "ff365bfa7ca61680fbbe4b27d3473d7b5d76adde64c199fb312dcd30c3302b0f", 
		  "parent": "a827709e978385e0e2998703fbe17f934c1a6bc233c7eb98820140ed5e279c23", 
		  "created": "2015-07-26T17:15:16.767121317Z", 
		  "container": "3a0dd2e1601f9def91d517491ccb1ce20abc9f8ac720f22e1b2e533bbb5039db", 
		  "container_config": {
			"Hostname": "dd360632d03c", 
			"Domainname": "", 
			"User": "", 
			"AttachStdin": false, 
			"AttachStdout": false, 
			"AttachStderr": false, 
			"PortSpecs": null, 
			"ExposedPorts": null, 
			"Tty": false, 
			"OpenStdin": false, 
			"StdinOnce": false, 
			"Env": [
			  "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", 
			  "HOME=/root"
			], 
			"Cmd": [
			  "/bin/sh", 
			  "-c", 
			  "#(nop) ENTRYPOINT ["/scripts/run.sh"]"
			], 
			"Image": "a827709e978385e0e2998703fbe17f934c1a6bc233c7eb98820140ed5e279c23", 
			"Volumes": null, 
			"VolumeDriver": "", 
			"WorkingDir": "/root", 
			"Entrypoint": [
			  "/scripts/run.sh"
			], 
			"NetworkDisabled": false, 
			"MacAddress": "", 
			"OnBuild": [ ], 
			"Labels": { }
		  }, 
		  "docker_version": "1.6.2", 
		  "author": "Ferran Rodenas <frodenas@gmail.com>", 
		  "config": {
			"Hostname": "dd360632d03c", 
			"Domainname": "", 
			"User": "", 
			"AttachStdin": false, 
			"AttachStdout": false, 
			"AttachStderr": false, 
			"PortSpecs": null, 
			"ExposedPorts": null, 
			"Tty": false, 
			"OpenStdin": false, 
			"StdinOnce": false, 
			"Env": [
			  "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", 
			  "HOME=/root"
			], 
			"Cmd": null, 
			"Image": "a827709e978385e0e2998703fbe17f934c1a6bc233c7eb98820140ed5e279c23", 
			"Volumes": null, 
			"VolumeDriver": "", 
			"WorkingDir": "/root", 
			"Entrypoint": [
			  "/scripts/run.sh"
			], 
			"NetworkDisabled": false, 
			"MacAddress": "", 
			"OnBuild": [ ], 
			"Labels": { }
		  }, 
		  "architecture": "amd64", 
		  "os": "linux", 
		  "Size": 0
		}

		laylersize:0

看到这里，其实还有一个疑问，就是btrfs怎么没有出现，其实btrfs在创建和挂载好后，我们所有的镜像操作都只在btrfs的文件系统里操作，至于怎么实现，这里就和
garden无关了，因为它依赖的是docker的实现，copy on write 机制，说到具体用，可能你也猜到了，就是在配额管理的时候使用：</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/6b419ed1e7020930425adc05bc29602dc774eb16/linux_container/quota_manager/btrfs_quota_manager.go</br>
这里就不进去了，主要为btrfs的磁盘配额进行限制和获取btrfs磁盘配额的具体信息等。</br>


* 接下来看看garden是如何构建网络的：</br>
我们知道docker在创建网络时会分三步，一个是在deamon启动的时候会初始化一个docker bridge,第二步则在创建容器的时候或者说启动的时候会分配一个veth pair，一端在容器里，
一端patch到docker网桥上，最后是将容器一端的container分配到现有的PID中也就是namespace中。</br>
这里garden比较特别，它会在启动每个容器的时候创建veth的同时会为每个子网（CIDR 30）创建一个bridge，并将veth的另一端patch到这个bridge上去</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/master/network/configure.go#L55</br>

* Bridge:

		Veth interface {
			Create(hostIfcName, containerIfcName string) (*net.Interface, *net.Interface, error)
		}
		Bridge interface {
			Create(bridgeName string, ip net.IP, subnet *net.IPNet) (*net.Interface, error)
			Add(bridge, slave *net.Interface) error
		}
		
先提前说一下bridgeName会放在每个实例下的一个bridge-name文件里：</br>
/var/vcap/data/garden/depot/08on3iof61u：wb-08on3ioescs0</br>

		//name就是网桥名，ip是网桥的ip，subnet是子网 一般是cidr /30,具体实现在docker的libcontainer/netlink下
		func (Bridge) Create(name string, ip net.IP, subnet *net.IPNet) (intf *net.Interface, err error) {
			netlinkMu.Lock()
			defer netlinkMu.Unlock()

			if err := netlink.NetworkLinkAdd(name, "bridge"); err != nil && err.Error() != "file exists" {
				return nil, fmt.Errorf("devices: create bridge: %v", err)
			}

			if intf, err = net.InterfaceByName(name); err != nil {
				return nil, fmt.Errorf("devices: look up created bridge interface: %v", err)
			}

			if err = netlink.NetworkLinkAddIp(intf, ip, subnet); err != nil && err.Error() != "file exists" {
				return nil, fmt.Errorf("devices: add IP to bridge: %v", err)
			}
			return intf, nil
		}

* Veth:
hostIfcName为容器内主机的名称，containerIfcName为容器实例名</br>

		func (VethCreator) Create(hostIfcName, containerIfcName string) (host, container *net.Interface, err error) {
			netlinkMu.Lock()
			defer netlinkMu.Unlock()

			if err := netlink.NetworkCreateVethPair(hostIfcName, containerIfcName, 1); err != nil {
				return nil, nil, fmt.Errorf("devices: create veth pair: %v", err)
			}

			if host, err = net.InterfaceByName(hostIfcName); err != nil {
				return nil, nil, fmt.Errorf("devices: look up created host interface: %v", err)
			}

			if container, err = net.InterfaceByName(containerIfcName); err != nil {
				return nil, nil, fmt.Errorf("devices: look up created container interface: %v", err)
			}

			return host, container, nil
		}
		
然后就是将veth的另一端和bridge串起来：</br>

		c.configureHostIntf(cLog, host, bridge, config.Mtu)

最后在将container的末端IP加入到namespace中去：</br>

		// move container end in to container
		if err = c.Link.SetNs(container, config.ContainerPid); err != nil {
			return &SetNsFailedError{err, container, config.ContainerPid}
		}

至此我们来看一下具体的网络情况，这个时候我们已经上传了一个docker应用：</br>

		w08on3iof61u-0 Link encap:Ethernet  HWaddr 6a:54:f3:2f:23:00  
				  inet6 addr: fe80::6854:f3ff:fe2f:2300/64 Scope:Link
				  UP BROADCAST RUNNING  MTU:1500  Metric:1
				  RX packets:12 errors:0 dropped:0 overruns:0 frame:0
				  TX packets:19 errors:0 dropped:0 overruns:0 carrier:0
				  collisions:0 txqueuelen:1 
				  RX bytes:928 (928.0 B)  TX bytes:1486 (1.4 KB)

		w08on3iof621-0 Link encap:Ethernet  HWaddr 72:74:d0:b1:e4:d7  
				  inet6 addr: fe80::7074:d0ff:feb1:e4d7/64 Scope:Link
				  UP BROADCAST RUNNING  MTU:1500  Metric:1
				  RX packets:11 errors:0 dropped:0 overruns:0 frame:0
				  TX packets:18 errors:0 dropped:0 overruns:0 carrier:0
				  collisions:0 txqueuelen:1 
				  RX bytes:801 (801.0 B)  TX bytes:1434 (1.4 KB)

		wb-08on3ioescs0 Link encap:Ethernet  HWaddr 6a:54:f3:2f:23:00  
				  inet addr:10.254.0.2  Bcast:0.0.0.0  Mask:255.255.255.252
				  inet6 addr: fe80::b4ad:1ff:fede:9357/64 Scope:Link
				  UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
				  RX packets:12 errors:0 dropped:0 overruns:0 frame:0
				  TX packets:12 errors:0 dropped:0 overruns:0 carrier:0
				  collisions:0 txqueuelen:0 
				  RX bytes:760 (760.0 B)  TX bytes:928 (928.0 B)

		wb-08on3ioescv0 Link encap:Ethernet  HWaddr 72:74:d0:b1:e4:d7  
				  inet addr:10.254.0.6  Bcast:0.0.0.0  Mask:255.255.255.252
				  inet6 addr: fe80::6c45:aeff:fe38:7803/64 Scope:Link
				  UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
				  RX packets:11 errors:0 dropped:0 overruns:0 frame:0
				  TX packets:11 errors:0 dropped:0 overruns:0 carrier:0
				  collisions:0 txqueuelen:0 
				  RX bytes:647 (647.0 B)  TX bytes:876 (876.0 B)

root@5ccb272f-018c-4b31-a5a4-92c8c7626a4c:/tmp/devices-cgroup/instance-08jaqet7bv6/instance-08on3iof61u# brctl show </br>

		bridge name     bridge id               STP enabled     interfaces
		wb-08on3ioescs0         8000.6a54f32f2300       no              w08on3iof61u-0
		wb-08on3ioescv0         8000.7274d0b1e4d7       no              w08on3iof621-0

root@5ccb272f-018c-4b31-a5a4-92c8c7626a4c:/tmp/devices-cgroup/instance-08jaqet7bv6/instance-08on3iof61u# bridge li</br>

		4: w08on3iof61u-0 state UP : <BROADCAST,UP,LOWER_UP> mtu 1500 master wb-08on3ioescs0 state forwarding priority 32 cost 2 
		13: w08on3iof621-0 state UP : <BROADCAST,UP,LOWER_UP> mtu 1500 master wb-08on3ioescv0 state forwarding priority 32 cost 2
