# Brains 部分(初版)
下面开始从brain开始简要源码分析，如果后期发现问题，择后在改 <br />

Brains一共有3个组件组成
-----------------------------------
* auctioneer 核心计算单元调度器
* converger  直接跟BBS通信，译为收敛者，意思是掌管tasks和lrps的状态信息，容错
* runtime_metrics_server 运行tasks和lrps的统计信息

### auctioneer组件分析
先看它的启动参数：</br>

		/var/vcap/packages/auctioneer/bin/auctioneer ${etcd_sec_flags} \
		-bbsAddress=http://bbs.service.cf.internal:8889 \
		-etcdCluster="https://etcd.service.cf.internal:4001" \
		-consulCluster=http://127.0.0.1:8500 \
		-receptorTaskHandlerURL=http://receptor.service.cf.internal:1169 \
		-debugAddr=0.0.0.0:17001 \
		-listenAddr=0.0.0.0:9016 \
		-logLevel=info
		
### 分析：</br>
根据**task**和**actualLrp**实例所设置的**action**去执行相应的动作，你可以把她理解为一个执行器的代表，
这里又要引出一个组件：**auction**组件，action通过检查http来进行通讯，它是auctioneer和cell Rep的沟通桥梁，
提供锁维持机制用来当控制只有一个auctioneer在某一时间操作actions，保持事务性。</br>

这里只用看一个地方就行了：</br>
https://github.com/cloudfoundry-incubator/auctioneer/blob/4d3c25962b9a55a3fcce6c7962ad32b346346d52/auctionrunnerdelegate/auction_runner_delegate.go#L31</br>

		func (a *AuctionRunnerDelegate) FetchCellReps() (map[string]auctiontypes.CellRep, error) {
			cells, err := a.legacyBBS.Cells()
			cellReps := map[string]auctiontypes.CellRep{}
			if err != nil {
				return cellReps, err
			}

			for _, cell := range cells {
				cellReps[cell.CellID] = auction_http_client.New(a.client, cell.CellID, cell.RepAddress, a.logger.Session(cell.RepAddress))
			}

			return cellReps, nil
		}
		
可以看到cell的一些描述信息：容量和rootfs_providers</br>

		[
		  {
			"cell_id": "cell_z1-0", 
			"zone": "z1", 
			"capacity": {
			  "memory_mb": 5968, 
			  "disk_mb": 79252, 
			  "containers": 256
			}, 
			"rootfs_providers": {
			  "docker": [ ], 
			  "preloaded": [
				"cflinuxfs2"
			  ]
			}
		  }
		]
		
跟一下auction_http_client这个方法，结果就是掉进了auction里，先看一下action的路由规则:</br>

		const (
			State   = "STATE"
			Perform = "PERFORM"

			Sim_Reset = "RESET"
		)

		var Routes = rata.Routes{
			{Path: "/state", Method: "GET", Name: State},
			{Path: "/work", Method: "POST", Name: Perform},

			{Path: "/sim/reset", Method: "POST", Name: Sim_Reset},
		}
		
然后再定位到https://github.com/cloudfoundry-incubator/auction/blob/master/communication/http/auction_http_client/auction_http_client.go#L28</br>

		func New(client *http.Client, repGuid string, address string, logger lager.Logger) *AuctionHTTPClient {
			return &AuctionHTTPClient{
				client:           client,
				repGuid:          repGuid,
				address:          address,
				requestGenerator: rata.NewRequestGenerator(address, routes.Routes),
				logger:           logger,
			}
		}
		
至于这个auction_http_client.go</br>
定义了三个actionHttpClient方法：</br>

		State() (auctiontypes.CellState, error),
		Perform(work auctiontypes.Work) (auctiontypes.Work, error),
		Reset() error

其实在往内部分析就到了任务在提交过来后，如何分配到cells，这里涉及到一些资源调度算法：</br>
这里有个有意思的称呼 cf的人管他叫拍卖，谁赢了，谁就拿到LRP的执行权，这里auction会在rep端响应。</br>
/auctionrunner/scheduler.go</br>
有这么一段注释：</br>
/*
Schedule takes in a set of job requests (LRP start auctions and task starts) and
assigns the work to available cells according to the diego scoring algorithm. The
scheduler is single-threaded.  It determines scheduling of jobs one at a time so
that each calculation reflects available resources correctly.  It commits the
work in batches at the end, for better network performance.  Schedule returns
AuctionResults, indicating the success or failure of each requested job.
*/</br>

他的意思是说，cf会采用diego的评分算法，将这些任务或者lrp分配给其中的某个cell，调度器是单线程的，而且是在一次调度jobs就将返回正确可用的资源。为了避免损耗网络，它会分批次的提交
对于每次请求，要么成功要么失败。</br>

在进入算法前，先看一个结构：</br>
https://github.com/cloudfoundry-incubator/auction/blob/master/auctiontypes/types.go#L61</br>

		type LRPAuction struct {
			DesiredLRP *models.DesiredLRP
			Index      int
			AuctionRecord
		}

		type AuctionRecord struct {
			Winner   string
			Attempts int

			QueueTime    time.Time
			WaitDuration time.Duration

			PlacementError string
		}
		
DesiredLRP：</br>
https://github.com/cloudfoundry-incubator/bbs/blob/master/models/desired_lrp.go#L51</br>

举个例子:</br>

		var lrpB = &models.DesiredLRP{
			ProcessGuid: "process-guid-b",
			Instances:   2,
			Domain:      domainB,
		}

好了，预备工作完成了，现在进到算法流程：</br>
https://github.com/cloudfoundry-incubator/auction/blob/master/auctionrunner/scheduler.go#L191</br>

只分析一个LRP的调度算法：</br>

		func (s *Scheduler) scheduleLRPAuction(lrpAuction auctiontypes.LRPAuction) (auctiontypes.LRPAuction, error) {
			var winnerCell *Cell
			winnerScore := 1e20

			zones := accumulateZonesByInstances(s.zones, lrpAuction)

			filteredZones := filterZonesByRootFS(zones, lrpAuction.DesiredLRP.RootFs)

			if len(filteredZones) == 0 {
				return auctiontypes.LRPAuction{}, auctiontypes.ErrorCellMismatch
			}

			sortedZones := sortZonesByInstances(filteredZones)

			for zoneIndex, lrpByZone := range sortedZones {
				for _, cell := range lrpByZone.zone {
					score, err := cell.ScoreForLRPAuction(lrpAuction)
					if err != nil {
						continue
					}

					if score < winnerScore {
						winnerScore = score
						winnerCell = cell
					}
				}

				if zoneIndex+1 < len(sortedZones) &&
					lrpByZone.instances == sortedZones[zoneIndex+1].instances {
					continue
				}

				if winnerCell != nil {
					break
				}
			}

			if winnerCell == nil {
				return auctiontypes.LRPAuction{}, auctiontypes.ErrorInsufficientResources
			}

			err := winnerCell.ReserveLRP(lrpAuction)
			if err != nil {
				return auctiontypes.LRPAuction{}, err
			}

			lrpAuction.Winner = winnerCell.Guid
			return lrpAuction, nil
		}

### 流程
1.首先给了一个超大的分数：1e20 没人能超过它的</br>

2.看到accumulateZonesByInstances这个函数：https://github.com/cloudfoundry-incubator/auction/blob/29173639425e010af0e0943ee6c1105a11498062/auctionrunner/zone_sorter.go#L22</br>
		
		func accumulateZonesByInstances(zones map[string]Zone, lrpAuction auctiontypes.LRPAuction) []lrpByZone {
			lrpZones := []lrpByZone{}

			for _, zone := range zones {
				instances := 0
				for _, cell := range zone {
					for _, lrp := range cell.state.LRPs {
						if lrp.ProcessGuid == lrpAuction.DesiredLRP.ProcessGuid {
							instances++
						}
					}
				}
				lrpZones = append(lrpZones, lrpByZone{zone, instances})
			}
			return lrpZones
		}
		
统计lrpAuction里DesiredLRP中的进程ID是否在所有zone中每个cell中，如果有，则统计出每个zone中存此进程的个数</br>

3.filterZonesByRootFS：https://github.com/cloudfoundry-incubator/auction/blob/29173639425e010af0e0943ee6c1105a11498062/auctionrunner/zone_sorter.go#L45</br>
		
		func filterZonesByRootFS(zones []lrpByZone, rootFS string) []lrpByZone {
			filteredZones := []lrpByZone{}

			for _, lrpZone := range zones {
				cells := lrpZone.zone.FilterCells(rootFS)
				if len(cells) > 0 {
					filteredZone := lrpByZone{
						zone:      Zone(cells),
						instances: lrpZone.instances,
					}
					filteredZones = append(filteredZones, filteredZone)
				}
			}
			return filteredZones
		}
		
https://github.com/cloudfoundry-incubator/auction/blob/fcf9393a3a76b2883ebe0eadf32cf0a06fb75195/auctionrunner/scheduler.go#L15</br>

		func (z *Zone) FilterCells(rootFS string) []*Cell {
			var cells = make([]*Cell, 0, len(*z))

			for _, cell := range *z {
				if cell.MatchRootFS(rootFS) {
					cells = append(cells, cell)
				}
			}

			return cells
		}
		
匹配rootfs，一般是docker和diego系统自带的rootfs</br>

4.sortedZones := sortZonesByInstances(filteredZones) 按照zones里存在实例个数的多少来排序</br>

https://github.com/cloudfoundry-incubator/auction/blob/29173639425e010af0e0943ee6c1105a11498062/auctionrunner/zone_sorter.go#L39

		func sortZonesByInstances(zones []lrpByZone) []lrpByZone {
			sorter := zoneSorterByInstances{zones: zones}
			sort.Sort(sorter)
			return sorter.zones
		}
		
5.接下来遍历这些zones，然后通过lrpAuction来算的分数：</br>
https://github.com/cloudfoundry-incubator/auction/blob/fcf9393a3a76b2883ebe0eadf32cf0a06fb75195/auctionrunner/cell.go#L28</br>
		
		func (c *Cell) ScoreForLRPAuction(lrpAuction auctiontypes.LRPAuction) (float64, error) {
			err := c.canHandleLRPAuction(lrpAuction)
			if err != nil {
				return 0, err
			}

			numberOfInstancesWithMatchingProcessGuid := 0
			for _, lrp := range c.state.LRPs {
				if lrp.ProcessGuid == lrpAuction.DesiredLRP.ProcessGuid {
					numberOfInstancesWithMatchingProcessGuid++
				}
			}

			remainingResources := c.state.AvailableResources
			remainingResources.MemoryMB -= int(lrpAuction.DesiredLRP.MemoryMb)
			remainingResources.DiskMB -= int(lrpAuction.DesiredLRP.DiskMb)
			remainingResources.Containers -= 1

			resourceScore := c.computeScore(remainingResources, numberOfInstancesWithMatchingProcessGuid)

			return resourceScore, nil
		}
		
根据DesiredLRP内预设的内存和磁盘分配，c.state.AvailableResources给出一个现有资源</br>
然后就开始减值：MemoryMB，DiskMB，Containers</br>
**然后进入算法核心：**computeScore</br>
https://github.com/cloudfoundry-incubator/auction/blob/fcf9393a3a76b2883ebe0eadf32cf0a06fb75195/auctionrunner/cell.go#L159</br>

		func (c *Cell) computeScore(remainingResources auctiontypes.Resources, numInstances int) float64 {
			fractionUsedMemory := 1.0 - float64(remainingResources.MemoryMB)/float64(c.state.TotalResources.MemoryMB)
			fractionUsedDisk := 1.0 - float64(remainingResources.DiskMB)/float64(c.state.TotalResources.DiskMB)
			fractionUsedContainers := 1.0 - float64(remainingResources.Containers)/float64(c.state.TotalResources.Containers)

			resourceScore := (fractionUsedMemory + fractionUsedDisk + fractionUsedContainers) / 3.0
			resourceScore += float64(numInstances)

			return resourceScore
		}
		
6.如果一切都OK，最后将LRP的action存储起来</br>

		func (c *Cell) ReserveLRP(lrpAuction auctiontypes.LRPAuction) error {
			err := c.canHandleLRPAuction(lrpAuction)
			if err != nil {
				return err
			}

			c.state.LRPs = append(c.state.LRPs, auctiontypes.LRP{
				ProcessGuid: lrpAuction.DesiredLRP.ProcessGuid,
				Index:       lrpAuction.Index,
				MemoryMB:    int(lrpAuction.DesiredLRP.MemoryMb),
				DiskMB:      int(lrpAuction.DesiredLRP.DiskMb),
			})

			c.state.AvailableResources.MemoryMB -= int(lrpAuction.DesiredLRP.MemoryMb)
			c.state.AvailableResources.DiskMB -= int(lrpAuction.DesiredLRP.DiskMb)
			c.state.AvailableResources.Containers -= 1

			c.workToCommit.LRPs = append(c.workToCommit.LRPs, lrpAuction)

			return nil
		}

算法里涉及到一个模拟测试框架：http://onsi.github.io/ginkgo/ 银杏BDD-style测试框架所有的代码都在</br>
simulation里，感兴趣的可以去看看，都是一些测试报告</br>

### converger组件分析
先看它的启动参数：</br>

		/var/vcap/packages/converger/bin/converger ${etcd_sec_flags} \
		-etcdCluster="https://etcd.service.cf.internal:4001" \
		-bbsAddress=http://bbs.service.cf.internal:8889 \
		-consulCluster=http://127.0.0.1:8500 \
		-receptorTaskHandlerURL=http://receptor.service.cf.internal:1169 \
		-debugAddr=0.0.0.0:17002 \
		-convergeRepeatInterval=30s \
		-kickTaskDuration=30s \
		-expireCompletedTaskDuration=120s \
		-expirePendingTaskDuration=1800s \
		-logLevel=debug
		
说明：中文意译为收敛者，官方给了4个功能说明：</br>

* 维护BBS锁，已确保这个组件的性能，特别提到收敛必须是等幂的，意思就是，如果有错误发生也不会影响到其状态</br>
* 使用收敛方法，确保bbs内事务的一致性，还有Tasks和LRPs的容错性</br>
* 当收敛LRPs的时候，收敛者需要确定或者说协商，DesiredLRP和ActualLRP的状态。</br>
这里需要注意，如果实例丢失，auction将会被发送，这意味着它会尝试执行丢失的动作
假如它是一个临时的实例，那么组件将会发送一个停止消息给cell主机的rep组件。
* 另外，收敛者组件还会观察一些潜在的丢失的消息，比如一个一直处于pending状态的任务，这时候对于auction的请求有可能永远不会被发送给auctioneer，所以要干掉它。</br>

这个组件给人的映像是还没有彻底完工，先看下它的主要代码：</br>
converger_process/converger_process.go</br>
进到run方法，看到：</br>

		convergeTimer := c.clock.NewTimer(c.convergeRepeatInterval)
		cellDisappeared := make(chan services_bbs.CellEvent)
		然后goroutine迭代一个匿名函数，其中一个通道：
		case event := <-events:
			switch event.EventType() {
			case services_bbs.CellDisappeared:
				c.logger.Info("received-cell-disappeared-event", lager.Data{"cell-id": event.CellIDs()})
				select {
				case cellDisappeared <- event:
				case <-done:
					return
				}
			}
			
这个是观察cells突然消失了，也就是从bbs中，然后就开始处理</br>

		case <-cellDisappeared:
					c.converge()
			
进到这个converge方法</br>

		func (c *ConvergerProcess) converge() {
			wg := sync.WaitGroup{}

			wg.Add(1)
			go func() {
				defer wg.Done()
				c.bbsClient.ConvergeTasks(
					c.kickTaskDuration,
					c.expirePendingTaskDuration,
					c.expireCompletedTaskDuration,
				)
			}()

			wg.Add(1)
			go func() {
				defer wg.Done()
				c.bbsClient.ConvergeLRPs()
			}()

			wg.Wait()
		}

可以看到一个是Tasks，一个是LRPs，因为cells丢了，所以要重新聚合</br>

https://github.com/cloudfoundry-incubator/bbs/blob/d643994d154d7f297da48a887f0df857a5899780/client.go#L484</br>

		func (c *client) ConvergeTasks(kickTaskDuration, expirePendingTaskDuration, expireCompletedTaskDuration time.Duration) error {
			request := &models.ConvergeTasksRequest{
				KickTaskDuration:            kickTaskDuration.Nanoseconds(),
				ExpirePendingTaskDuration:   expirePendingTaskDuration.Nanoseconds(),
				ExpireCompletedTaskDuration: expireCompletedTaskDuration.Nanoseconds(),
			}
			response := models.ConvergeTasksResponse{}
			route := ConvergeTasksRoute
			err := c.doRequest(route, nil, nil, request, &response)
			if err != nil {
				return err
			}
			return response.Error.ToError()
		}

这里涉及到两个路由：</br>

		ConvergeLRPsRoute = "ConvergeLRPs"
		ConvergeTasksRoute = "ConvergeTasks"
		{Path: "/v1/lrps/converge", Method: "POST", Name: ConvergeLRPsRoute},
		{Path: "/v1/tasks/converge", Method: "POST", Name: ConvergeTasksRoute},

https://github.com/cloudfoundry-incubator/bbs/blob/d643994d154d7f297da48a887f0df857a5899780/handlers/task_handlers.go#L169</br>

		func (h *TaskHandler) ConvergeTasks(w http.ResponseWriter, req *http.Request) {
			var err error
			logger := h.logger.Session("converge-tasks")

			request := &models.ConvergeTasksRequest{}
			response := &models.ConvergeTasksResponse{}

			err = parseRequest(logger, req, request)
			if err == nil {
				h.db.ConvergeTasks(
					logger,
					time.Duration(request.KickTaskDuration),
					time.Duration(request.ExpirePendingTaskDuration),
					time.Duration(request.ExpireCompletedTaskDuration),
				)
			}

			response.Error = models.ConvertError(err)
			writeResponse(w, response)
		}

之后来到bbs/db/etcd/task_convergence.go 最终操作还是etcd</br>
这块代码很复杂，因为涉及了一些<strong>CAS</strong>原则：</br>
https://github.com/cloudfoundry-incubator/bbs/blob/d643994d154d7f297da48a887f0df857a5899780/db/etcd/task_convergence.go#L27</br>

1.加载cells:</br>

		cellSet, modelErr := cellsLoader.Cells()

2.完成的task的结构</br>

		tasksToComplete := []*models.Task{}
		scheduleForCompletion := func(task *models.Task) {
			if task.CompletionCallbackUrl == "" {
				return
			}
			tasksToComplete = append(tasksToComplete, task)
		}
		
3.然后对tasksCAS 也就是一致性判断，需要做CAS的任务，也就是重建索引</br>

		tasksToCAS := []compareAndSwappableTask{}
		scheduleForCASByIndex := func(index uint64, newTask *models.Task) {
			tasksToCAS = append(tasksToCAS, compareAndSwappableTask{
				OldIndex: index,
				NewTask:  newTask,
			})
		}
		
4.遍历tasks nodes节点的状态：</br>
case models.Task_Pending</br>
分两种情况，第一种是，假如任务需要被标记为失败（也就是在有限的时间没有启动,过期了），则CAS的修改索引
第二种是如果任务一直在请求auction时被pending，则将这个任务交给auction
在选择之前有一句这个要注意：当一个任务是持续被更新状态的时候</br>

		shouldKickTask := db.durationSinceTaskUpdated(task) >= kickTaskDuration

		shouldMarkAsFailed := db.durationSinceTaskCreated(task) >= expirePendingTaskDuration

		if shouldMarkAsFailed {
			logError(task, "failed-to-start-in-time")
			db.markTaskFailed(task, "not started within time limit")
			scheduleForCASByIndex(node.ModifiedIndex, task)
			tasksKicked++
		} else if shouldKickTask {
			logger.Info("requesting-auction-for-pending-task", lager.Data{"task-guid": task.TaskGuid})
			tasksToAuction = append(tasksToAuction, task)
			tasksKicked++
		}

		case models.Task_Running:
		//因为是cells丢失，任务需要迁移，所以只有一种情况
		cellIsAlive := cellSet.HasCellID(task.CellId)
		if !cellIsAlive {
			logError(task, "cell-disappeared")
			db.markTaskFailed(task, "cell disappeared before completion")
			scheduleForCASByIndex(node.ModifiedIndex, task)
			tasksKicked++
		}
		
一般情况下，task里的cells已经无效，说明cell没有存活，则开始CAS的修改调度器中的任务node索引</br>

case models.Task_Completed:
对于一些已经完成的任务，但还没来得及清除的，也就是过期的，节点将会被删除
如果这个任务是一种持续更新状态，则将这个task放到scheduleForCompletion里处理</br>

		shouldDeleteTask := db.durationSinceTaskFirstCompleted(task) >= expireCompletedTaskDuration
		if shouldDeleteTask {
			logError(task, "failed-to-start-resolving-in-time")
			keysToDelete = append(keysToDelete, node.Key)
		} else if shouldKickTask {
			logger.Info("kicking-completed-task", lager.Data{"task-guid": task.TaskGuid})
			scheduleForCompletion(task)
			tasksKicked++
		}

case models.Task_Resolving:</br>
最后一种情况，就是存在分歧，和前一种状态不同的是，他没有被完成过，处于决策阶段：</br>
也是分两种情况，一种是需要被删除，第二种是需要被塞进scheduleForCompletion,
但在塞进这里之前，因为这个task没有完成还处于中间状态，先降到Completed里，然后再将这个node进行更新或者说移走</br>

		shouldDeleteTask := db.durationSinceTaskFirstCompleted(task) >= expireCompletedTaskDuration
		if shouldDeleteTask {
			logError(task, "failed-to-resolve-in-time")
			keysToDelete = append(keysToDelete, node.Key)
		} else if shouldKickTask {
			logger.Info("demoting-resolving-to-completed", lager.Data{"task-guid": task.TaskGuid})
			demoted := demoteToCompleted(task)
			scheduleForCASByIndex(node.ModifiedIndex, demoted)
			scheduleForCompletion(demoted)
			tasksKicked++
		}
		
看到https://github.com/cloudfoundry-incubator/bbs/blob/master/db/etcd/task_convergence.go#L213</br>

		accumulateZonesByInstances//这个函数：降到completed的状态
		httpsfunc demoteToCompleted(task *models.Task) *models.Task {
			task.State = models.Task_Completed
			return task
		}

这时候所有的状态记录都已经处理完成：</br>
running的任务会交给auctionner客户端处理：</br>
db.auctioneerClient.RequestTaskAuctions(tasksToAuction)</br>

对于tasktoCAS的任务，直接批处理了：这里的tasksToCAS在scheduleForCASByIndex</br>

		tasksKickedCounter.Add(tasksKicked)
		err := db.batchCompareAndSwapTasks(tasksToCAS, logger)
		
针对批处理，cf组织专门为其写了一个线程池的工具workpool,主要是task的node节点的时间更新，然后根据taskToCAS.OldIndex和新task的对比</br>

		for _, taskToCAS := range tasksToCAS {
			task := taskToCAS.NewTask
			task.UpdatedAt = db.clock.Now().UnixNano()
			value, err := db.serializeModel(logger, task)
			if err != nil {
				logger.Error("failed-to-marshal", err, lager.Data{
					"task-guid": task.TaskGuid,
				})
				continue
			}

			index := taskToCAS.OldIndex
			works = append(works, func() {
				_, err := db.client.CompareAndSwap(TaskSchemaPathByGuid(task.TaskGuid), value, NO_TTL, index)
				if err != nil {
					logger.Error("failed-to-compare-and-swap", err, lager.Data{
						"task-guid": task.TaskGuid,
					})
				}
			})
		}

跟一下这个CompareAndSwap，其实是一个原子操作</br>

		func (sc *storeClient) CompareAndSwap(key string, payload []byte, ttl uint64, prevIndex uint64) (*etcd.Response, error) {
			return sc.client.CompareAndSwap(key, string(payload), ttl, "", prevIndex)
		}
		
跟到go-etcd里，会发现会校之前的prevValue进行比较，这里就不深入了：</br>

		func (c *Client) RawCompareAndSwap(key string, value string, ttl uint64,
			prevValue string, prevIndex uint64) (*RawResponse, error) {
			if prevValue == "" && prevIndex == 0 {
				return nil, fmt.Errorf("You must give either prevValue or prevIndex.")
			}

			options := Options{}
			if prevValue != "" {
				options["prevValue"] = prevValue
			}
			if prevIndex != 0 {
				options["prevIndex"] = prevIndex
			}

			raw, err := c.put(key, value, ttl, options)

			if err != nil {
				return nil, err
			}

			return raw, err
		}

然后是提交完成的tasks</br>

		for _, task := range tasksToComplete {
			db.taskCompletionClient.Submit(db, task)
		}
		tasksPrunedCounter.Add(uint64(len(keysToDelete)))

最后就剩下需要清理的无效任务了</br>

		db.batchDeleteTasks(keysToDelete, logger)



