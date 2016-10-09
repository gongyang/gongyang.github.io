---
title : Use capistrano sequence restart service
date: 2016-6-30
---
由于capistrano的发布是一个并行的处理，所以在restart某个服务进程的时候，也是同时进行的。从某种程度上来说，在restart的过程当中（先stop，然后start），服务会存在短时间内不可用的状态。当然，如果服务进程支持平滑启动，就不存在上述的问题，像unicorn的启动，就是一种平滑启动方式。
**tips**
平滑启动，大家可以简单的理解为，在restart过程中，先启动一组新的进程用来接受新的请求，老的进程停止接收新的请求，然后老的进程处理完当前的请求自动退出。

目前我们在发布过程中，除了unicorn是平滑启动之外，sidekiq、sneaker等服务都是先stop、然后再start，那么这个过程当中，就会存在一段时间服务不可能。由于sidekiq是用来处理异步任务的，他的数据都是从redis里面读取，短时间的不可用还是可以接受。但是sneaker不行（原因是各应用之间的依赖），所以我们需要在发布过程中，修改sneaker的启动方式，先restart其中一台server的sneaker进程，然后在接着对另外一台server进行restart（当然，前提是至少你要在两台服务器上运行了sneaker work）

我们要修改sneaker的启动过程，前提是要知道deploy的哪个步骤会去调用sneaker的stop、start以及restart task。

我们先来了解一下运行cap production deploy会调用那些tasks，以及执行顺序。

```
deploy:starting    - start a deployment, make sure everything is ready
deploy:started     - started hook (for custom tasks)
deploy:updating    - update server(s) with a new release
deploy:updated     - updated hook
deploy:publishing  - publish the new release
deploy:published   - published hook
deploy:finishing   - finish the deployment, clean up everything
deploy:finished    - finished hook
```

cap production deploy:rollback 调用的tasks

```
deploy:starting
deploy:started
deploy:reverting           - revert server(s) to previous release
deploy:reverted            - reverted hook
deploy:publishing
deploy:published
deploy:finishing_rollback  - finish the rollback, clean up everything
deploy:finished
```

查看sneakers.rb
```
after :publishing, :restart_sneakers do
    invoke 'sneakers:restart' if fetch(:sneakers_default_hooks)
  end
```
从上面代码我们可以分析得出，在deploy:publishing之后会调用sneakers:restart。然后我们在搜一下有没有在deploy过程中调用sneakers:stop之类的，因为如果有在sneakers:restart之前就是执行了sneakers:stop，那其实整个服务就停止了，哪怕做到分台停止也没有任何意义了，如果有的话，那么就需要注释掉，因为在restart过程中我们会先stop，然后start。

然后我们来看一下sneakers:restart会做哪些操作
```
desc 'Restart sneakers'
  task :restart do
    invoke 'sneakers:stop'
    invoke 'sneakers:start'
  end
```
是不是一脸懵逼，没错，就是这么简单；然后我就直接修改这个task，
```
desc 'Restart sneakers'
  task :restart do
    on roles(:sneakers_server), in: :sequence do
    invoke 'sneakers:stop'
    sleep 2
    invoke 'sneakers:start'
  end
 end
```
上面的意思就是sneakers_server定义的服务器按照顺序执行stop、start。表面上看上去问题就这样解决了，如果有多台 sneakers 服务器，一台一台执行stop、start。然后并没有这么简单，部署的时候发现，stop的时候，所有的sneaker服务器还是同时停止work。这个时候，我们就得看sneakers:stop这个task究竟干了什么
```
 desc 'Stop sneakers'
  task :stop do
    on roles fetch(:sneakers_role) do
      if test("[ -d #{current_path} ]")
        for_each_sneakers_process(true) do |pid_file, idx|
          if sneakers_pid_process_exists?(pid_file)
            stop_sneakers(pid_file)
          end
        end
      end
    end
  end
```
又是一脸懵逼，原来在stop的时候fetch(:sneakers_role)所有的sneaker服务器，执行invoke 'sneakers:stop'的时候，必然会停掉所有sneaker work。看来只能重构restart这个task了。
```
 desc 'Restart sneakers'
  task :restart do
    on roles(:sneakers_server), in: :sequence  do
    if test("[ -d #{current_path} ]")
        for_each_sneakers_process(true) do |pid_file, idx|
          if sneakers_pid_process_exists?(pid_file)
            stop_sneakers(pid_file)
        end
      end
    end  
    sleep 3
    for_each_sneakers_process do |pid_file, idx|
        start_sneakers(pid_file, idx) unless sneakers_pid_process_exists?(pid_file)
     end
   end
 end
```
虽说是重构，实际上就是把stop和start task里面的代码整合到一起，中间sleep 3秒的作用就是：因为stop是使用kill -TERM终止信号，如果进程很繁忙的时候，发送该信号，需要等待几秒钟进程才会退出，然后才能执行start操作。不然进程还没有退出就开始进行start操作，sneaker是不会执行start task，导致服务重启失败

最后测试，达到预期效果，了解了如何修改sneaker重启过程，同样的像sidekiq，unicorn也是类似的过程。

