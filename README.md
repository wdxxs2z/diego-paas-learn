# diego-paas-learn
cloudfoundry第三代引擎diego源码学习</br>

本文将按如下结构开始分析diego
-----------------------------------
cloudfoundry 把第三代runtime的引擎命名为diego,虽然是一次被动的变革,但稍加分析后,发现diego变动的不仅仅是支持了docker那么简单,从整个架构的改变来看,后端容器的选择更加灵活，而且增加了对docker,buildpack 等形式应用的生命周期的管理,从一次tasks到一个LRP实例,至少和V2对比起来,这些都看起来优雅了很多。

还是先上一个图：</br>
![Peter don't care](https://github.com/wdxxs2z/PictureStore/blob/master/diego/diego-overview.png) <br />

可将其分为6层：
* CC-Bridge层</br>
* Route-emitter层</br>
* Access层</br>
* Brain层</br>
* Database层</br>
* Cells层</br>

Diego还是给我们带来了些惊喜，比如在某些时候(--exportNetworkEnvVars=true)可以自己指定容器IP端口等;对作业进行了生命周期式的划分，有点类似mesos+marathon的感觉;提供了计分的资源调度算法;引入btrfs(copy on write)文件系统,终于摆脱了aufs的束缚,也让docker更好的被引入;后端不仅支持linux还支持windows;引入cells的概念,跟k8s有点类似,一个cell就是一个调度的基本单元,上面可以有很多实例,rep相当于k8s的kubelete,garden相当于cell的代理，用来对接后端的各种类型容器;还有基本的容错功能,这都要归结于brain层的设计,等等等等。</br>

缺点和目前所有容器化一样，网络问题，假如采用ovs的gre/vxlan方式,可能在外层会更不方便管理多租户的情况和网络ip冲突问题,会增加系统设计的复杂度,而不具备通用性,更好的设计应该是根据特定的情况开启SDN功能。</br>

接下来，我将按照上述顺序进行学习。
</br>

2015年9月6日
