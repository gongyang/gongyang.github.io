---
title : 使用unicorn-worker-killer限制unicorn使用内存大小
date: 2016-7-24
---
## 使用unicorn-worker-killer限制unicorn使用内存大小
unicorn是[rack](https://en.wikipedia.org/wiki/Rack_(web_server_interface))应用广泛使用的http server。不过unicorn有一个缺点，就是可能会无限的增加内存使用，
unicorn-worker-killer gem提供基于unicorn进程使用最大内存和最大请求数自动重启的功能。并且不会影响正常的连接请求，这对于网站应用的稳定性提供了保障。

### 安装
unicorn-worker-killer使用很简单，只需要在Gemfile中添加该gem就可以了
```
gem 'unicorn-worker-killer'
```

### 使用
在config.ru添加以下内容，这里注意一点，配置要添加在“require ::File.expand_path('../config/environment',  __FILE__)”之上。

限制内存大小
**Unicorn::WorkerKiller::Oom(memory_limit_min=(1024\*\*3), memory_limit_max=(2\*(1024\*\*3)), check_cycle = 16, verbose = false)**

memory_limit_min和memory_limit_max指定了最大内存的一个范围值。实际值是由 rand()从memory_limit_min和memory_limit_max中间取一个值，这样做的好处就是为了防止所有work在同一个时间重启。
```
# Unicorn self-process killer
require 'unicorn/worker_killer'

# Max memory size (RSS) per worker
use Unicorn::WorkerKiller::Oom, (300*(1024**2)), (350*(1024**2))
```

unicorn-worker-killer还支持基于请求数来重启进程
**Unicorn::WorkerKiller::MaxRequests(max_requests_min=3072, max_requests_max=4096, verbose=false)**

max_requests_min和max_requests_max指定了最大请求数的一个范围值。实际值是由 rand()从max_requests_min和max_requests_max中间取一个值，同样这样也是为了防止所有work在同一个时间重启。

```
# Unicorn self-process killer
require 'unicorn/worker_killer'

# Max requests per worker
use Unicorn::WorkerKiller::MaxRequests, 3000, 4000
```
如果verbose设置为true，每一次的检查都会写到日志里面，日志级别为info。
