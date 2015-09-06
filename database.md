# database 部分(初版)
下面开始从database部分开始简要源码分析，如果后期发现问题，择后在改 <br />

database一共有2个组件组成
-----------------------------------
* etcd 这个是database的存储介质
* bbs BBS在整套diego里充当的就是etcd的数据提供了原始数据的存储，并提供了一套restful内部使用的API

### bbs组件分析
先看它的启动参数：</br>

		/var/vcap/packages/bbs/bin/bbs ${etcd_sec_flags} \
		-address=0.0.0.0:8889 \
		-auctioneerAddress=http://auctioneer.service.cf.internal:9016 \
		-debugAddr=0.0.0.0:17017 \
		-consulCluster=http://127.0.0.1:8500 \
		-etcdCluster="https://etcd.service.cf.internal:4001" \
		-logLevel=info
		
因为是etcd的解耦，所以涉及的操作也越来越底层

		var Routes = rata.Routes{
			// Domains
			{Path: "/v1/domains/list", Method: "POST", Name: DomainsRoute},
			{Path: "/v1/domains/upsert", Method: "POST", Name: UpsertDomainRoute},

			// Actual LRPs
			{Path: "/v1/actual_lrp_groups/list", Method: "POST", Name: ActualLRPGroupsRoute},
			{Path: "/v1/actual_lrp_groups/list_by_process_guid", Method: "POST", Name: ActualLRPGroupsByProcessGuidRoute},
			{Path: "/v1/actual_lrp_groups/get_by_process_guid_and_index", Method: "POST", Name: ActualLRPGroupByProcessGuidAndIndexRoute},

			// Actual LRP Lifecycle
			{Path: "/v1/actual_lrps/claim", Method: "POST", Name: ClaimActualLRPRoute},
			{Path: "/v1/actual_lrps/start", Method: "POST", Name: StartActualLRPRoute},
			{Path: "/v1/actual_lrps/crash", Method: "POST", Name: CrashActualLRPRoute},
			{Path: "/v1/actual_lrps/fail", Method: "POST", Name: FailActualLRPRoute},
			{Path: "/v1/actual_lrps/remove", Method: "POST", Name: RemoveActualLRPRoute},
			{Path: "/v1/actual_lrps/retire", Method: "POST", Name: RetireActualLRPRoute},

			// Evacuation
			{Path: "/v1/actual_lrps/remove_evacuating", Method: "POST", Name: RemoveEvacuatingActualLRPRoute},
			{Path: "/v1/actual_lrps/evacuate_claimed", Method: "POST", Name: EvacuateClaimedActualLRPRoute},
			{Path: "/v1/actual_lrps/evacuate_crashed", Method: "POST", Name: EvacuateCrashedActualLRPRoute},
			{Path: "/v1/actual_lrps/evacuate_stopped", Method: "POST", Name: EvacuateStoppedActualLRPRoute},
			{Path: "/v1/actual_lrps/evacuate_running", Method: "POST", Name: EvacuateRunningActualLRPRoute},

			// Desired LRPs
			{Path: "/v1/desired_lrps/list", Method: "POST", Name: DesiredLRPsRoute},
			{Path: "/v1/desired_lrps/get_by_process_guid", Method: "POST", Name: DesiredLRPByProcessGuidRoute},

			// Desire LPR Lifecycle
			{Path: "/v1/desired_lrp/desire", Method: "POST", Name: DesireDesiredLRPRoute},
			{Path: "/v1/desired_lrp/update", Method: "POST", Name: UpdateDesiredLRPRoute},
			{Path: "/v1/desired_lrp/remove", Method: "POST", Name: RemoveDesiredLRPRoute},

			// LRP Convergence
			{Path: "/v1/lrps/converge", Method: "POST", Name: ConvergeLRPsRoute},

			// Tasks
			{Path: "/v1/tasks/list", Method: "POST", Name: TasksRoute},
			{Path: "/v1/tasks/get_by_task_guid", Method: "GET", Name: TaskByGuidRoute},

			// Task Lifecycle
			{Path: "/v1/tasks/desire", Method: "POST", Name: DesireTaskRoute},
			{Path: "/v1/tasks/start", Method: "POST", Name: StartTaskRoute},
			{Path: "/v1/tasks/cancel", Method: "POST", Name: CancelTaskRoute},
			{Path: "/v1/tasks/fail", Method: "POST", Name: FailTaskRoute},
			{Path: "/v1/tasks/complete", Method: "POST", Name: CompleteTaskRoute},
			{Path: "/v1/tasks/resolving", Method: "POST", Name: ResolvingTaskRoute},
			{Path: "/v1/tasks/delete", Method: "POST", Name: DeleteTaskRoute},

			// Task Convergence
			{Path: "/v1/tasks/converge", Method: "POST", Name: ConvergeTasksRoute},

			// Event Streaming
			{Path: "/v1/events", Method: "GET", Name: EventStreamRoute},
		}
		
在receptor那里已经分析过了他的主要逻辑，主要是两个目录：</br>
handlers和db，但是db里有一部分是接口，真正的实现在db/etcd里，所有的请求都会在这里hold住</br>

etcd里的node里存储的有：</br>

		/v1/desired
		/v1/actual
		/v1/domain
		/v1/task

这里要注意的是针对于事件类型的，会用etcd的watcher来监听：</br>

		/db/etcd/event_db.go

		func (db *ETCDDB) WatchForActualLRPChanges(logger lager.Logger,
			created func(*models.ActualLRPGroup),
			changed func(*models.ActualLRPChange),
			deleted func(*models.ActualLRPGroup),
		) (chan<- bool, <-chan error)
		
		//这个函数会监听actualLRP节点的前后变化：
		events, stop, err := db.watch(ActualLRPSchemaRoot) 
		
		//然后开启一个goroutine:遍历所有的events
		case event.Node != nil && event.PrevNode == nil:新增节点
		case event.Node != nil && event.PrevNode != nil:更新节点
		case event.PrevNode != nil && event.Node == nil:删除节点

这里的ActualLRP node有个TTL的概念，假如发现某个节点即将过期，针对以上三种情况就会做出三种处理逻辑，WatchForDesiredLRPChanges没有这个概念</br>

		evacuating := isEvacuatingActualLRPNode(event.PrevNode)
		actualLRPGroup := &models.ActualLRPGroup{}
		if evacuating {
			actualLRPGroup.Evacuating = &actualLRP
		} else {
			actualLRPGroup.Instance = &actualLRP
		}

这里不是固定的，更新的时候可能既要考虑前驱节点，又要考虑后继节点：</br>

		evacuating := isEvacuatingActualLRPNode(event.Node)
		beforeGroup := &models.ActualLRPGroup{}
		afterGroup := &models.ActualLRPGroup{}
		if evacuating {
			afterGroup.Evacuating = &after
			beforeGroup.Evacuating = &before
		} else {
			afterGroup.Instance = &after
			beforeGroup.Instance = &before
		}

跟一下这个isEvacuatingActualLRPNode函数：</br>
https://github.com/cloudfoundry-incubator/runtime-schema/blob/ba8fb9905c07f6f0598e4f9db06368374064ae67/bbs/lrp_bbs/actual_group_getters.go#L104</br>

		func isEvacuatingActualLRPNode(node storeadapter.StoreNode) bool {
			return path.Base(node.Key) == shared.ActualLRPEvacuatingKey
		}

		const ActualLRPEvacuatingKey = "evacuating"

分析到这就不用多说了，遇到它就必须先把节点调整一遍。