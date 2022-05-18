---
layout: post
title: "Chaz6chez/structure的使用方法"
date: 2016-07-08 01:00:00
image: ''
description: 'Chaz6chez/structure的使用方法'
tags:
  - PHP
  - 组件
categories:
  - Chaz6chez
---

# [Structure](https://github.com/chaz6chez/structure)

**A useless validator**

## 应用场景

- 出入参的判断及过滤
- 数据转化及映射

## 示例

### 验证场景

- 创建一个Structure

````
namespace YoursNamespace;
use Structure\Struct;
class User extends Struct {

    // 当id为非null值时，必须满足int 10 < id < 20
    /**
     * @rule int,min:10,max:20|id format error:1001
     */
    public $id; 
    
    // name为必填，必须满足string 1 < name长度 < 20
    /**
     * @rule string,min:1,max:20|name format error:2001
     * @required true|name cannot be empty:2002 
     */
    public $name;
    
    // 当sex为null时默认 string female
    // 满足string 0 < sex长度 < 10
    /**
     * @rule string,min:0,max:10|sex format error:1001
     * @default string:female
     */
    public $sex;
}
````

- 使用

````
    $struct = \YoursNamespace\User::factory();
    $struct->create([
        'id' => 12,
        'name' => 'John Smith'
    ]);
    
    // or
    
    $struct = \YoursNamespace\User::factory([
        'id' => 12,
        'name' => 'John Smith'
    ]);
    
    // or
    
    $struct = new \YoursNamespace\User();
    $struct = \YoursNamespace\User::factory();
    
    $struct->id = 12;
    $struct->name = 'John Smith';
    
    // or
    
    $struct = new \YoursNamespace\User([
        'id' => 12,
        'name' => 'John Smith'
    ]);
    

    if($struct->hasError()) {
        throw new \RuntimeException(
            $struct->getError()->getMessage() . '->' . $struct->getError()->getPosition(),
            $struct->getError()->getCode()
        );
    }
    return $struct->output(); // array
````


## 使用说明

- 继承 Structure\Struct 及实现结构体
- public属性接参

````
namespace Example;

use Structure\Struct;

class User extends Struct{
    public $id;
    public $name;
    public $sex;
}
````

- 对要操作和转化的public属性进行注释

````

    /**
     * @rule string,min:10,max:20|name format error:1001 
     */
    public $name;
````

- 标签分为四个区域
    - <a href="#标签区">a区：标签区</a>
    - b区：场景区
    - c区：验证区
    - d区：内容信息
```PHP
/**
 *  a区   b区        c区              d区
 * @标签 [场景]   验证方式   | 错误信息     : 错误码
 *      ↓           ↓           ↓             ↓
 * @rule[check] string,min:1|error message:error code  
 */                       
````

***
## <a id="标签区">标签区</a>：

- **转换类的标签配合 filter()在output() 方法内生效，
  会对包含该标签的属性执行转换或者过滤操作**


- **验证类的标签在 validate() 中生效返回布尔值，
  通过getError() 可以获得错误信息**

|标签名|方式|类型|说明|
|:---:|:--------------:|:----------------:|:----------------------:|
| <a href="#@default">@default</a> | Structure\Handler、func、method | 转换 | func与method是将返回值默认赋予该标签 |
| <a href="#@required">@required</a>| true |验证| 判断是否为必要值 |
| <a href="#@rule">@rule</a> | Structure\Handler、func、method |验证| 以func与method的bool返回类型判断验证 |
| <a href="#@skip">@skip</a> | 无 |验证| 跳过验证 |
| <a href="#@ghost">@ghost</a>| 无 |转换| 跳过输出 |
| <a href="#@key">@key</a>| 无 |转换| 标记钥匙属性|
| <a href="#@mapping">@mapping</a>| 映射键名 |转换| 映射键转换 |
| <a href="#@operator">@operator</a>| true、func、method |转换| 键值特殊转换 |

### <a id="@default">@default</a>
- 将该属性标记默认模式
- 当该属性值为null且具备@default标签时生效

````
    /**
     * @default string:abc
     * @default int:123
     * @default float:1.1
     * @default object:Handler\Help
     * @default map:{"a":"1"}
     * @default array:["1"]
     * @default bool:true
     */
    public $name;
````

- 验证区可使用func、method进行方法赋值
    - **method:className,methodName 必须是静态方法**
    - **方法执行过程中抛出的任何异常都会被忽略，并以默认Null赋值**

````
    /**
     * @default func:is_array              会找到is_array函数
     * @default method:_set                会定位当前类的_set方法
     * @default method:Handler\Help,get   会定位Handler\Help类的get方法 
     */
    public $name;
    public static function _set(){
        return 'abc';
    }
````

- **@default仅在output()输出时生效，若要直接使用类属性获取@default赋值，请使用以下方法：**
- **但不建议频繁使用，会多执行一次object clone操作**

```PHP
    // 以name的@default标签为string:John举例
    /**
     * @default string:John
     */
    public $name;
 ```

```PHP 
    $struct = new Struct();
    // @default无法生效，值为null
    $struct->name;
    
    // @default可以生效，值为字符串John
    $struct()->name;
```

### <a id="@required">@required</a>

````
    /**
     * @required true|name cannot empty
     */
    public $name;
````

### <a id="@rule">@rule</a>

- 通过预置Handler进行验证

````
    /**
     * @rule string,min:10,max:20|name format error
     * @rule int,min:10,max:20|name format error
     * @rule float,min:1.0,max:2.1,scale:3|name format error
     * @rule bool,true|name format error
     * @rule object,class:|name format error
     * @rule array,min:10,max:20,values:string|name format error
     * @rule map,min:1,max:5,keys:string,values:int|name format error
     * @rule url,path:true,query:true|name format error
     * @rule ip,ipv4:false,ipv6:false,private:false,reserved:false|name format error
     * @rule regex,min:10,max:20,regex:/.?/|name format error
     */
    public $name;
````

- 验证区可使用func、method进行方法判断
    - **method:className,methodName 必须是静态方法**
    - **方法执行过程中任何异常会转化成StructureException抛出**

````
    /**
     * @rule func:_set                  会找到_set函数
     * @rule method:_set                会定位当前类的_set方法
     * @rule method:Handler\Help,get   会定位Handler\Help类的get方法
     */
    public $name;
    
    public static function _set($value) : bool
    {
        return $value === '_method';
    }
}


function _set($value) : bool
{
    return $value === '_func';
}
````
### <a id="@ghost">@ghost</a>
- <a href="#输出">output()</a> 不会输出该标签

````
    // @ghost true
    $user->id = 'id';

    $user->name = 'name';
    $user->output();
    // 以上会输出
    [
        'name' => 'name'
    ];

    $user->output(true);
    // 以上会输出
    [
        'id' => 'id',
        'name' => 'name'
    ];
````

### <a id="@key">@key</a>
- 将该属性标记钥匙字段
- 通过 <a href="#过滤">filter()</a>-><a href="#输出">output()</a> 可以做到仅输出钥匙字段

````
    // @key true
    $user->id = 'id';

    $user->name = 'name';
    $user->filter(STRUCT_FILTER_KEY)->output();
    // 以上会输出
    [
        'name' => 'name'
    ];
    
    $user->transfer(STRUCT_FILTER_KEY_REVERSE)->output();
    // 以上会输出
    [
        'id' => 'id',
    ];
````

### <a id="@skip">@skip</a>
- 跳过验证，但不影响输出

### <a id="@operator">@operator</a>

#### 1. 识别[medoo语法-where](https://medoo.in/api/where) 并转换
````
    /**
     * @operator true 
     */ 
    public $name;
````

通过 <a href="#转换">transfer()</a>-><a href="#输出">output()</a> 可以做到转换输出

````
    // @operator true
    $user->id = 'abc[>]';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出
    [
        'id[>]' => 'abc'
    ];

    $user->id = '123,456[<>]';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出
    [
        'id[<>]' => ['123','456'],
    ];
````

    2.2以上版本完善了该标签下的类型转换
    2.2以下版本中不会对类型转换
##### 2.2以下版本：
````
    // @operator true
    $user->id = '123[>]';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出
    [
        'id[>]' => '123'
    ];
    // 并非期待的
    [
       'id[>]' => 123
    ];
    // 此种状况会影响数据库查询的索引
````
##### 2.2以上版本：

- 不仅可以配合medoo语法做处理转换，也可以直接做类型转换
- 字符串类型的数字默认会根据值类型转换
    - 整型内容字符串转换成整型
    - 小数字符串转换成浮点型

````
    // @operator true
    $user->id = '123[>]';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出 int
    [
        'id[>]' => 123
    ];
    
    // @operator true
    $user->id = '123';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出 int
    [
        'id' => 123
    ];
    
    $user->id = '1.23[>]';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出 float
    [
        'id[>]' => 1.23
    ];

    $user->id = '1.23';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出
    [
        'id' => 1.23
    ];
````

- 可以使用强制转换标签


````

# String :

    // @operator true
    $user->id = '123[String][>]';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出 int
    [
        'id[>]' => '123'
    ];
    
    // @operator true
    $user->id = '123[String]';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出 int
    [
        'id' => '123'
    ];

# Int ：

    // @operator true
    $user->id = '1[Int][>]';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出 int
    [
        'id[>]' => 1
    ];
    
    // @operator true
    $user->id = '1[Int]';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出 int
    [
        'id' => 1
    ];


# Float ：

    // @operator true
    $user->id = '123[Float][>]';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出 int
    [
        'id[>]' => 123.0
    ];
    
    // @operator true
    $user->id = '123[Float]';
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出 int
    [
        'id' => 123.0
    ];

# Bool ：

    // @operator true
    $user->id = '1[Bool][>]'; // 0 false
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出 int
    [
        'id[>]' => true
    ];
    
    // @operator true
    $user->id = '1[Bool]'; // 0 false
    $user->transfer(STRUCT_TRANSFER_OPERATOR)->output();
    // 以上会输出 int
    [
        'id' => true
    ];
````  





#### 2.使用func、method进行转换
- **method:className,methodName 必须是静态方法**
- **方法执行过程中任何异常会转化成StructureException抛出**

````

    /**
     * @operator func:_add                  相当于_add($name)
     * @operator method:_add                相当于$this->_add($name)
     * @operator method:Handler\Help,_add  相当于Handler\Help::_add($name)
     */
    public $name;
````


### <a id="@mapping">@mapping</a>
- 将该属性标记映射处理
- 通过 <a href="#转换">transfer()</a>-><a href="#输出">output()</a> 可以做到转换输出

````
    // @mapping key
    $user->id = 123;

    $user->name = 'john';
    $user->output();
    // 输出
    [
        'id' => 123,
        'name' => 'john'
    ];

    $user->transfer(STRUCT_TRANSFER_MAPPING)->output();
    // 输出
    [
        'key' => 123,
        'name' => 'john'
    ];
````

***
## 方法

- 实例化

````
    $user = User::factory([
        'id' => 1,
        'name' => 'john'
    ],'check');
````

- 输入

    - 使用create方法输入数据
    - 使用属性赋值输入数据
    - **使用create可以保存<a href="#获取原始数据">原始数据</a>，使用属性赋值则不会保留原始数据**

````
    // 1.使用create输入数据
    $user->create([
        'id' => 1,
        'name' => 'john'
    ]); // return $this
````

````
    // 2.使用属性赋值输入数据
    $user->id = 1;
    $user->name = 'john';

    // 使用create可以保存原始数据，建议使用create输入数据
````

- <a id="获取原始数据">获取原始数据</a>

````
    $user->getRaw(); // return array
````

- 设置场景

````
    $user->scene('check'); // return $this
````

- <a id="转换">转换</a>

    - STRUCT_TRANSFER_MAPPING
    - STRUCT_TRANSFER_OPERATOR

````
    $user->transfer(STRUCT_TRANSFER_MAPPING); // return $this

    // STRUCT_TRANSFER_MAPPING
    // STRUCT_TRANSFER_OPERATOR

    // 接受可变长参数
    $user->transfer(
        STRUCT_TRANSFER_MAPPING,
        STRUCT_TRANSFER_OPERATOR
    ); // return $this
````

- <a id="过滤">过滤</a>
    - STRUCT_FILTER_NULL
    - STRUCT_FILTER_EMPTY
    - STRUCT_FILTER_ZERO
    - STRUCT_FILTER_KEY
    - STRUCT_FILTER_KEY_REVERSE
    - STRUCT_FILTER_OPERATOR
    - STRUCT_FILTER_OPERATOR_REVERSE

````
    $user->filter(STRUCT_FILTER_NULL); // return $this

    // STRUCT_FILTER_NULL
    // STRUCT_FILTER_EMPTY
    // STRUCT_FILTER_ZERO
    // STRUCT_FILTER_KEY
    // STRUCT_FILTER_KEY_REVERSE
    // STRUCT_FILTER_OPERATOR
    // STRUCT_FILTER_OPERATOR_REVERSE

    // 接受可变长参数
    $user->filter(
        STRUCT_FILTER_NULL,
        STRUCT_FILTER_EMPTY
    ); // return $this
````

- <a id="验证">验证</a>

````
    $user->validate(); // return bool
    $user->hasError(); // return bool

    // true 有错误，验证未通过
    // false 无错误，验证通过
````

- 获取错误

    - 需要在<a href="#验证">验证</a>执行后才能获取错误信息

````
    $user->getError(); // return Structure\Error

    $user->getError()->getMessage();  // 错误信息 string
    $user->getError()->getCode();     // 错误码 string
    $user->getError()->getField();    // 字段名 string
    $user->getError()->getPosition(); // 错误定位 对应Handler对应的options字段

    $user->getErrors(); // return Structure\Error[]

````

- <a id="输出">输出</a>

    - 全量输出会进行<a href="#转换">转换</a>和default赋值
    - 全量输出不进行<a href="#过滤">过滤</a>

````
    $user->output(); // return array

    $user->output(true); // 全量输出
````

- 清洗

````
    $user->clean(); // 默认不装载raw数据

    $user->clean(true); // 装载raw数据
````

## 补充

- Handler 接受自定义注册

````
    \Structure\Handler::register();
````

- StructureException

````
    try {
        // ...                          
    }catch (\Structure\Exceptions\StructureException $exception){
        $exception->getPosition(); // 错误定位信息
        $exception->getMessage(); // Structure Exception [{position info}]
        $exception->getCode(); // 始终以-666返回
    }
````