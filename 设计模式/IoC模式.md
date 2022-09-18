## 开篇简介
通过用户登录时记录日志这一实际问题，引出控制反转/依赖注入/反射概念，最后再介绍IoC容器。

## 依赖与耦合

我们接到任务：用户登录时需要记录日志，同时可以在文件与数据库中切换。下面我们用代码来展示。

```php
<?php
// 日志接口
interface Log
{
	public function write();
}

// 文件记录
class FileLog implements Log
{
    public function write() {
        echo 'file log write' . PHP_EOL;
    }
}

// 数据库记录
class DatabaseLog implements Log
{
    public function write() {
        echo 'database log write' . PHP_EOL;
    }
}

// 用户
class User
{
    protected $log;

    public function __construct() {
        $this->log = new FileLog();
    }

    public function login() {
        echo 'login successful' . PHP_EOL;
        $this->log->write();
    }
}

$user = new User();
$user->login();

```

上面的代码有一个问题，如果想修改成数据库日志记录的话，User类就要修改，代码之间有耦合。把日志处理类通过构造函数参数的形式传递进去，即可解耦合。User类修改如下：

```php
class User
{
    protected $log;

    public function __construct(Log $log) {
        $this->log = $log;
    }

    public function login() {
        echo 'login successful' . PHP_EOL;
        $this->log->write();
    }
}

$user = new User(new FileLog());
$user->login();
```

## 控制反转与依赖注入

1.1控制反转

控制反转即IoC（Inversion of Control）,是一种设计原则，用来降低代码之间的耦合度。将组件之间的依赖关系从程序内部提到外部容器进行管理。哪些方面的控制被反转了？是依赖对象的获取被反转了。拿上述例子来说，之前是User对象主动获取依赖对象Log，现在改成了User对象被动的接受依赖对象Log。

1.2 依赖注入

DI(Dependency Injection,依赖注入)，是一种控制反转的实现方式。组件的依赖通过外部以参数或者其他形式注入。拿上述例子来说，User对象的依赖对象Log通过构造函数的参数注入其中，这也同时解释了依赖与注入两个词的意义。

## 反射

Laravel是通过反射来实现依赖注入的。反射的概念就是通过类名来返回该类的信息，比如有什么方法，参数等。下面通过创建一个make函数来实现，通过User类名来实现User类的。

```php
// 用户
class User
{
    protected $log;

    // 由于接口不能实例化，所以改为具体类。如果要用接口，下一节Ioc容器会讲到怎样实现
    public function __construct(FileLog $log) {
        $this->log = $log;
    }

    public function login() {
        echo 'login successful' . PHP_EOL;
        $this->log->write();
    }
}

// 通过类名实例化
function make($concrete) {
    $reflection = new ReflectionClass($concrete);
    $constructor = $reflection->getConstructor();
    // 没有构造器，直接实例化
    if (is_null($constructor)) {
        return $reflection->newInstance();
    } else {
        $dependencies = $constructor->getParameters();
        // 获取依赖
        $instances = getDependencies($dependencies);

        return $reflection->newInstanceArgs($instances);
    }
}

// 获取依赖
function getDependencies($paramters) {
    $dependencies = [];
    // 通过ReflectionParameter类获取实例化对象
    foreach ($paramters as $param) {
        $dependencies[] = make($param->getClass()->name);
    }

    return $dependencies;
}

$user = make('User');
$user->login();
```



## IoC容器

再进一步，我们可以新建一个专门处理类与类之间依赖关系的类，这个类就是IoC容器。
具体代码实现的思路为：
1.IoC容器维护bindings数组记录，通过bind传入键值，提前把log、user绑定到容器中
2.make函数来获取User对象，通过反射机制获取到依赖对象FileLog，最终完成User对象

最终的调用代码

```php
$app = new Container();
// 完成容器填充
$app->bind('Log', 'FileLog');
$app->bind('user', 'User');
// 完成实例化
$user = $app->make('user');
$user->login();
```

核心容器代码

```php
class Container
{
    /**
     * 用于存储提供实例的回调函数
     *
     * @var array
     */
    protected $bindings = [];

    /**
     * 绑定接口和生成相应实例的回调函数
     *
     * @param string $abstract
     * @param string $concrete
     * @return void
     */
    public function bind($abstract, $concrete)
    {
        // bind时还不需要创建对象，采用回调函数是等make再创建对象
        $this->bindings[$abstract]['concrete'] = function ($container) use ($concrete) {
            return $container->build($concrete);
        };
    }

    /**
     * 生成实例对象
     *
     * @param string $abstract
     * @return object
     */
    public function make($abstract)
    {
        $concrete = $this->bindings[$abstract]['concrete'];
        return $concrete($this);
    }

    /**
     * 实例化对象
     *
     * @param string $concrete
     * @return object
     */
    public function build($concrete)
    {
        $reflection = new ReflectionClass($concrete);
        $constructor = $reflection->getConstructor();
        if (is_null($constructor)) {
            return $reflection->newInstance();
        } else {
            $dependencies = $constructor->getParameters();
            $instances = $this->getDependencies($dependencies);
            return $reflection->newInstanceArgs($instances);
        }
    }

    /**
     * 获取实例化对象的依赖
     *
     * @param array $paramters
     * @return array
     */
    public function getDependencies($paramters)
    {
        $dependencies = [];
        foreach ($paramters as $paramter) {
            $dependencies[] = $this->make($paramter->getClass()->name);
        }
        return $dependencies;
    }
}
```

## 总结
本篇文章通过用户登录时记录日志这一需求的实现，不断提出新的问题来引出一些概念，最终写出了IoC容器的核心代码。下一章我们详解一下Laravel的IoC容器代码。

## 参考文档
[依赖注入、控制反转、反射各个概念的理解和使用](https://laravelacademy.org/post/9782.html)  
[如何实现 IoC 容器和服务提供者是什么概念](https://laravelacademy.org/post/9783.html)

