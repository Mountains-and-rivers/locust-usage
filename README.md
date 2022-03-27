# locust使用

locustfile.py

```
from locust import HttpUser, task, constant,SequentialTaskSet,TaskSet,events
import logging as log
import json

@events.test_start.add_listener
def on_start(**kwargs):
    log.info("A test will start...")

@events.test_stop.add_listener
def on_stop(**kwargs):
    log.info("A test is ending...")

# 该类中的任务执行顺序是 index ---> login  Task 可以加到类上
WEB_HEADER = None

# 无序并发任务
class SetTask(TaskSet):
    @task(1)
    def getLogDetail(self):
        deatil_url = "/auth/online?page=0&size=10&sort=id%2Cdesc"
        with self.client.request(method='GET',
                                 url=deatil_url,
                                 headers=WEB_HEADER,
                                 name='获取日志详情') as response:
            log.info("11111111111111111111111111111111111")
            #log.info(response.text)
    @task
    def stop(self):
        log.info("33333333333333333333333333333333")
        self.interrupt()
# 有序序并发任务
class FlaskTask(SequentialTaskSet): #该类下没有智能提示
    # 登录获取 token，拼接后续请求头
    def on_start(self):
        res = self.client.request(method='GET', url="/auth/code",name="获取验证码")
        uuid = res.json()['uuid']
        headers = {
            "content-type": "application/json;charset=UTF-8",
        }
        datda_info = {
            "code": "8",
            "password": "B0GdcVWB+XtTwsyBjzoRkn8VnSgtPVKpl8mp7AuQ+BTeU030grUkRwmOHXFCjEhKXB7yezBS7dFEJ63ykR2piQ==",
            "username": "admin",
            "uuid": uuid
        }
        with self.client.request(method='POST',url="/auth/login", headers=headers, catch_response=True,data=json.dumps(datda_info),name="获取token") as response:
            self.token = response.json()['token']
            if response.status_code == 200:
                self.token = response.json()['token']
                response.success()
            else:
                response.failure("获取token失败")
            global WEB_HEADER
            WEB_HEADER = {
                "Authorization": self.token
            }
    #嵌套无序并发任务
    tasks = [SetTask]
    @task #关联登录成功后的token，也即 前一个业务返回的业务需要传递给后续的请求
    def getUserDetail(self):
        deatil_url = "/api/dictDetail?dictName=user_status&page=0&size=9999"
        with self.client.request(method='GET',
                                 url=deatil_url,
                                 headers=WEB_HEADER,
                                 name='获取用户详情') as response:
            log.info("22222222222222222222222222222222222222")
            #log.info(response.text)

def function_task():
    log.info("This is the function task")

class FlaskUser(HttpUser):
    host = 'http://192.168.31.243:8888'  # 设置根地址
    wait_time = constant(1)  # 每次请求的延时时间
    #wait_time = between(1,3)  # 等待时间1~3s
    tasks = [FlaskTask] # 指定测试任务
```

### 单机模式启动测试

```
locust -f locustfile.py
```

### 集群模式启动

```
locust -f locustflask.py --master #制定master 节点

locust -f locustflask.py --worker  --master-host=192.168.31.182 #制定worker节点
```

# 参数化

### 队列参数化

```
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
from locust import HttpUser, task
import queue
#使用队列参数化
class FecMallUser(HttpUser): #该类下没有智能提示
    host = 'http://192.168.31.243:8888'  # 设置根地址
    def __init__(self,*args,**kwargs):
        super().__init__(*args,**kwargs)
        self.q = queue.Queue()
    def on_start(self):
        for i in range(3):
            self.q.put(i)
    @task
    def login(self):
        m = self.q.get()
        print(m)
        self.q.put(m)
```

### 使用开源库参数化

```
from mimesis import Person
from mimesis.enums import Gender
from mimesis.schema import Schema
import random
import re
def get_register_data(iteration=1):
    p = Person()
    schema = Schema(schema=lambda :{
        'email': p.email(domains=['qq.com','163.com','126.com'],unique=True),
        'password': '123456',
        'firstname': p.first_name(gender=Gender.FEMALE if random.randint(1,10) > 6 else Gender.MALE),
        'lastname': p.last_name(),
        'is_subscribed': True if random.randint(1,10) > 6 else False
    })
    return schema.create(iteration)
s = 'qweqeqeqwrwfrwerfewrfabc888888ssssrwereqwrq;lwhjr;lqwhjer;ljqw'

def find_all_data(data,LB='',RB=''):
    rule = LB + r'(.+?)' + RB
    data_list = re.findall(rule,data)
    return data_list

#查找文本格式中的 字符串
ss = find_all_data(s,'abc','ssss')
print(ss)
```



# 安装依赖

```
pip3 install locust -i http://pypi.douban.com/simple --trusted-host pypi.douban.com
pip3 install mimesis -i http://pypi.douban.com/simple --trusted-host pypi.douban.com
```



## 问题解决

```
win 11 启动master模式 报错，把pysmq版本降低 即可解决 参考：
https://github.com/locustio/locust/issues/2058
```

# python 编译安装

```
1，解压缩Python-3.7.3.tgz文件

 运行以下命令：

 tar -xzvf Python-3.7.3.tgz,2，建立一个编译目录：

mkdir /usr/local/python3

3、安装依赖包

通常情况下，在CentOS下安装Python 3.7序列版本还需要下面的的依赖包：

 yum install -y libffi-devel zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel zlib zlib-devel gcc make

4、编译和安装Python-3.7.3.tgz

 cd Python-3.7.3

./configure --prefix=/usr/local/bin/python3
make
make install
```

