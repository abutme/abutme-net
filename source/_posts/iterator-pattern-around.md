title: 让iterator改变php传统编码
date: 2014-05-14 16:59:43
tags: 
- iterator
- 设计模式
- 数据库
- 迭代器模式
category: 编码
---

设计模式里有个迭代器模式，该模式是依赖Iterator接口的。php5开始支持内置的Iterator接口，有java编码经验的应该对java.util下的iterator不陌生。

之前读过鸟哥的一篇博文[关于一笔试题(Iterator模式) ][1]，其中深入分析了怎么用foreach遍历对象的私有属性。php5开始foreach支持遍历对象，不过是public属性。

<!-- more -->

Q1看完了`深入PHP面向对象、模式与实践`，顺便说这本书很值得一看。`数据库模式`这章翻译得不接地气，看了两遍才能结合实际开发来理解。数据库模式里提到的`数据映射器（mapper）`，相当于是一层对象关系映射，解决关系数据库和领域对象的`阻抗不匹配问题`，将数据库操作从领域模型中分离出来。

其中提到`一个问题`，mapper里如果需要向调用层返回一个对象数组，而数组中有1000个对象，直接返回是浪费资源影响性能的。这里提到一个解决方案，就是Iterator。

之前看到的数据库操作代码，大部分的风格都差不多：在各种getXX、updateYY...函数里，调用DB类（mysqli）连数据库，传进一坨查各种表的混乱的sql语句然后返回一个数据数组，也用不到领域模型。本人是面向对象的粉丝，自然对这种传统的写法不屑，维护和扩展起来实在不友好。

说这个，我表达的观点就是，mapper的场景，用面向对象来编码是常见的。

Iterator在这个场景的优势：

> 延迟对象实例化到调用那一刻

> 延迟数据库查找到确实需要那一刻

> 方便扩展和调用

当然，这种实现方式不好的地方也有，那就是增加类层次和复杂度。

我用了以下的代码来练习了一下：

```php Mapper.php

<?php

//trace function
function trace($str){
    echo $str."\n";
}

class Feature{

    private $appid;
    private $featureid;
    private $featurename;

    public function __construct($appid, $featureid, $featurename){
        $this->appid = $appid;
        $this->featureid = $featureid;
        $this->featurename = $featurename;
    }

    public function __toString(){
        return 'featureid: '.$this->featureid.'    featurename: '.$this->featurename."\n";
    }

}

class AppModel{

    private $appid;
    private $appname;
    private $features;

    public function __construct($appid, $appname){
        $this->appid = $appid;
        $this->appname = $appname;
    }

    public function setFeatures(Collection $features){
        $this->features = $features;
    }
    public function getFeatures(){
        return $this->features;
    }

}

abstract class Mapper{

    protected static $pdo = null;

    public function __construct(\PDO $pdo){
        if(!isset(static::$pdo)){
            static::$pdo = $pdo;
        }
    }

    protected function dsn(){
        return static::$pdo;
    }

    public function createObject($ary){
        return $this->doCreateObject($ary);
    }

    abstract public function selectStmt();
    abstract public function doCreateObject($ary);

}

class AppMapper extends Mapper{

    public function selectStmt(){
        if(!isset($this->selectStmt)){
            $this->selectStmt = $this->dsn()->prepare('SELECT * FROM app where id=?');
        }
        return $this->selectStmt;
    }

    public function findApp($id){
        $this->selectStmt()->execute(array($id));
        trace('Query __APP__');
        $ary = $this->selectStmt()->fetch();
        $this->selectStmt()->closeCursor();
        if(!is_array($ary) || !isset($ary['id'])){
            return null;
        }
        $object = $this->doCreateObject($ary);
        return $object;
    }

    public function doCreateObject($ary){
        trace('Create __APP__ object');
        $app = new AppModel($ary['appid'], $ary['appname']);
        $featureMapper = new FeatureMapper($this->dsn());
        $featureCollection = $featureMapper->findByApp($ary['appid']);
        $app->setFeatures($featureCollection);
        return $app;
    }
}

class FeatureMapper extends Mapper{

    public function selectStmt(){
        if(!isset($this->selectStmt)){
            $this->selectStmt = $this->dsn()->prepare('SELECT * FROM feature WHERE appid=?');
        }

        return $this->selectStmt;
    }

    public function findByApp($appid){
        return new FeatureCollection($this->selectStmt(), $this, array($appid));
    }

    public function doCreateObject($ary){
        trace('Create __FEATURE__ object');
        return new Feature($ary['appid'], $ary['featureid'], $ary['featurename']);
    }

}

abstract class Collection implements Iterator{

    private $raw;
    private $total;
    private $pointer;
    private $objects;

    public function __construct($raw = null, Mapper $mapper = null){
        if(!is_null($raw) && !is_null($mapper)){
            $this->raw = $raw;
            $this->total = count($raw);
        }
        $this->mapper = $mapper;
    }

    protected function setDeferredRaw($raw){
        $this->raw = $raw;
        $this->total = count($raw);
    }

    public function key(){
        return $this->pointer;
    }

    public function next(){
        $row = $this->getRow($this->pointer);
        if($row) {
            $this->pointer++;
        }
        return $row;
    }

    public function rewind(){
        $this->pointer = 0;
    }

    public function valid(){
        return !is_null($this->current());
    }

    public function current(){
        return $this->getRow($this->pointer);
    }

    private function getRow($num){
        $this->notifyAccess();
        if($num > $this->total || $num < 0){
            return null;
        }
        if(isset($this->objects[$num])){
            return $this->objects[$num];
        }
        if(isset($this->raw[$num])){
            $this->objects[$num] = $this->mapper->createObject($this->raw[$num]);
            return $this->objects[$num];
        }
        return null;
    }

    abstract protected function notifyAccess();

}

class FeatureCollection extends Collection{

    private $deferredValues = null;
    private $stmt;

    public function __construct($stmt, $mapper, $values){
        parent::__construct(null, $mapper);
        $this->stmt = $stmt;
        $this->deferredValues = $values;
    }

    protected function notifyAccess() {
        if(!isset($this->run)){
            trace('Query __FEATURE__');
            $this->stmt->execute($this->deferredValues);
            $this->setDeferredRaw($this->stmt->fetchAll());
        }
        $this->run = true;
    }
}

```

调用的代码如下：

```php PdoMapper.php

<?php

require_once './Mapper.php';

$dsn = 'mysql:dbname=abutme;host=localhost:3306';
$pdo = new PDO($dsn, 'root', 'wisetest');

$appMapper = new AppMapper($pdo);

//查询1次数据库 只创建了APP的对象
$app = $appMapper->findApp(2);

//获得Feature的Collection 并不查表和创建对象
$features = $app->getFeatures();

//真正有需求才查Feature表和创建Feature对象
foreach($features as $feature){
    echo $feature;
    unset($feature);
}

```

为了测试，随便建了两个表app表和feature表，假定一个app有多个feature：

```php

app表：
+----+-------+----------+
| id | appid | appname  |
+----+-------+----------+
|  1 | 1     | baidu    |
|  2 | 2     | baiduapp |
|  3 | 3     | freeapp  |
|  4 | 4     | weixin   |
|  5 | 5     | weibo    |
|  6 | 6     | qqmusic  |
+----+-------+----------+

feature表：
+----+-------+-----------+--------------+
| id | appid | featureid | featurename  |
+----+-------+-----------+--------------+
|  1 | 2     | 1         | ocr          |
|  2 | 2     | 2         | image search |
|  3 | 2     | 3         | bar code     |
|  4 | 2     | 4         | web search   |
+----+-------+-----------+--------------+


```

打印结果：

```php

Query __APP__
Create __APP__ object
Query __FEATURE__
Create __FEATURE__ object
featureid: 1    featurename: ocr
Create __FEATURE__ object
featureid: 2    featurename: image search
Create __FEATURE__ object
featureid: 3    featurename: bar code
Create __FEATURE__ object
featureid: 4    featurename: web search

```

可以看到iterator的写法很有意思，在没有真正用features时，是不会查询数据库和创建对象的。

当然测试代码还不是最优的，因为有sql语句的硬编码，还有创建语句都可以从中分离出来。

实际中，我还可以联想到的一个场景是`PB数据格式`的使用上，采用iterator可以解决大量数据对象的创建问题。


[1]: http://www.laruence.com/2008/10/31/574.html

