title: 工厂模式的理解和php实践
date: 2013-10-13 01:07:04
tags:
- 工厂模式
category: 编码
---

更多人知道的模式应该是单例模式，工厂模式我觉得次之。对象的创建是模式的基础，这些模式或是借力于工厂，或是有些许工厂的影子。

工厂模式通常分简单工厂、工厂方法和抽象工厂。

举个很有体会的例子来说明，一个app产品，同时有Android，iOS，Win Phone几个手机平台的适配。假定这几个平台分别有同样的需求，即检测版本是否升级的接口。如果代码里有这样一段决策的逻辑：

<!-- more -->

```php
class Update{
    ...
    function toUpdate($uap){
        if($uap === 'android'){
            $upm = new AndroidUpm();
        }elseif($uap === 'ios'){
            $upm = new IosUpm();
        }else{
            $upm = new WpUpm();
        }
        return $upm->toUpdate();
    }
    ...
}
```
这种设计导致Update类依赖的是具体的产品类，形成了向下依赖。随着产品的变化，可能增加更多的平台例如：symbian、bada，就需要不断修改if/else的逻辑。还有可能，某天将iOS的逻辑修改为和Android一样。

这个时候，是时候想起OO的一个原则：`封装变化`，以此来达到`对修改封闭`的效果。分离决策对象创建的逻辑，引入工厂来专业做这件事情吧。
```php
class Update{
    ...
    function toUpdate($uap){
        $upm = SimpleUpmFactory::createUpm($uap);
        return $upm->toUpdate();
    }
    ...
}

class SimpleUpmFactory{
    static function createUpm($uap){
        if($uap === 'android'){
            $upm = new AndroidUpm();
        }elseif($uap === 'ios'){
            $upm = new IosUpm();
        }else{
            $upm = new WpUpm();
        }
        return $upm;
    }
}
```
这就引出了`简单工厂`SimpleUpmFactory，不少号称用了工厂方法的代码实际只是一个简单工厂，很可能还是个静态工厂。

不过不难发现，静态工厂不可取，一方面Update对其强依赖，可测性不高；另一方面限制了不能在运行时修改行为了。另一种更好的写法是这样：
```php
class Update{
    private $upmFac;
    
    public function __construct($upmFac){
        $this->upmFac = $upmFac;
    }
    
    public function setUpmFac($upmFac){
        $this->upmFac = $upmFac;
    }
    
    function toUpdate($uap){
        $upm = $this->upmFac->createUpm($uap);
        return $upm->toUpdate();
    }

    ...
}

class SimpleUpmFactory{
    function createUpm($uap){
        if($uap === 'android'){
            $upm = new AndroidUpm();
        }elseif($uap === 'ios'){
            $upm = new IosUpm();
        }else{
            $upm = new WpUpm();
        }
        return $upm;
    }
}
```
这样让依赖工厂的类采用组合并提供set方法实现依赖注入而不是静态依赖。

再回到简单工厂的问题，这个时候增加了一个类，分离了变化，以后要修改也只需修改工厂了。看似已经够用了，但是产品需求又变化了。这个时候需要增加pad版的适配，怎么办？一种扩展方法是直接修改现有的逻辑，增加if分支。你可能会说createUpm方法需要增加pad还是phone标识的形参，这个问题不用考虑，实际中的参数都是一个信息很全的参数数组，这里只是示例。另一种相对更好的扩展方法是再增加一个针对pad的简单工厂*SimplePadUpmFactory*，让phone和pad解耦，实现方法类似。
```php
class SimplePadUpmFactory{
    function createUpm($uap){
        if($uap === 'android'){
            $upm = new AndroidPadUpm();
        }elseif($uap === 'ios'){
            $upm = new IosPadUpm();
        }else{
            $upm = new WpPadUpm();
        }
        return $upm;
    }
}
```
这样又增加了一个类，依赖工厂的*Update*类也不需要做修改，看似简单工厂完全够用了呢。这种说法基于两个前提：

1）*Update*类的组合对象*upmFac*是超类型

在C++/Java里类属性是需要指定类型的。php用模式`不爽的地方`就是不用标明变量和函数返回的类型，包括函数形参的类型也是后面才可选地支持类型提示。

2）*Update*类的*toUpdate*接口返回类型是超类型

理由同上。这之前在说工厂`创建者`，并没有提创建的`产品`，是的产品也需要继承自同一个父类的。

也就是说，php里理论上可以不让创建者或者产品形成继承层次，但是你应该这么做才称得上是模式。

基于假设修改后，这么修改后就不再是`简单工厂`了，而是引入到`工厂方法`模式。重新调整代码后：
```php
interface UpmFactory{
    public function createUpm();
}

class PadUpmFactory implements UpmFactory{
    public function createUpm($uap){
        //略去if/else
    }
}

class PhoneUpmFactory implements UpmFactory{
    public function createUpm($uap){
        //略去if/else
    }
}

abstract class Upm{}
class AndroidAppUpm extends Upm{}
class AndroidPadUpm extends Upm{}
class IosAppUpm extends Upm{}
class IosPadUpm extends Upm{}
class WpAppUpm extends Upm{}
class WpPadUpm extends Upm{}
```

###也许你会有如下问题：

1）我怎么多了这么多类？

是的，工厂方法模式的特点或者说不太好的地方就是会架设两套平行的类层次，类数量会增加（抽象工厂更甚）。想想分散在业务逻辑里的if/else代码你会觉得这种集中管理是值得的。

2）为什么创建者的顶层类用接口而产品的顶层类用抽象类？

其实都可以，看实际需要。面向接口编程实际就是面向抽象而不是具体。

3）跟简单工厂相比的优势在哪？

你也看到前面的场景了，依赖工厂的*Update*类没有做任何修改就可以实现扩展，已有的创建者也不会受影响。工厂方法相当于一个创建框架，可以起到一定的收敛作用，例如：顶层类提供强制或推荐的一些方法调用。

4）那实例化延迟到子类怎么理解？

可以看到创建者父类并不知道要创建哪个产品，而是交给了子类。决定用什么子类实际是调用者（顾客）来决定。例如在MVC框架下，C层经常做这种事情。有的人就问了，我到C层的时候要用if/else来决定用哪个子类呢？这难道陷入死循环了？在实例化*UpmFactory*的时候你需要知道该用哪个了。如果出现分支选择，就需要修改类的设计了。一方面可以在C层也加一层工厂（MVC框架下，`每一层都可能都需要个工厂`），还有一种方案是利用框架的分发能力拆解。否则，你就需要下移这个决策到现有的工厂，也是需要修改设计的。

5）通过传参用简单工厂来实现，不也是直接依赖具体的类么？

前面的例子中工厂方法实现是简单工厂的方式，结合3）看他们并不是简单的替换关系。这种依赖是必要的，因为没法消灭new，只是把new依赖转移到工厂，而工厂是不妨碍单测的。

尽管如此，本人还是推荐让`工厂方法不要有if/else`。下面引入`抽象工厂`来改写。抽象工厂意味着产品簇，相当于创建工作的横向化。

由于平台有多个，就把各个平台当成一个簇来创建。
```php
interface UpmFactory{
    public function createAndroidUpm();
    public function createIosUpm();
    public function createWpUpm();
}

class PadUpmFactory implements UpmFactory{
    public function createAndroidUpm(){
        return new PadAndroidUpm();
    }
    public function createIosUpm(){
        return new PadIosUpm();
    }
    public function createWpUpm(){
        return new PadWpUpm();
    }
}

class PhoneUpmFactory implements UpmFactory{
    public function createAndroidUpm(){
        return new PhoneAndroidUpm();
    }
    public function createIosUpm(){
        return new PhoneIosUpm();
    }
    public function createWpUpm(){
        return new PhoneWpUpm();
    }
}

interface PadUpm{}
interface PhoneUpm{}
class PhoneAndroidUpm implements PhoneUpm{}
class PhoneIosUpm implements PhoneUpm{}
class PhoneWpUpm implements PhoneUpm{}
class PadAndroidUpm implements PadUpm{}
class PadIosUpm implements PadUpm{}
class PadWpUpm implements PadUpm{}
```
与工厂方法的实现相比较，去掉了if/else逻辑，创建框架也更加清晰。可以发现`抽象工厂的方法都是一个工厂方法`！这对于复用的修改是很灵活的。例如，Pad的iOS逻辑可以复用Phone的iOS逻辑。

再回到需求，如果不只是有检测版本是否升级的接口，还有是否激活检测、数据更新检测等接口呢？我们又想公用一个抽象工厂，还得修改设计，至少要把类名改成更通用：
```php
interface XmFactory{
    public function createAndroidUpm();
    public function createIosUpm();
    public function createWpUpm();
    public function createAndroidAcm();
    public function createIosAcm();
    public function createWpAcm();
    //...more
}
```
这样修改有个问题，会导致创建者子类不管需不需要都要加一堆实现。也许你会想把顶级*XmFactory*接口修改为abstract，对于可能不常用的方法提供默认实现。

本人更推荐稳定抽象工厂的簇维度，就是说设计抽象工厂的时候，也许有多种关联关系，优先`把稳定的维度作为簇`更好。总之能用interface有时候会比abstract好一点点，因为一般的面向对象语言都不提供多重继承，但是却有多重实现。这个案例，有三个维度：产品、平台、接口。`抽象工厂需要结合工厂方法`。

```php
interface XmFactory{
    public function createAndroid();
    public function createIos();
    public function createWp();
}

class PadmFactory implements XmFactory{
    public function createAndroid(){
        return new PadAndroid();
    }
    public function createIos(){
        return new PadIos();
    }
    public function createWp(){
        return new PadWp();
    }
}

abstract Pad{
    abstract public function createUpm();
    abstract public function createAcm();
}

class PadAndroid implements Pad{
    public function createUpm(){
        return new PhoneAndroidUpm();
    }
    public function createAcm(){
        return new PhoneAndroidAcm();
    }
}
//..其他类似
```
可以看到，维度超过2个的时候，抽象工厂创建的产品本身就是个工厂方法模式。在实践中，这么用的确容易绕进去，不过效果很好。

当维度再增加的时候，光用抽象工厂也不好做了，在项目中尝试了装饰模式的整合，效果不错。

这个模式的个人理解就说到这了。
