# Access 部分(初版)
下面开始从access层开始简要源码分析，如果后期发现问题，择后在改 <br />

access一共有2个组件组成
-----------------------------------
* Receptor  负责接收CC-bridge发来的创建tasks和Lrps请求，以前在dea中，是没有任务和长运行进程之说的，而且提供了restful API 打通了 后端garden-linux和各个组建之间的联系 同时还保证了与BBS端的数据同步
* ssh-proxy 一个ssh端的负载均衡器

### 在一次提及一下Desired和Actual这两个LRP关键词
DesiredLRP里存放的是一份清单，其最终存放在database里，也就是它保证了后续应用实例如果挂了可以自愈，所以desiredLRP不是多余的</br>
ActualLRP里维护的是实例所依赖的DesiredLRP还有一些实例运行信息，这些也都是存放在database里的</br>

### Receptor组件分析
先看它的启动参数：</br>

		/var/vcap/packages/receptor/bin/receptor ${etcd_sec_flags} \
		-address=0.0.0.0:8887 \
		-bbsAddress=http://bbs.service.cf.internal:8889 \
		-taskHandlerAddress=10.244.16.6:1169 \
		-debugAddr=0.0.0.0:17014 \
		-etcdCluster="https://etcd.service.cf.internal:4001" \
		-consulCluster=http://127.0.0.1:8500 \
		-username= \
		-password= \
		-registerWithRouter=true \
		-domainNames=receptor.10.244.0.34.xip.io \
		-natsAddresses=10.244.0.6:4222 \
		-natsUsername=nats \
		-natsPassword=nats \
		-corsEnabled=false \
		-logLevel=debug
		
### 分析：
先不分析源码，先看两个实例.我们注意到receptor实际上提供了一层友好的restful api供我们访问，具体我们能看到什么？</br>
通过调用http://receptor.10.244.0.34.xip.io/v1/desired_lrps </br>
可以看到它只是一份清单，里面说明了这个应用有多少实例，采用的rootfs，资源限制，安全组等等。</br>
<string>DesiredLRP</strong></br>

		[
		{
		"process_guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
		"domain": "cf-apps", 
		"rootfs": "docker:///tutum/tomcat#8.0", 
		"instances": 2, 
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
		"action": {
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
		"start_timeout": 60, 
		"disk_mb": 1024, 
		"memory_mb": 256, 
		"cpu_weight": 1, 
		"privileged": false, 
		"ports": [
		  8080, 
		  2222
		], 
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
		}, 
		"log_guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
		"log_source": "CELL", 
		"metrics_guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
		"annotation": "1441088697.8285718", 
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
		], 
		"modification_tag": {
		  "epoch": "c69ce119-896b-4a3a-77be-3dc793908340", 
		  "index": 3
		}
		}
		]
	
我们在来看看ActualLRP里面定义了什么：</br>
还是通过调用 http://receptor.10.244.0.34.xip.io/v1/actual_lrps</br>
从下面可以看到，actualLRP只关心应用的运行实例内部容器的具体信息，包括host主机地址，容器端口和主机映射端口，运行状态等等。</br>
<string>ActualLRP</strong></br>

		[
		{
		"process_guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
		"instance_guid": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe", 
		"cell_id": "cell_z1-0", 
		"domain": "cf-apps", 
		"index": 0, 
		"address": "10.244.16.138", 
		"ports": [
		  {
			"container_port": 8080, 
			"host_port": 60000
		  }, 
		  {
			"container_port": 2222, 
			"host_port": 60001
		  }
		], 
		"state": "RUNNING", 
		"crash_count": 0, 
		"since": 1441087965748636200, 
		"evacuating": false, 
		"modification_tag": {
		  "epoch": "d292b1b4-fd3e-4df1-5ea3-733e14a616ae", 
		  "index": 27
		}
		}, 
		{
		"process_guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
		"instance_guid": "21811b3c-a37e-4277-5878-c646a7213bf0", 
		"cell_id": "cell_z1-0", 
		"domain": "cf-apps", 
		"index": 1, 
		"address": "10.244.16.138", 
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
		"state": "RUNNING", 
		"crash_count": 0, 
		"since": 1441088706589340400, 
		"evacuating": false, 
		"modification_tag": {
		  "epoch": "c5e63fbb-eea9-4c1c-4a4a-1513ecfbb781", 
		  "index": 2
		}
		}
		]

了解了上面两个LRPs，我们来看看环境变量，主要看Networking Information</br>
以下的功能是比较cool的功能，首先你要在cells里的rep组建中将-exportNetworkEnvVars 设置成enable</br>
CF_INSTANCE_IP：可以指定实例在主机运行的IP，</br>
CF_INSTANCE_PORT： 这个端口设定对应这desiredLRP中ports数组中第一个port的映射</br>
CF_INSTANCE_ADDR： $CF_INSTANCE_IP:$CF_INSTANCE_PORT</br>
CF_INSTANCE_PORTS： [{"external":60413,"internal":8080},{"external":60414,"internal":2222}]</br>
	
接下来我们还能看到无论是tasks还是lrps在执行的时候都有动作的执行，比如在编译的时候会有download,upload等动作</br>
**RunAction**: 在容器中运行一个进程</br>
**DownloadAction**: 获取一个tgz或者zip包并解压它进容器中</br>
**UploadAction**: 触发从容器中上传一个单独的文件到指定的fileserver,这里一般为droplet</br>
**ParallelAction**: 同时运行多个动作</br>
**CodependentAction**: 并行的执行所有的动作，只要有一个执行成功，则退出所有任务，这个设置太给力了</br>
**SerialAction**: 按照一定顺序执行动作</br>
**EmitProgressAction**: 嵌入式的执行动作</br>
**TimeoutAction**: 假如这个嵌入式的动作在一定时间内没有完成则报错</br>
**TryAction**: 如果在执行嵌入式的动作时有错误，则将其忽略</br>
	
现在开始看一下源码：</br>
	
以下定义了基本的常量和restful请求等等。</br>

		const (
		// Tasks
		CreateTaskRoute = "CreateTask"
		TasksRoute      = "Tasks"
		GetTaskRoute    = "GetTask"
		DeleteTaskRoute = "DeleteTask"
		CancelTaskRoute = "CancelTask"

		// DesiredLRPs
		CreateDesiredLRPRoute = "CreateDesiredLRP"
		GetDesiredLRPRoute    = "GetDesiredLRP"
		UpdateDesiredLRPRoute = "UpdateDesiredLRP"
		DeleteDesiredLRPRoute = "DeleteDesiredLRP"
		DesiredLRPsRoute      = "DesiredLRPs"

		// ActualLRPs
		ActualLRPsRoute                         = "ActualLRPs"
		ActualLRPsByProcessGuidRoute            = "ActualLRPsByProcessGuid"
		ActualLRPByProcessGuidAndIndexRoute     = "ActualLRPByProcessGuidAndIndex"
		KillActualLRPByProcessGuidAndIndexRoute = "KillActualLRPByProcessGuidAndIndex"

		// Cells
		CellsRoute = "Cells"

		// Domains
		UpsertDomainRoute = "UpsertDomain"
		DomainsRoute      = "Domains"

		// Event Streaming
		EventStream = "EventStream"

		// Authentication Cookie
		GenerateCookie = "GenerateCookie"
		)

		var Routes = rata.Routes{
		// Tasks
		{Path: "/v1/tasks", Method: "POST", Name: CreateTaskRoute},
		{Path: "/v1/tasks", Method: "GET", Name: TasksRoute},
		{Path: "/v1/tasks/:task_guid", Method: "GET", Name: GetTaskRoute},
		{Path: "/v1/tasks/:task_guid", Method: "DELETE", Name: DeleteTaskRoute},
		{Path: "/v1/tasks/:task_guid/cancel", Method: "POST", Name: CancelTaskRoute},

		// DesiredLRPs
		{Path: "/v1/desired_lrps", Method: "GET", Name: DesiredLRPsRoute},
		{Path: "/v1/desired_lrps", Method: "POST", Name: CreateDesiredLRPRoute},
		{Path: "/v1/desired_lrps/:process_guid", Method: "GET", Name: GetDesiredLRPRoute},
		{Path: "/v1/desired_lrps/:process_guid", Method: "PUT", Name: UpdateDesiredLRPRoute},
		{Path: "/v1/desired_lrps/:process_guid", Method: "DELETE", Name: DeleteDesiredLRPRoute},

		// ActualLRPs
		{Path: "/v1/actual_lrps", Method: "GET", Name: ActualLRPsRoute},
		{Path: "/v1/actual_lrps/:process_guid", Method: "GET", Name: ActualLRPsByProcessGuidRoute},
		{Path: "/v1/actual_lrps/:process_guid/index/:index", Method: "GET", Name: ActualLRPByProcessGuidAndIndexRoute},
		{Path: "/v1/actual_lrps/:process_guid/index/:index", Method: "DELETE", Name: KillActualLRPByProcessGuidAndIndexRoute},

		// Cells
		{Path: "/v1/cells", Method: "GET", Name: CellsRoute},

		// Domains
		{Path: "/v1/domains/:domain", Method: "PUT", Name: UpsertDomainRoute},
		{Path: "/v1/domains", Method: "GET", Name: DomainsRoute},

		// Event Streaming
		{Path: "/v1/events", Method: "GET", Name: EventStream},

		// Authentication Cookie
		{Path: "/v1/auth_cookie", Method: "POST", Name: GenerateCookie},
		}

接下来所有的操作都会通过客户端来处理，这里的处理是交给了很多不同类别的handler，我们看其中1个：</br>
--->handlers/actual_lrp_handlers.go</br>
这些handler都涉及到一个比较给力名字的客户端：bbs bbs.Client</br>
我们看到这个函数func (h *ActualLRPHandler) GetAllByProcessGuid(w http.ResponseWriter, req *http.Request) 其它大同小异</br>
	
1.首先获取process_guid </br>
processGuid := req.FormValue(":process_guid")</br>

2.开始调取所有actualLRPs</br>
actualLRPGroupsByIndex, err := h.bbs.ActualLRPGroupsByProcessGuid(processGuid)</br>
--->跟到ActualLRPGroupsByProcessGuid里面看一下：https://github.com/cloudfoundry-incubator/bbs/blob/373fe5f7af9b9bfdb311a6f88bf8ba8c06555d86/handlers/actual_lrp_handlers.go#L38</br>
		
		func (h *ActualLRPHandler) ActualLRPGroupsByProcessGuid(w http.ResponseWriter, req *http.Request) {
			logger := h.logger.Session("actual-lrp-groups-by-process-guid")

			request := &models.ActualLRPGroupsByProcessGuidRequest{}
			response := &models.ActualLRPGroupsResponse{}

			response.Error = parseRequest(logger, req, request)
			if response.Error == nil {
				response.ActualLrpGroups, response.Error = h.db.ActualLRPGroupsByProcessGuid(logger, request.ProcessGuid)
			}

			writeResponse(w, response)
		}
		
--->这里还是看不到，继续跟https://github.com/cloudfoundry-incubator/bbs/blob/cd77526f2067736f163a784c6d1a351162ddf855/db/etcd/actual_lrp_db.go#L70</br>
看到一个熟悉的东西，**etcd**,没错就是通过fetch etcd里面的node还获取基本的元数据的，我们进去看一下：</br>

		func (db *ETCDDB) ActualLRPGroupsByProcessGuid(logger lager.Logger, processGuid string) ([]*models.ActualLRPGroup, *models.Error) {
			node, bbsErr := db.fetchRecursiveRaw(logger, ActualLRPProcessDir(processGuid))
			if bbsErr.Equal(models.ErrResourceNotFound) {
				return []*models.ActualLRPGroup{}, nil
			}
			if bbsErr != nil {
				return nil, bbsErr
			}
			if node.Nodes.Len() == 0 {
			return []*models.ActualLRPGroup{}, nil
			}

			return parseActualLRPGroups(logger, node, models.ActualLRPFilter{})
		}
		
--->最终我们将代码定位到了etcd和核心区继而看一下fetchRecursiveRaw</br>

		https://github.com/cloudfoundry-incubator/bbs/blob/a504a4e4bdfdeb1363a75a380aaa422bf8799f87/db/etcd/etcd_db.go
		func (db *ETCDDB) fetchRecursiveRaw(logger lager.Logger, key string) (*etcd.Node, *models.Error) {
			logger.Debug("fetching-recursive-from-etcd")
			response, err := db.client.Get(key, false, true)
			if err != nil {
				return nil, ErrorFromEtcdError(logger, err)
			}
			logger.Debug("succeeded-fetching-recursive-from-etcd", lager.Data{"num-nodes": response.Node.Nodes.Len()})
			return response.Node, nil
		}
		
我决定不往下跟了，因为db.client.Get(key, false, true)就是在通过process_guid来看里面的信息</br>
	
最后返回的就是这个结构体：其信息就是刚在在最前面展示的部分了:</br>
etcd里存储的路径为：</br>
/v1/actual/c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6</br>
我们进去看看：</br>

		/v1/actual/c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6/0
		/v1/actual/c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6/1
		
发现有两个索引，说明这里面有两个实例：</br>

		/v1/actual/c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6/0/instance
	
进到这个实例里去瞧一眼：发现跟我们刚才得到的是一样的</br>

		{
		"process_guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
		"index": 0, 
		"domain": "cf-apps", 
		"instance_guid": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe", 
		"cell_id": "cell_z1-0", 
		"address": "10.244.16.138", 
		"ports": [
		{
		  "container_port": 8080, 
		  "host_port": 60000
		}, 
		{
		  "container_port": 2222, 
		  "host_port": 60001
		}
		], 
		"crash_count": 0, 
		"state": "RUNNING", 
		"since": 1441087965748636200, 
		"modification_tag": {
		"epoch": "d292b1b4-fd3e-4df1-5ea3-733e14a616ae", 
		"index": 27
		}
		}
	
DesiredLRP是这样的：/v1/desired/c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6</br>
这边是序列化的结构：</br>
--->/serialization/actual_lrps.go</br>
	
		func ActualLRPProtoToResponse(actualLRP *models.ActualLRP, evacuating bool) receptor.ActualLRPResponse {
		return receptor.ActualLRPResponse{
			ProcessGuid:     actualLRP.ProcessGuid,
			InstanceGuid:    actualLRP.InstanceGuid,
			CellID:          actualLRP.CellId,
			Domain:          actualLRP.Domain,
			Index:           int(actualLRP.Index),
			Address:         actualLRP.Address,
			Ports:           PortMappingFromProto(actualLRP.Ports),
			State:           actualLRPProtoStateToResponseState(actualLRP.State),
			PlacementError:  actualLRP.PlacementError,
			Since:           actualLRP.Since,
			CrashCount:      int(actualLRP.CrashCount),
			CrashReason:     actualLRP.CrashReason,
			Evacuating:      evacuating,
			ModificationTag: actualLRPProtoModificationTagToResponseModificationTag(actualLRP.ModificationTag),
		}
		}
	
### 小插曲，进到安全加强版的etcd里非常不易：

		CERT_DIR=/var/vcap/jobs/etcd/config/certs
		ca_cert_file=${CERT_DIR}/server-ca.crt
		server_cert_file=${CERT_DIR}/server.crt
		server_key_file=${CERT_DIR}/server.key
		client_cert_file=${CERT_DIR}/client.crt
		client_key_file=${CERT_DIR}/client.key
		
		etcdctl_sec_flags=" \
		-ca-file=${ca_cert_file} \
		-cert-file=${client_cert_file} \
		-key-file=${client_key_file}"
		
		etcd_sec_flags=" \
		--client-cert-auth \
		--trusted-ca-file ${ca_cert_file} \
		--cert-file ${server_cert_file} \
		--key-file ${server_key_file}"
		
		peer_ca_cert_file=${CERT_DIR}/peer-ca.crt
		peer_cert_file=${CERT_DIR}/peer.crt
		peer_key_file=${CERT_DIR}/peer.key

		etcd_peer_sec_flags=" \
		--peer-client-cert-auth \
		--peer-trusted-ca-file ${peer_ca_cert_file} \
		--peer-cert-file ${peer_cert_file} \
		--peer-key-file ${peer_key_file}"

		etcd_sec_flags="${etcd_sec_flags} ${etcd_peer_sec_flags}"
  
这时我们再执行：
/var/vcap/packages/etcd/etcdctl ${etcdctl_sec_flags} -C 'https://database-z1-0.etcd.service.cf.internal:4001' ls