# cc_bridge ����(����)
���濪ʼ��cc_bridge���ֿ�ʼ��ҪԴ�������������ڷ������⣬����ڸ� <br />

cc_bridgeһ����4��������
-----------------------------------
* stager     ����cc�������ı����������,����һ����һЩtasks���������ڵ����񣬱��룬��������أ��ϴ��ȵ�
* nsync      ����LRP����,����֤��˵�Ӧ���ܹ�������ʱ������У�����һ�����Ƕ��ڵ���CCͨ�ţ���ȷ����ЩӦ����diego�ﶼ���µ�
* tps        ������������ֱ�Ӻ�CCͨ��ͬʱ��ȡ��ǰ�������е�LRPs���¼�������crash�ȵ�
* fileServer �������ʵ��ŵĶ���һЩ���ںͱ�������ص�tar��������docker_app_lifecycle��windows_app_lifecycle��buildpack_app_lifecycle

### �������ĸ��������˵��һ��ʲô��tasks,ʲô��LRPs
Tasks: ָһ�ζ����񣬿���˵��һ��ִ�����̣�����������pushһ��Ӧ�õ�ʱ�򣬻�������appFiles������lifecycle,ִ��lifecycle����������droplets,�ϴ�droplet��һϵ�ж��������ڵ�
�����������ǿ���ͳ������Ϊtasks </br>
LRPs�� ָ���е�ʵ�����ھ���һЩ�е�tasks������ɹ���diego�ᰴ����ǰӦ�����ƶ��������嵥,���ڵ�����ĳ��APP������LRPs�ַ�ΪDesiredLRPs��ActualLRPs��DesiredLRP��ָӦ��
��ִ���嵥�������ָ��rootfs,ports,��ȫ��,��Դ����,���ļ�ϵͳ,��Ҫ��ʵ�������ȵ� ; ��acutalLRPs��ָ���е�ʵ��������һ��Ӧ���е�һ������ʵ���ٱ�����ʱ,������Ӧ������
ip,�����䵽���Ǹ�cells��ȵ�</br>

### Stager�������
�ȿ���������������

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
		
��Ϊstager������������������������ģ�����������Կ�������lifecycle,˵��֧��buildpack��docker��windows </br>

### ������</br>
    stager������cc�˷����ı�������</br>
	
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
	
�������Զ�������ʽ�ύ����һ���<strong>Receptor</strong>�齨ִ�У�����ִ�н�����أ�ע��������Ϊһ�ζ�������ִ�б��룩</br>
��˷�Ϊbuildpack_app_lifecycle��docker_app_lifecycle ��������ʵ������ƴ���ַ���������ִ���ǽ���Receptor��ִ�ж��� </br>
	
buildpack_app_lifecycle:���ᴥ��һ�¶���
1.Download app package
2.Download builder��docker or buildpack��
3.Download buildpacks
4.Download buildpack artifacts cache
5.Run Builder
6.Upload Droplet
7.Upload Buildpack Artifacts Cache
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
	
docker_app_lifecycle:���ᴥ�����¶�����</br>
1.Download builder��������docker image cache�Ƿ񻺴棩</br>
registryServices, err := getDockerRegistryServices(backend.config.ConsulCluster, backend.logger)</br>
����docker image�Ƿ��Ѿ�ע�ᵽ��Consul,���û�����ύ����docker image������</br>
registryRules := addDockerRegistryRules(request.EgressRules, registryServices)</br>
docker register�ĵ�ַ</br>
registryIPs := strings.Join(buildDockerRegistryAddresses(registryServices), ",")</br>
����ע��docker�ĵ�ַ</br>
runActionArguments, err = addDockerCachingArguments(runActionArguments, registryIPs, backend.config.InsecureDockerRegistry, host, port, lifecycleData)</br>
���е������ֶζ�������ɣ��ύrun action�����Receptor</br>
2.Run builder
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
	
��������һ�´�������docker image����ע����ʲô����������docker image�ľ�����Ϣ��diego_docker_cache�</br>
��һ����ʵ������builder��������docker image ������ </br>

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
	
	