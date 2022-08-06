# Thinkphp inc数组注入

**影响版本**：

tp5.0.13-5.0.15

tp5.1.0-5.1.5

### 准备

centos php5.4.0

thinkphp 5.0.13

phpstorm

#### 安装源文件

通过composer直接安装

```
composer create-project --prefer-dist topthink/think=5.0.15 tpdemo
```

![](https://s2.loli.net/2022/08/06/OvwsFopWezqytQK.png)

安装时有出现报错信息，原因为镜像地址问题，无法找到框架包，切换下composer镜像地址即可成功安装

```
composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/
```

更改tpdemo里面的composer.json的内容，将require修改下：

```
  "require": {

​    "php": ">=5.4.0",

​    "topthink/framework": "5.0.15"

  }
```

更改后执行composer update，在index.php下面添加以下代码：

```
public function sqli() {
// 从GET数组方式获取用户信息
$user = input('get.username/a');
// 实例化数据库类并且调用insert方法进行数据库插入操作
db('users')->where(['id' =>1])->insert(['username' => $user]);
  }
```

新建数据库

```
create database tpdemo;

use tpdemo;

create table users(

  id int primary key auto_increment,

  username varchar(50) not null

);
```



首先放上payload：

http://localhost/tpdemo/public/index.php/index/index/sqli?username[0]=inc&username[1]=updatexml(1,concat(0x7,user(),0x7e),1)&username[2]=1**

![](https://s2.loli.net/2022/08/06/jt7GE5dVgvUPpSu.png)

这里放上mochazz师傅文章里的流程图

![](https://s2.loli.net/2022/08/06/zEDHC5q6IicP42u.jpg)

首先查看报错的信息：

![](https://s2.loli.net/2022/08/06/oQc7whiKxZ69va1.png)

6-11条主要是tp的初始化部分，重点在1-5，首先第五行是我们的sqli方法

\```

```
  public function sqli() {
// 从GET数组方式获取用户信息
$user = input('get.username/a');
// 实例化数据库类并且调用insert方法进行数据库插入操作
db('users')->where(['id' =>1])->insert(['username' => $user]);
  }
```

\```

input这里不做具体分析，主要看insert这里，跟组函数来到insert方法，第一步这里可以看到data的值，也就是我们传入的内容

![](https://s2.loli.net/2022/08/06/O9NYlAoZrSUuwL4.png)

下一步进入parseExpress方法，该方法的功能是分析表达式，其他的值对我们的不是很重要，看下经过分析后的option的值这里的重点就是下一步就会用到的data，也就是$option['data']

![](https://s2.loli.net/2022/08/06/7pYGrW2RCO68zTe.png)

下一步是array_merge()，该函数为数组合并函数，可以将多个数组合并为一个数组，合并后的data值仍然是我们之前传入的

![](https://s2.loli.net/2022/08/06/H2CvLlMQEu4Fhdm.png)

接下来调用builder的insert方法，也就是关键部分

![](https://s2.loli.net/2022/08/06/RuvbF6cUmWjZ4yt.png)

跟下来一步步调试，首先进入的parseData方法，这里对data内容进行处理，产生漏洞的关键点也就是在这里

在方法里面对data的val进行遍历并赋值给key，经过遍历，我们的val值为Inc，接下来的代码意思是如果val为exp、inc、dec的话就走parseKey方法将SQL语句拼接起来

![](https://s2.loli.net/2022/08/06/A34WspZO7gyHEcq.png)

经过parseKey方法后，key的值为：

![](https://s2.loli.net/2022/08/06/tYuSjhwnRxG9lN5.png)

接下来看这段代码，如果是标量数据的话就会进入这段，从而进行预处理，php里面的标量类型数据有四种，分别是int、float、bool、string，而我们传入的内容为数组，进入inc之后就会break跳出，也就不会进入这个预处理的地方

![](https://s2.loli.net/2022/08/06/g6OyvIjYzR7BhDk.png)

至此parseData方法结束

![](https://s2.loli.net/2022/08/06/3DnL8zigAdZl4ay.png)

接下来的就是正则替换函数，我们继续往下跟

![](https://s2.loli.net/2022/08/06/iW2kOT5q7Ujfocy.png)

之后我们的SQL语句是这样的

![](https://s2.loli.net/2022/08/06/87BTmLFgcKVd1nz.png)

接下来就是执行语句啦

主要漏洞点是用数组传入的值，这样就绕过