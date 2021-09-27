# 服务容器


# 服务容器

## 简介

Laravel 服务容器是一个用于管理类依赖以及实现依赖注入的强有力工具。依赖注入这个名词表面看起来花哨，实质上是指：通过构造函数，或者某些情况下通过「setter」方法将类依赖「注入」到类中。

我们来看一个简单的例子:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Repositories\UserRepository;
use App\Models\User;

class UserController extends Controller
{
    /**
     * user 仓储的实现
     *
     * @var UserRepository
     */
    protected $users;

    /**
     * 创建一个新的控制器实例
     *
     * @param  UserRepository  $users
     * @return void
     */
    public function __construct(UserRepository $users)
    {
        $this->users = $users;
    }

    /**
     * 展示给定用户的信息
     *
     * @param  int  $id
     * @return Response
     */
    public function show($id)
    {
        $user = $this->users->find($id);

        return view('user.profile', ['user' => $user]);
    }
}
```

在这个例子中，`UserController` 控制器需要从数据源中获取 `users`。所以，我们可以注入一个能够获取 users 的服务。在这种情况下，我们的存储仓库 `UserRepository` 极有可能使用 Eloquent 从数据库中获取用户信息。然而，因为仓储是通过 UserRepository 注入的，我们可以很轻易的将其切换为另一个实现。 另外，这种方式的便利之处也体现在：当需要为应用编写测试的时候，我们也可以很轻松地 “模拟” 或者创建一个 `UserRepository` 存储层的伪实现来操作。

深入理解服务容器，对于构建一个强大的、大型的应用，以及对 Laravel 核心本身的贡献都是至关重要的。

### 零配置解决方案

如果一个类没有依赖项或只依赖于其他具体类 (而不是接口），则不需要指定容器如何解析该类。例如，您可以将以下代码放在 `routes/web.php` 文件中：

```php
<?php

class Service
{
    //
}

Route::get('/', function (Service $service) {
    die(get_class($service));
});
```

在本例中，点击应用程序的 `/` 路由将自动解析 `Service` 类并将其注入路由的处理程序。这是一个有趣的改变。这意味着您可以开发应用程序并利用依赖注入，而不用再去关心臃肿的配置。

很荣幸的通知你，在构建 Laravel 应用程序时，您将要编写的许多类都可以通过容器自动接收它们的依赖关系，包括 [控制器](https://learnku.com/docs/laravel/8.x/controllers)，[事件](https://learnku.com/docs/laravel/8.x/events)，[中间件](https://learnku.com/docs/laravel/8.x/middleware) 。此外，您可以在 [队列](https://learnku.com/docs/laravel/8.x/queues) 的 `handle` 方法中键入提示依赖项。一旦你尝到了零配置依赖注入的威力，你就会觉得没有它是不可以开发的。

### 何时使用容器

得益于零配置解决方案，通常情况下，你只需要在路由、控制器、事件侦听器和其他地方键入提示依赖项，而不必手动与容器打交道。例如，可以在路由定义中键入 `Illuminate\Http\Request` 对象，以便轻松访问当前请求的 Request 类。尽管我们不必与容器交互来编写此代码，但它在幕后管理着这些依赖项的注入：

```php
use Illuminate\Http\Request;

Route::get('/', function (Request $request) {
    // ...
});
```

在许多情况下，由于自动依赖注入和 [facades](https://learnku.com/docs/laravel/8.x/facades)，你在构建 laravel 应用程序的时候无需 **手动** 绑定或解析容器中的任何内容。**那么，你将在什么时候手动与容器打交道呢**？让我们来看看下面两种情况。

首先，如果要编写一个实现接口的类，并希望在路由或类的构造函数上键入该接口的提示，则必须 [告诉容器如何解析该接口](about:blank#binding-interfaces-to-implementations)。第二，如果您正在 [编写一个 Laravel 包](https://learnku.com/docs/laravel/8.x/packages) 计划与其他 Laravel 开发人员共享，那么您可能需要将包的服务绑定到容器中。

## 绑定

### 基础绑定

### 简单绑定

几乎所有的服务容器绑定都会在 [服务提供者](https://learnku.com/docs/laravel/8.x/providers) 中注册，下面示例中的大多数将演示如何在该上下文（服务提供者）中使用容器。

在服务提供者中，你总是可以通过 `$this->app` 属性访问容器。我们可以通过容器的 `bind` 方法注册绑定，`bind` 方法的第一个参数为要绑定的类或接口名，第二个参数是一个返回类实例的闭包：

```php
use App\Services\Transistor;
use App\Services\PodcastParser;

$this->app->bind(Transistor::class, function ($app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

我们可以看下bind()方法的源代码

```php
public function bind($abstract, $concrete = null, $shared = false)
{
		//删除旧的绑定
    $this->dropStaleInstances($abstract);
    //如果没有给出实现类 那么实现类就是抽象
    if (is_null($concrete)) {
        $concrete = $abstract;
    }
    //如果$concrete不是闭包, 要把$concrete转换成闭包
    if (! $concrete instanceof Closure) {
        if (! is_string($concrete)) {
            throw new TypeError(self::class.'::bind(): Argument #2 ($concrete) must be of type Closure|string|null');
        }
        $concrete = $this->getClosure($abstract, $concrete);
    }
		//保存到$this->bindings数组
    $this->bindings[$abstract] = compact('concrete', 'shared');
    //如果已经实例化过, 要调用一下实例化的回调
    if ($this->resolved($abstract)) {
        $this->rebound($abstract);
    }
}
```

bind方法的核心就是把抽象和实现的对应关系保存到bindings数组里面, 为了保持统一, 所有的实现类都被转换成了闭包. 

注意，我们接受容器本身作为解析器的参数。然后，我们可以使用容器来解析正在构建的对象的子依赖。

### 单例的绑定

`singleton` 方法将类或接口绑定到只应解析一次的容器中。解析单例绑定后，后续调用容器时将返回相同的对象实例：

```php
use App\Services\Transistor;
use App\Services\PodcastParser;

$this->app->singleton(Transistor::class, function ($app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

我们可以看下singleton()方法是怎么实现的:

```php
public function singleton($abstract, $concrete = null)
{
    $this->bind($abstract, $concrete, true);
}
```

直接调用的bind()方法, $shared参数传true, 表示这个绑定是共享的, 也就是单例.

### 绑定实例

还可以使用 `instance` 方法将现有对象实例绑定到容器中。给定实例将始终在随后调用容器时返回：

```php
use App\Services\Transistor;
use App\Services\PodcastParser;

$service = new Transistor(new PodcastParser);

$this->app->instance(Transistor::class, $service);
```

我们来看就源码:

```php
public function instance($abstract, $instance)
{
		//从上下文绑定别名缓存中删除别名。
    $this->removeAbstractAlias($abstract);
		//确定给定的抽象类型是否已绑定。
    $isBound = $this->bound($abstract);
		//删除已经注册过的别名
    unset($this->aliases[$abstract]);

		//核心 把实例直接保存在instances数组中
		$this->instances[$abstract] = $instance;

    if ($isBound) {
        $this->rebound($abstract);
    }

    return $instance;
}
```

我们发现如果绑定的是一个实例, 会直接把这个实例放在实例数组中, 使用的时候可以直接取到.

### 绑定接口至实现

服务容器的一个非常强大的特性是它能够将接口绑定到给定的实现。例如，假设我们有一个 `EventPusher` 接口和一个 `RedisEventPusher` 实现。一旦我们对这个接口的 `RedisEventPusher` 实现进行了编码，我们就可以像这样在服务容器中注册它：

```php
use App\Contracts\EventPusher;
use App\Services\RedisEventPusher;

$this->app->bind(EventPusher::class, RedisEventPusher::class);
```

此语句告诉容器，当类需要 `EventPusher` 的实现时，它应该注入 `RedisEventPusher`。现在我们可以在容器解析的类的构造函数中加上 `EventPusher` 接口作为类型提示。请记住，控制器、事件侦听器、中间件和 Laravel 应用程序中的各种其他的类始终使用容器进行解析：

```php
use App\Contracts\EventPusher;

/**
 * 创建一个新的实例
 *
 * @param  \App\Contracts\EventPusher  $pusher
 * @return void
 */
public function __construct(EventPusher $pusher)
{
    $this->pusher = $pusher;
}
```

在前文中我们提到, 如果绑定的是一个实现类, 不是闭包, 那么会把实现类转换成闭包, 然后再放进bindings数组中保存.

我们看下获取一个实现类的闭包是怎么实现的:

```php
protected function getClosure($abstract, $concrete)
{
    return function ($container, $parameters = []) use ($abstract, $concrete) {
        if ($abstract == $concrete) {
						//如果抽象和实现类相同, 那么调用build方法
            return $container->build($concrete);
        }
				//如果抽象和实现类不同, 那么调用resolve方法
        return $container->resolve(
            $concrete, $parameters, $raiseEvents = false
        );
    };
}
```

build方法和resolve方法都是创建实例相关的代码, 我们后面在讲解析make()方法的时候会着重讲解这两个方法. 在这里你知道要知道, 如果穿进来的是一个实现类, 那么在保存关系的时候会把实现类转换成创建实现类实例的闭包. 这样bindings中的抽象和实现的关联关系就统一了起来, 都是抽象类型对应实现类的闭包.

### 上下文绑定

> 译者注：所谓「上下文绑定」就是根据上下文进行动态的绑定。

有时你可能有两个类使用相同的接口，但是你希望将不同的实现分别注入到各自的类中。例如，两个控制器可能依赖于 `Illuminate\Contracts\Filesystem\Filesystem` [契约](https://learnku.com/docs/laravel/8.x/contracts) 的不同实现。Laravel 提供了一个简单流畅的方式来定义这种行为：

```php
use App\Http\Controllers\PhotoController;
use App\Http\Controllers\UploadController;
use App\Http\Controllers\VideoController;
use Illuminate\Contracts\Filesystem\Filesystem;
use Illuminate\Support\Facades\Storage;

$this->app->when(PhotoController::class)
          ->needs(Filesystem::class)
          ->give(function () {
              return Storage::disk('local');
          });

$this->app->when([VideoController::class, UploadController::class])
          ->needs(Filesystem::class)
          ->give(function () {
              return Storage::disk('s3');
          });
```

有了上下文关系之后, 我们就可以先设置上下文的信息, 然后容器反射创建类的实例的时候, 就可以根据上下文信息, 对构造方法进行传参.

我们可以看下代码:

```php
public function when($concrete)
{
    $aliases = [];
		//如果传进来$concrete是个数组, 这个地方也是可以兼容的
    foreach (Util::arrayWrap($concrete) as $c) {
        $aliases[] = $this->getAlias($c);
    }

    return new ContextualBindingBuilder($this, $aliases);
}
```

调用when方法的时候返回了一个`\Illuminate\Container\ContextualBindingBuilder`实例

```php
public function needs($abstract)
{
    $this->needs = $abstract;

    return $this;
}
```

needs设置了需要的抽象是什么

```php
public function give($implementation)
{
    foreach (Util::arrayWrap($this->concrete) as $concrete) {
				//保存上下文绑定信息
        $this->container->addContextualBinding($concrete, $this->needs, $implementation);
    }
}
```

我们依次给容器添加了上下文绑定

```php
public function addContextualBinding($concrete, $abstract, $implementation)
{
    $this->contextual[$concrete][$this->getAlias($abstract)] = $implementation;
}
```

由此可知绑定信息是方放在容器的contextual数组里面, 通过这个数组, 我们在实例化的时候, 可以轻松找到构造方法对应的参数应该怎么传.

### 绑定基本值

有时您可能有一个类接收一些注入的类，但也需要一个注入的原语值，如整数。您可以轻松地使用上下文绑定来注入类可能需要的任何值：

```php
$this->app->when('App\Http\Controllers\UserController')
          ->needs('$variableName')
          ->give($value);

```

有时类可能依赖于 [标记](about:blank#tagging) 实例数组。使用 `givetaged` 方法，可以轻松地将所有容器绑定插入该标记中：

```php
$this->app->when(ReportAggregator::class)
    ->needs('$reports')
    ->giveTagged('reports');
```

```php
public function giveTagged($tag)
{
		//调用give方法, 传入了闭包, 这个闭包的作用是使用标签来获取被打标的对应的所有抽象类型
    $this->give(function ($container) use ($tag) {
        $taggedServices = $container->tagged($tag);

        return is_array($taggedServices) ? $taggedServices :iterator_to_array($taggedServices);
    });
}
```

give方法前面已经讲解过了, 依次保存上下文信息到服务容器中去. 

如果需要从应用程序的某个配置文件中插入值，可以使用 `giveConfig` 方法：

```php
$this->app->when(ReportAggregator::class)
    ->needs('$timezone')
    ->giveConfig('app.timezone');
```

```php
public function giveConfig($key, $default = null)
{
		//传入了闭包, 这个闭包的作用是使用$key和$default来获取系统的
    $this->give(function ($container) use ($key, $default) {
        return $container->get('config')->get($key, $default);
    });
}
```

如果

### 绑定变长参数类型

有时，您可能会有一个类，它的构造函数使用可变参数接收指定类型的对象的数组：

```php
<?php

use App\Models\Filter;
use App\Services\Logger;

class Firewall
{
    /**
     * 日志实例
     *
     * @var \App\Services\Logger
     */
    protected $logger;

    /**
     * 过滤器实例数组
     *
     * @var array
     */
    protected $filters;

    /**
     * 创建一个类实例
     *
     * @param  \App\Services\Logger  $logger
     * @param  array  $filters
     * @return void
     */
    public function __construct(Logger $logger, Filter ...$filters)
    {
        $this->logger = $logger;
        $this->filters = $filters;
    }
}

```

使用上下文绑定，可以通过为 `give` 方法提供一个闭包来解析此依赖关系，该闭包返回已解析的 `Filter` 实例数组：

```php
$this->app->when(Firewall::class)
          ->needs(Filter::class)
          ->give(function ($app) {
                return [
                    $app->make(NullFilter::class),
                    $app->make(ProfanityFilter::class),
                    $app->make(TooLongFilter::class),
                ];
          });
```

为了方便起见，您还可以只提供一个类名数组，以便在 `Firewall` 需要 `Filter` 实例时由容器解析：

```php
$this->app->when(Firewall::class)
          ->needs(Filter::class)
          ->give([
              NullFilter::class,
              ProfanityFilter::class,
              TooLongFilter::class,
          ]);
```

> 译者注：后一种方法可以在用到这些过滤器的时候，才实例化它，而不是在 Firwall 类实例化的时候，就实例化所有的过滤器。

### 变长参数的关联标签

有时一个类可能有变长参数的依赖关系，它的类型提示是给定的类（ `Report` ）。使用 `needs` 和 `givetaged` 方法，您可以轻松地为给定的依赖项注入所有具有该 [tag](about:blank#tagging) 的容器绑定：

```php
$this->app->when(ReportAggregator::class)
    ->needs(Report::class)
    ->giveTagged('reports');
```

> 译者注：最后一句比较绕，可以这样理解：你可以把这些类组织到一个 标签 中，这样你就可以轻松地为特定的依赖项打包一起注入到容器中。

### 标签

有时，您可能需要解析绑定的所有特定 「分类」 的类。例如，您可能正在构建一个报表分析器，它接收许多不同的 `report` 接口实现的数组。注册 `Report` 实例后，可以使用 `tag` 方法为它们分配一个统一的标记：

```php
$this->app->bind(CpuReport::class, function () {
    //
});

$this->app->bind(MemoryReport::class, function () {
    //
});

$this->app->tag([CpuReport::class, MemoryReport::class], 'reports');
```

```php
public function tag($abstracts, $tags)
{
		//$tags既可以是一个tag数组, 也可以是tag字符串, laravel框架有很多种这样的写法, 来兼容输入的类型
    $tags =is_array($tags) ? $tags :array_slice(func_get_args(), 1);

    foreach ($tags as $tag) {
        if (! isset($this->tags[$tag])) {
            $this->tags[$tag] = [];
        }

        foreach ((array) $abstracts as $abstract) {
            $this->tags[$tag][] = $abstract;
        }
    }
}
```

tag方法代码很简单, 就是使用tags数组去保存tag对应的抽象类型.

一旦服务被打上标签，你就可以通过容器的 `tagged` 方法轻松地解析它们：

```php
$this->app->bind(ReportAnalyzer::class, function ($app) {
    return new ReportAnalyzer($app->tagged('reports'));
});
```

```php
public function tagged($tag)
{
    if (! isset($this->tags[$tag])) {
        return [];
    }
		//返回了一个迭代器, 我们可以直接foreach tagged($tag)的结果 每个元素是tag对应抽象类型的实例
    return new RewindableGenerator(function () use ($tag) {
        foreach ($this->tags[$tag] as $abstract) {
            yield $this->make($abstract);
        }
    },count($this->tags[$tag]));
}
```

### 继承绑定

`extend` 方法允许修改已解析的服务。例如，解析服务时，可以运行其他代码来修饰或配置服务。 `extend` 方法接受闭包，该闭包应返回修改后的服务作为其唯一参数。闭包接收正在解析的服务和容器实例：

```php
$this->app->extend(Service::class, function ($service, $app) {
    return new DecoratedService($service);
});
```

```php
public function extend($abstract, Closure $closure)
{
    $abstract = $this->getAlias($abstract);
		
    if (isset($this->instances[$abstract])) {
        $this->instances[$abstract] = $closure($this->instances[$abstract], $this);

        $this->rebound($abstract);
    } else {
        $this->extenders[$abstract][] = $closure;

        if ($this->resolved($abstract)) {
            $this->rebound($abstract);
        }
    }
}
```

extend可以对已经解析的实例进行扩展, 使用闭包实现.  如果没有解析的实例, 会放在容器的extenders数组中进行保存, 到时候解析的时候, 可以直接扩展.

> 译者注：类似于类的继承

## 解析

### `make` 方法

可以使用 `make` 方法从容器中解析类实例。 `make` 方法接受要解析的类或接口的名称：

```php
use App\Services\Transistor;

$transistor = $this->app->make(Transistor::class);
```

```php
public function make($abstract, array $parameters = [])
{
    return $this->resolve($abstract, $parameters);
}
```

```php
protected function resolve($abstract, $parameters = [], $raiseEvents = true)
{
		//获取抽象的别名
    $abstract = $this->getAlias($abstract);

		if ($raiseEvents) {
				//调用解析之前回调
        $this->fireBeforeResolvingCallbacks($abstract, $parameters);
    }

		//获取上下文绑定的实现
    $concrete = $this->getContextualConcrete($abstract);

    $needsContextualBuild = ! empty($parameters) || !is_null($concrete);

		//如果有单例且不需要参数构造 直接返回
		if (isset($this->instances[$abstract]) && ! $needsContextualBuild) {
        return $this->instances[$abstract];
    }

    $this->with[] = $parameters;

		//上下文没有定义实现的时候 要找绑定的实现
    if (is_null($concrete)) {
        $concrete = $this->getConcrete($abstract);
    }
		
		//可以构造的条件是: $concrete === $abstract || $concrete instanceof Closure
		if ($this->isBuildable($concrete, $abstract)) {
				**//方法的核心build**
        $object = $this->build($concrete);
    } else {
				//不能构造就创建实现类
        $object = $this->make($concrete);
    }

		//如果对实例有扩展回调的话, 也要依次调用一下
		foreach ($this->getExtenders($abstract) as $extender) {
        $object = $extender($object, $this);
    }
		
		//如果是分享的实例, 要放在instances中保存
		if ($this->isShared($abstract) && ! $needsContextualBuild) {
        $this->instances[$abstract] = $object;
    }
		
    if ($raiseEvents) {
				//解析后回调
        $this->fireResolvingCallbacks($abstract, $object);
    }
		
		//所有解析过的抽象要标记
		$this->resolved[$abstract] = true;

		array_pop($this->with);

    return $object;
}
```

```php
public function build($concrete)
{
		//如果实现是闭包, 那么直接调用闭包方法就创建了实例了
		if ($concrete instanceof Closure) {
				//这个闭包有两个参数, 第一个参数是服务容器, 所有自定义闭包的时候一定要传容器
        return $concrete($this, $this->getLastParameterOverride());
    }

    try {
				//通过反射获取反射类实例
        $reflector = new ReflectionClass($concrete);
    } catch (ReflectionException $e) {
        throw new BindingResolutionException("Target class [$concrete] does not exist.", 0, $e);
    }

		//如果构造方法是private的, 直接抛异常
		if (! $reflector->isInstantiable()) {
        return $this->notInstantiable($concrete);
    }

		$this->buildStack[] = $concrete;
		//获取到了反射的构造方法
    $constructor = $reflector->getConstructor();
		//如果没有构造方法, 直接就new一个实例
		if (is_null($constructor)) {
				array_pop($this->buildStack);

        return new $concrete;
    }
		
		//获取构造方法的依赖
    $dependencies = $constructor->getParameters();

		try {
				//获取构造方法的实例
        $instances = $this->resolveDependencies($dependencies);
    } catch (BindingResolutionException $e) {
				array_pop($this->buildStack);

        throw $e;
    }

		array_pop($this->buildStack);
		//传参构造方法创建实例
    return $reflector->newInstanceArgs($instances);
}
```

```php
protected function resolveDependencies(array $dependencies)
{
    $results = [];

    foreach ($dependencies as $dependency) {
				//如果参数覆盖了, 使用覆盖后的参数
				if ($this->hasParameterOverride($dependency)) {
            $results[] = $this->getParameterOverride($dependency);
            continue;
        }
				
				//如果参数的声明是某个类型 实例化类 如果是基本类型 就去上下文找
				$result =is_null(Util::getParameterClassName($dependency))
												//解析基本类型
                        ? $this->resolvePrimitive($dependency)
												//解析类
                        : $this->resolveClass($dependency);

        if ($dependency->isVariadic()) {
            $results =array_merge($results, $result);
        } else {
            $results[] = $result;
        }
    }

    return $results;
}
```

```php
protected function resolvePrimitive(ReflectionParameter $parameter)
{
		//上下文如果定义的话, 直接从上下文获取
    if (!is_null($concrete = $this->getContextualConcrete('$'.$parameter->getName()))) {
        return $concrete instanceof Closure ? $concrete($this) : $concrete;
    }
		//没有定义看有没有默认值
    if ($parameter->isDefaultValueAvailable()) {
        return $parameter->getDefaultValue();
    }
		
		//除此之外要抛出异常
    $this->unresolvablePrimitive($parameter);
}
```

```php
protected function resolveClass(ReflectionParameter $parameter)
{
    try {
				//判断是否是可变参数
        return $parameter->isVariadic()
										//如果是可变参数
                    ? $this->resolveVariadicClass($parameter)
										//如果不是可变参数可以直接创建参数对应的实例 getParameterClassName是获取参数的类名
                    : $this->make(Util::getParameterClassName($parameter));
    }

		catch (BindingResolutionException $e) {
        if ($parameter->isDefaultValueAvailable()) {
						array_pop($this->with);

            return $parameter->getDefaultValue();
        }

        if ($parameter->isVariadic()) {
						array_pop($this->with);

            return [];
        }

        throw $e;
    }
}
```

```php
protected function resolveVariadicClass(ReflectionParameter $parameter)
{
    $className = Util::getParameterClassName($parameter);

    $abstract = $this->getAlias($className);

    if (!is_array($concrete = $this->getContextualConcrete($abstract))) {
        return $this->make($className);
    }

    returnarray_map(function ($abstract) {
        return $this->resolve($abstract);
    }, $concrete);
}
```

如果类的某些依赖项不能通过容器进行解析，可以通过将它们作为关联数组传递到 `makeWith` 方法来注入它们。例如，我们可以手动传递 `Transistor` 服务所需的 `$id` 构造函数参数：

```php
use App\Services\Transistor;

$transistor = $this->app->makeWith(Transistor::class, ['id' => 1]);
```

```php
public function makeWith($abstract, array $parameters = [])
{
    return $this->make($abstract, $parameters);
}
```

makeWith其实是make方法的别名, 只不过用makeWith更符合语义: 带参数的make. make方法在前面已经讲解, 这里就不再重复了. 

如果你在服务提供商之外的代码位置无法访问 `$app` 变量，则可以使用 `App` [facade](https://learnku.com/docs/laravel/8.x/facades) 从容器解析类实例：

```php
use App\Services\Transistor;
use Illuminate\Support\Facades\App;

$transistor = App::make(Transistor::class);
```

如果希望将 Laravel 容器实例本身注入到由容器解析的类中，可以在你的类的构造函数中添加 `Illuminate\container\container`：

```php
use Illuminate\Container\Container;

/**
 * 实例化一个类
 *
 * @param  \Illuminate\Container\Container
 * @return void
 */
public function __construct(Container $container)
{
    $this->container = $container;
}
```

### 自动注入

另外，并且更重要的是，你可以简单地使用「类型提示」的方式在类的构造函数中注入那些需要容器解析的依赖项，包括 [控制器](https://learnku.com/docs/laravel/8.x/controllers)，[事件](https://learnku.com/docs/laravel/8.x/events)，[中间件](https://learnku.com/docs/laravel/8.x/middleware) 等 。此外，你也可以在 [队列任务](https://learnku.com/docs/laravel/8.x/queues) 的 `handle` 方法中使用「类型提示」注入依赖。实际上，这才是大多数对象应该被容器解析的方式。

例如，你可以在控制器的构造函数中添加一个 repository 的类型提示，然后这个 repository 将会被自动解析并注入类中：

```php
<?php

namespace App\Http\Controllers;

use App\Repositories\UserRepository;

class UserController extends Controller
{
    /**
     * user 仓储实例
     *
     * @var \App\Repositories\UserRepository
     */
    protected $users;

    /**
     * 创建一个控制器实例
     *
     * @param  \App\Repositories\UserRepository  $users
     * @return void
     */
    public function __construct(UserRepository $users)
    {
        $this->users = $users;
    }

    /**
     * 使用给定的 id 显示 user
     *
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function show($id)
    {
        //
    }
}
```

## 容器事件

服务容器每次解析对象会触发一个事件，你可以使用 `resolving` 方法监听这个事件：

```php
use App\Services\Transistor;

$this->app->resolving(Transistor::class, function ($transistor, $app) {
    // 当容器解析类型为 "Transistor" 的对象时调用...
});

$this->app->resolving(function ($object, $app) {
    // 当容器解析任何类型的对象时调用
});
```

如你所见，正在解析的对象将被传递给回调，从而允许你在将对象提供给其使用者之前设置该对象的任何附加属性。

```php
public function resolving($abstract, Closure $callback = null)
{
    if (is_string($abstract)) {
        $abstract = $this->getAlias($abstract);
    }
		//如果只传了$abstract恰巧$abstract还是一个闭包 就注册一个全局的解析回调
    if (is_null($callback) && $abstract instanceof Closure) {
        $this->globalResolvingCallbacks[] = $abstract;
    } else {
				//否则就注册一个抽象类型的回调
        $this->resolvingCallbacks[$abstract][] = $callback;
    }
}
```

这个方法其实是有歧义的, 我不建议这样写代码, 这样会给阅读源码的人带来困惑, 违反了单一指责原则. 一个方法实际上实现了两个功能, 一个是注册全局回调, 一个是注册了解析了某个抽象的回调. 当然原理是简单的, 还是用数组保存抽象对应的回调就可以了.

## PSR-11

Laravel 的容器服务实现了 [PSR-11](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-11-container.md) 接口。 因此，您可以在 PSR-11 容器接口的类型提示中获取 Laravel 容器的实例：

```php
use App\Services\Transistor;
use Psr\Container\ContainerInterface;

Route::get('/', function (ContainerInterface $container) {
    $service = $container->get(Transistor::class);

    //
});

```

如果给定的标识符无法解析，将引发异常。如果标识符从未绑定，则异常将是 `Psr\Container\NotFoundExceptionInterface` 的实例。如果标识符已绑定但无法解析，则将抛出 `Psr\Container\ContainerExceptionInterface` 的实例。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接
我们的翻译工作遵照 CC 协议，如果我们的工作有侵犯到您的权益，请及时联系我们。
