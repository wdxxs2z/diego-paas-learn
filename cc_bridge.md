# cc_bridge 部分(初版)
下面开始从cc_bridge部分开始简要源码分析，如果后期发现问题，择后在改 <br />

cc_bridge一共有4个组件组成
-----------------------------------
* stager     接收cc发过来的编译请求组件,这里一般是一些tasks短生命周期的任务，编译，打包，下载，上传等等
* nsync      处理LRP任务,够保证后端的应用能够持续长时间的运行，还有一个就是定期的于CC通信，已确保这些应用在diego里都是新的
* tps        这个组件是用来直接和CC通信同时获取当前正在运行的LRPs的事件，比如crash等等
* fileServer 此组件其实存放的都是一些关于和编译打包相关的tar包，比如docker_app_lifecycle，windows_app_lifecycle，buildpack_app_lifecycle

### 分析这四个组件，先说明一下什么是tasks,什么是LRPs
Tasks: 指一次短任务，可以说是一个执行历程，比如我们在push一个应用的时候，会有下载appFiles，下载lifecycle,执行lifecycle编译命令，打包droplets,上传droplet等一系列短生命周期的
任务，这里我们可以统称他们为tasks </br>
LRPs： 指运行的实例，在经过一些列的tasks后，如果成功，diego会按照先前应用所制定的运行清单,长期的运行某个APP，这里LRPs又分为DesiredLRPs和ActualLRPs，DesiredLRP是指应用
的执行清单，比如会指定rootfs,ports,安全组,资源限制,根文件系统,需要的实例个数等等 ; 而acutalLRPs是指运行的实例，比如一个应用中的一个运行实例再被创建时,它所对应的容器
ip,被分配到了那个cells里等等</br>

### Stager组件分析
先看它的启动参数：

		/var/vcap/packages/stager/bin/stager \
		-diegoAPIURL=http://:@receptor.service.cf.internal:8887 \
		-stagerURL=http://stager.service.cf.internal:8888 \
		-ccBaseURL=http://cloud-controller-ng.service.cf.internal:9022 \
		-ccUsername=internal_user \
		-ccPassword=internal-password \
		-skipCertVerify=true \
		-debugAddr=0.0.0.0:17011 \
		-lifecycle buildpack/cflinuxfs2:buildpack_app_lifecycle/buildpack_app_lifecycle.tgz -lifecycle buildpack/windows2012R2:windows_app_lifecycle/windows_app_lifecycle.tgz -lifecycle docker:docker_app_lifecycle/docker_app_lifecycle.tgz \
		-dockerStagingStack=cflinuxfs2 \
		-fileServerURL=http://file-server.service.cf.internal:8080 \
		-ccUploaderURL=http://cc-uploader.service.cf.internal:9090 \
		-consulCluster=http://127.0.0.1:8500 \
		-dockerRegistryAddress=docker-registry.service.cf.internal:8080 \
		-logLevel=info
		
因为stager本身就是用来处理编译任务的，所以这里可以看到三个lifecycle,说明支持buildpack、docker、windows </br>

### 分析：</br>
Stager负责处理cc端发来的编译请求
	
		stager/routes.go 
		const (
		StageRoute            = "Stage"
		StopStagingRoute      = "StopStaging"
		StagingCompletedRoute = "StagingCompleted"
		)
		
		var Routes = rata.Routes{
			{Path: "/v1/staging/:staging_guid", Method: "PUT", Name: StageRoute},
			{Path: "/v1/staging/:staging_guid", Method: "DELETE", Name: StopStagingRoute},
			{Path: "/v1/staging/:staging_guid/completed", Method: "POST", Name: StagingCompletedRoute},
		}
	
将任务以动作的形式提交到后一层的<strong>Receptor</strong>组建执行，并将执行结果返回（注意这里做为一次短任务来执行编译）</br>
后端分为<strong>buildpack_app_lifecycle</strong>和<strong>docker_app_lifecycle</strong>这两步其实都是在拼接字符串，真正执行是交给Receptor来执行动作 </br>
	
buildpack_app_lifecycle:将会触发一下动作</br>
1.Download app package</br>
2.Download builder（docker or buildpack）</br>
3.Download buildpacks</br>
4.Download buildpack artifacts cache</br>
5.Run Builder</br>
6.Upload Droplet</br>
7.Upload Buildpack Artifacts Cache</br>

		task := receptor.TaskCreateRequest{
			TaskGuid:              stagingGuid,
			Domain:                backend.config.TaskDomain,
			RootFS:                models.PreloadedRootFS(lifecycleData.Stack),
			ResultFile:            builderConfig.OutputMetadata(),
			MemoryMB:              request.MemoryMB,
			DiskMB:                request.DiskMB,
			CPUWeight:             StagingTaskCpuWeight,
			Action:                models.WrapAction(models.Timeout(models.Serial(actions...), timeout)),
			LogGuid:               request.LogGuid,
			LogSource:             TaskLogSource,
			CompletionCallbackURL: backend.config.CallbackURL(stagingGuid),
			EgressRules:           request.EgressRules,
			Annotation:            string(annotationJson),
			Privileged:            true,
			EnvironmentVariables:  []*models.EnvironmentVariable{{"LANG", DefaultLANG}},
		}
	
docker_app_lifecycle:将会触发以下动作：</br>
1.Download builder（将会检查docker image cache是否缓存）</br>
registryServices, err := getDockerRegistryServices(backend.config.ConsulCluster, backend.logger)</br>
会检查docker image是否已经注册到了Consul,如果没有则提交构建docker image的任务</br>
registryRules := addDockerRegistryRules(request.EgressRules, registryServices)</br>
docker register的地址</br>
registryIPs := strings.Join(buildDockerRegistryAddresses(registryServices), ",")</br>
构建注册docker的地址</br>
runActionArguments, err = addDockerCachingArguments(runActionArguments, registryIPs, backend.config.InsecureDockerRegistry, host, port, lifecycleData)</br>
所有的请求字段都构建完成，提交run action任务给Receptor</br>
2.Run builder</br>

		task := receptor.TaskCreateRequest{
			TaskGuid:              stagingGuid,
			ResultFile:            DockerBuilderOutputPath,
			Domain:                backend.config.TaskDomain,
			RootFS:                models.PreloadedRootFS(backend.config.DockerStagingStack),
			MemoryMB:              request.MemoryMB,
			DiskMB:                request.DiskMB,
			Action:                models.WrapAction(models.Timeout(models.Serial(actions...), dockerTimeout(request, backend.logger))),
			CompletionCallbackURL: backend.config.CallbackURL(stagingGuid),
			LogGuid:               request.LogGuid,
			LogSource:             TaskLogSource,
			Annotation:            string(annotationJson),
			EgressRules:           request.EgressRules,
			Privileged:            true,
		}
	
我们来看一下触发构建docker参数，到底注册了什么：如果熟悉docker的童鞋一眼就能看出，这个貌似是启动docker deamon的参数：</br>
这一步其实是在向builder发出启动docker并构建docker image 的命令 </br>

		func addDockerCachingArguments(args []string, registryIPs string, insecureRegistry bool, host string, port string, stagingData cc_messages.DockerStagingData) ([]string, error) {
		args = append(args, "-cacheDockerImage")

		args = append(args, "-dockerRegistryHost", host)
		args = append(args, "-dockerRegistryPort", port)

		args = append(args, "-dockerRegistryIPs", registryIPs)
		if insecureRegistry {
			args = append(args, "-insecureDockerRegistries", fmt.Sprintf("%s:%s", host, port))
		}

		if len(stagingData.DockerLoginServer) > 0 {
			args = append(args, "-dockerLoginServer", stagingData.DockerLoginServer)
		}
		if len(stagingData.DockerUser) > 0 {
			args = append(args, "-dockerUser", stagingData.DockerUser,
				"-dockerPassword", stagingData.DockerPassword,
				"-dockerEmail", stagingData.DockerEmail)
		}

		return args, nil
		}
		
### 为了不让自己偷懒，到docker_app_lifecycle里去一看究竟
https://github.com/cloudfoundry-incubator/docker_app_lifecycle </br>

1.首先检查docker是否已经启动</br>
err := waitForDocker(signals, builder.DockerDaemonTimeout)
</br>
2.如果/var/run/docker.sock正常，则运行build,这时候就会去fetch image，最终将整个镜像保存起来</br>
img, err := helpers.FetchMetadata(builder.RepoName, builder.Tag, builder.InsecureDockerRegistries, authConfig)</br>
这里面的整个操作都是在后端garden-linux里执行的，后面会分析</br>
https://github.com/cloudfoundry-incubator/docker_app_lifecycle/blob/master/helpers/helpers.go#L49
</br>
3.当镜像完全下载完成后，就可以使用luncher来把这个镜像run起来</br>
https://github.com/cloudfoundry-incubator/docker_app_lifecycle/blob/master/launcher/main.go#L20</br>
设置环境变量：</br>
		err := json.Unmarshal([]byte(os.Getenv("VCAP_APPLICATION")), &vcapAppEnv)
		if err == nil {
			vcapAppEnv["host"] = "0.0.0.0"

			vcapAppEnv["instance_id"] = os.Getenv("INSTANCE_GUID")

			port, err := strconv.Atoi(os.Getenv("PORT"))
			if err == nil {
				vcapAppEnv["port"] = port
			}

			index, err := strconv.Atoi(os.Getenv("INSTANCE_INDEX"))
			if err == nil {
				vcapAppEnv["instance_index"] = index
			}

			mungedAppEnv, err := json.Marshal(vcapAppEnv)
			if err == nil {
				os.Setenv("VCAP_APPLICATION", string(mungedAppEnv))
			}
		}
</br>
如果DockerFile中有启动命令就将其按照dockerfile里定义的entrypoint里的命令来启动</br>
		if startCommand != "" {
			syscall.Exec("/bin/sh", []string{
				"/bin/sh",
				"-c",
				startCommand,
			}, os.Environ())
		} else {
			if len(executionMetadata.Entrypoint) == 0 && len(executionMetadata.Cmd) == 0 {
				fmt.Fprintf(os.Stderr, "No start command found or specified")
				os.Exit(1)
			}

			// https://docs.docker.com/reference/builder/#entrypoint and
			// https://docs.docker.com/reference/builder/#cmd dictate how Entrypoint
			// and Cmd are treated by docker; we follow these rules here
			argv := executionMetadata.Entrypoint
			argv = append(argv, executionMetadata.Cmd...)
			syscall.Exec(argv[0], argv, os.Environ())
		}

