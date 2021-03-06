# [(转)详解spl_autoload_register()函数][0]

[**Corwien**][3] 6月12日发布 


> 看到一篇不错的博文，转载过来，可以通过这个自动加载函数`spl_autoload_register()`来理解PHP的类自动加载原理。

在了解这个函数之前先来看另一个函数：`__autoload`。

## 一、__autoload

这是一个自动加载函数，在PHP5中，当我们实例化一个未定义的类时，就会触发此函数。看下面例子：

    printit.class.php 

```php  
<?php 
class PRINTIT { 

    function doPrint() {
        echo 'hello world';
    }
}
```

    index.php 

```php
<?php
function __autoload( $class ) {
    $file = $class . '.class.php';  
    if ( is_file($file) ) {  
        require_once($file);  
    }
} 

$obj = new PRINTIT();
$obj->doPrint();
```

运行index.php后正常输出hello world。在index.php中，由于没有包含printit.class.php，在实例化printit时，自动调用`__autoload`函数，参数$class的值即为类名printit，此时printit.class.php就被引进来了。

> 在面向对象中这种方法经常使用，可以避免书写过多的引用文件，同时也使整个系统更加灵活。

## 二、spl_autoload_register()

再看 `spl_autoload_register()`，这个函数与`__autoload`有异曲同工之妙，看个简单的例子：

```php
<?php
function loadprint( $class ) {
    $file = $class . '.class.php';  
    if (is_file($file)) {  
        require_once($file);  
    } 
} 
 
spl_autoload_register( 'loadprint' ); 
 
$obj = new PRINTIT();
$obj->doPrint();
```

将`__autoload`换成loadprint函数。但是loadprint不会像`__autoload`自动触发，这时`spl_autoload_register()`就起作用了，它告诉PHP碰到没有定义的类就执行loadprint()。 

**spl_autoload_register() 调用静态方法**

```php
<?php
 
class test {
    public static function loadprint( $class ) {
        $file = $class . '.class.php';  
        if (is_file($file)) {  
            require_once($file);  
        } 
    }
} 
 
spl_autoload_register(  array('test','loadprint')  );
//另一种写法：spl_autoload_register(  "test::loadprint"  ); 
 
$obj = new PRINTIT();
$obj->doPrint();
```

小结：实例化时`__autoload`会被自动触发该函数， 如果没有执行的对象时，就会执行s`pl_autoload_register`该方法。

## 三、composer类自动加载研究

vendor/autoload.php

```php
<?php

// autoload.php @generated by Composer

require_once __DIR__ . '/composer/autoload_real.php';

return ComposerAutoloaderInit83cb48187cf44a304a7a6be5e700ede3::getLoader();
```

autoload_real.php

```php
<?php

// autoload_real.php @generated by Composer

class ComposerAutoloaderInit83cb48187cf44a304a7a6be5e700ede3
{
    private static $loader;

    public static function loadClassLoader($class)
    {
        if ('Composer\Autoload\ClassLoader' === $class) {
            require __DIR__ . '/ClassLoader.php';
        }
    }

    public static function getLoader()
    {
        if (null !== self::$loader) {
            return self::$loader;
        }

        spl_autoload_register(array('ComposerAutoloaderInit83cb48187cf44a304a7a6be5e700ede3', 'loadClassLoader'), true, true);
        self::$loader = $loader = new \Composer\Autoload\ClassLoader();
        spl_autoload_unregister(array('ComposerAutoloaderInit83cb48187cf44a304a7a6be5e700ede3', 'loadClassLoader'));

        $useStaticLoader = PHP_VERSION_ID >= 50600 && !defined('HHVM_VERSION') && (!function_exists('zend_loader_file_encoded') || !zend_loader_file_encoded());
        if ($useStaticLoader) {
            require_once __DIR__ . '/autoload_static.php';

            call_user_func(\Composer\Autoload\ComposerStaticInit83cb48187cf44a304a7a6be5e700ede3::getInitializer($loader));
        } else {
            $map = require __DIR__ . '/autoload_namespaces.php';
            foreach ($map as $namespace => $path) {
                $loader->set($namespace, $path);
            }

            $map = require __DIR__ . '/autoload_psr4.php';
            foreach ($map as $namespace => $path) {
                $loader->setPsr4($namespace, $path);
            }

            $classMap = require __DIR__ . '/autoload_classmap.php';
            if ($classMap) {
                $loader->addClassMap($classMap);
            }
        }

        $loader->register(true);

        return $loader;
    }
}
```

[0]: https://segmentfault.com/a/1190000009742195
[1]: https://segmentfault.com/t/autoload/blogs
[2]: https://segmentfault.com/t/php/blogs
[3]: https://segmentfault.com/u/corwien