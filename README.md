# diego-paas-learn
cloudfoundry第三代引擎diego源码学习</br>

本文将按如下结构开始分析diego
-----------------------------------
cloudfoundry 把第三代runtime的引擎命名为diego,虽然是一次被动的变革,但稍加分析后,发现diego变动的不仅仅是支持了docker那么简单,从整个架构的改变来看,后端容器的选择更加灵活，而且增加了对docker,buildpack 等形式应用的生命周期的管理,从一次tasks到一个LRP实例,至少和V2对比起来,这些都看起来优雅了很多。


