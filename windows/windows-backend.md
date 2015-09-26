# Windows-backend篇

由于对.net不熟悉，所以只看表面和部署，只是说明目前V3引擎能支持跑windows .net程序了。</br>

## 说明

由于目前V3版还在研发和测试中，对于一些复杂的.NET程序，个人心里还是没谱，windows大家都懂的。</br>

## 安装历程

1.下载windows2012 R2官方镜像，附加一句，这个镜像装在我的virtualBox上，配置两个网卡，一个默认net方式，一个桥接到bosh-lite虚机中</br>

		桥接到Adapt-host-2：192.168.50.4的bosh-lite虚机，设置IP为192.168.50.6，网关为192.168.50.4
		
		Net采用官方默认，保证互联网能够联通
		
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/network.JPG) <br />

2.下载官方指定初始化脚本，并执行：</br>

		https://github.com/cloudfoundry-incubator/diego-windows-msi/releases/tag/v0.694
		//power shell下执行，也可在cmd下执行
		setup.ps1
		
这里的过程就是设置防火墙规则，启动windows更新，前提是必须是正版用户，这里确实需要注意，在安装完虚机后，需要激活，您必须是**正版用户**，还有设置一大堆
.net com组件等等</br>

3.配置windows-backend安装参数</br>

		msiexec /norestart /i c:\temp\DiegoWindowsMSI.msi ^
		ADMIN_USERNAME=Administrator ^
		ADMIN_PASSWORD=a12345678A ^
		BBS_CA_FILE=c:\temp\bbs_ca.crt ^
		BBS_CLIENT_CERT_FILE=c:\temp\bbs_client.crt ^
		BBS_CLIENT_KEY_FILE=c:\temp\bbs_client.key ^
		CONSUL_IPS=10.244.0.54 ^
		CONSUL_ENCRYPT_FILE=c:\temp\consul_encrypt.key ^
		CONSUL_CA_FILE=c:\temp\consul_ca.crt ^
		CONSUL_AGENT_CERT_FILE=c:\temp\consul_agent.crt ^
		CONSUL_AGENT_KEY_FILE=c:\temp\consul_agent.key ^
		CF_ETCD_CLUSTER=http://10.244.16.130:4001 ^
		STACK=windows2012R2 ^
		REDUNDANCY_ZONE=windows ^
		LOGGREGATOR_SHARED_SECRET=loggregator-secret ^
		EXTERNAL_IP=192.168.50.6 ^
		MACHINE_NAME=WIN-6KT73FK5PFV
		
由于diego组件都是通过consul实现服务发现的，所以这里在配置认证的时候一定得注意，还有一个是BBS，设置它主要是能够通过receptor的API调用使得后端windows-garden能够持续健康运行，一样符合LRP和TASK</br>

还有是需要配置**external_ip**,这个IP是用作容器内部和外部联通的，如果整个网络不能互联，则会出现无法找到适合的CELL错误。</br>

![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/services.JPG) <br />

4.下载一个测试APP并通过命令上传到CF</br>

* 先搞定插件，不过这个在之前使用docker时已经安装过了</br>
		
		cf install-plugin -r CF-Community Diego-Beta
		cf install-plugin -r CF-Community Diego-SSH
		
* 上传应用</br>
		
		git clone https://github.com/Canopy-OCTO/cip-windows-test-app
		cd cip-windows-test-app
		cf push dotnet-test-app -s windows2012R2 -b https://github.com/ryandotsmith/null-buildpack -m 1G --no-start
		cf enable-diego dotnet-test-app
		cf disable-ssh dotnet-test-app
		cf start dotnet-test-app
		
* scale应用</br>

		cf scale dotnet-test-app -i 3
		
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/runningapp.JPG) <br />
		
5.如果成功，可以到这个windows主机上看看一些行为</br>

* 守护进程**Guard**，每启动一个实例，便会增加一个,管理员权限</br>
* **IronFrame.Host** 隔离主机，以C打头的容器用户</br>
* **launcher** 启动进程，主要是联合一些.net动态库，最终将程序run起来,以C打头的容器用户</br>
* **WebAppserver** 应用程序进程，以C打头的容器用户,以上这些进程都是实例有多少就会启动多少</br>

![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/process.JPG) <br />

## 观察容器目录结构
* 主目录</br>
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/containerdir.JPG) <br />
* 容器内部user目录</br>
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/content.JPG) <br />
* lifecycle目录</br>
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/lifecycle.JPG) <br />

可以看到几个熟悉的进程。</br>

## 应用的环境变量

curl http://dotnet-test-app.10.244.0.34.xip.io/env</br>

		{
		  "SystemDrive": "C:", 
		  "ProgramFiles(x86)": "C:\Program Files (x86)", 
		  "Path": "C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\", 
		  "ProgramW6432": "C:\Program Files", 
		  "PROCESSOR_IDENTIFIER": "Intel64 Family 6 Model 60 Stepping 3, GenuineIntel", 
		  "CF_STACK": "windows2012R2", 
		  "TMP": "C:\Windows\TEMP", 
		  "PROCESSOR_ARCHITECTURE": "AMD64", 
		  "CF_INSTANCE_PORT": "50460", 
		  "VCAP_SERVICES": "{}", 
		  "PROCESSOR_REVISION": "3c03", 
		  "TEMP": "C:\Windows\TEMP", 
		  "USERPROFILE": "C:\containerizer\858DE92B64DF613951\user\", 
		  "USERNAME": "SYSTEM", 
		  "SystemRoot": "C:\Windows", 
		  "PORT": "50460", 
		  "MEMORY_LIMIT": "1024m", 
		  "FP_NO_HOST_CHECK": "NO", 
		  "CF_INSTANCE_INDEX": "0", 
		  "CommonProgramFiles": "C:\Program Files\Common Files", 
		  "__COMPAT_LAYER": "ElevateCreateProcess", 
		  "ProgramData": "C:\ProgramData", 
		  "LANG": "en_US.UTF-8", 
		  "PATHEXT": ".COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC", 
		  "COMPUTERNAME": "WIN-6KT73FK5PFV", 
		  "ARGJSON": "["app","","{\"start_command\":\"tmp/lifecycle/WebAppServer.exe\",\"start_command_args\":[\".\"]}"]", 
		  "CF_INSTANCE_GUID": "2ab613f5-7c40-447c-540d-c269bb0f5e25", 
		  "ALLUSERSPROFILE": "C:\ProgramData", 
		  "CommonProgramW6432": "C:\Program Files\Common Files", 
		  "CommonProgramFiles(x86)": "C:\Program Files (x86)\Common Files", 
		  "CF_INSTANCE_IP": "192.168.50.6", 
		  "windir": "C:\Windows", 
		  "NUMBER_OF_PROCESSORS": "1", 
		  "OS": "Windows_NT", 
		  "ProgramFiles": "C:\Program Files", 
		  "ComSpec": "C:\Windows\system32\cmd.exe", 
		  "INSTANCE_GUID": "2ab613f5-7c40-447c-540d-c269bb0f5e25", 
		  "PSModulePath": "C:\Windows\system32\WindowsPowerShell\v1.0\Modules\", 
		  "INSTANCE_INDEX": "0", 
		  "CF_INSTANCE_ADDR": "192.168.50.6:50460", 
		  "PROCESSOR_LEVEL": "6", 
		  "CF_INSTANCE_PORTS": "[{"external":50460,"internal":8080}]", 
		  "VCAP_APPLICATION": "{"limits":{"mem":1024,"disk":1024,"fds":16384},"application_id":"d6a19802-d7f9-4a31-88db-82ff7ad361ee","application_version":"edec73a9-2289-48d1-8991-cec51839f6c6","application_name":"dotnet-test-app","version":"edec73a9-2289-48d1-8991-cec51839f6c6","name":"dotnet-test-app","space_name":"cloud","space_id":"9a42bc77-39d5-4175-9c93-d550e376de1a"}", 
		  "APP_POOL_ID": "AppPool50460", 
		  "PUBLIC": "C:\Users\Public", 
		  "APP_POOL_CONFIG": "C:\containerizer\858DE92B64DF613951\user\tmp\lifecycle\config\applicationHost.config"
		}
		
## 一些小问题
日志无法写入bug</br>
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/issue.JPG) <br />

## 总结
目前这个只是做为演示，不具备线上发布，个人对.net不感冒，所以也就到此为止了。至于源码，抽空在研究，现在精力放在docker上。	