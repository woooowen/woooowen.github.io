---
layout: post
title: php array_merge方法跟 + 的区别
category: [PHP]
tags: [PHP]
---

####php中合并数组除了可以用到array_merge 方法,还有 + 号
例如:

```js
$c = array_merge($a,$b);
$c = $a + $b;
```
但是说到这2者之间的区别还是有很多不同地方的.主要是跟数组的键值存在关系
###example1 : 键值为数字时
```js
//code:
$a = array(1,2,3);
$b = array(3,4,5);
print_r($a + $b);
$c = array_merge($a,$b);
print_r($c);
//echo:
Array
(
    [0] => 1
    [1] => 2
    [2] => 3
)
Array
(
    [0] => 1
    [1] => 2
    [2] => 3
    [3] => 3
    [4] => 4
    [5] => 5
)
```

当2组数组键值为纯数字的时候,出现键值冲突的情况下

* 使用 + : 会返回前一个数组的value
* 使用 array_merge : 返回合并之后的一个数组,且重置键值

###example2 : 键值为字符时
```js
//code:
$a = array(
	'a' => 1,
	'b' => 2,
	'c' => 3
);
$b = array(
	'a' => 11,
	'bb' => 22,
	'c' => 33
);	
print_r($a + $b);
$c = array_merge($a,$b);
print_r($c);
//echo:
Array
(
    [a] => 1
    [b] => 2
    [c] => 3
    [bb] => 22
)
Array
(
    [a] => 11
    [b] => 2
    [c] => 33
    [bb] => 22
)
```

当2组数组键值为字符时

* 使用 + : 返回数组1的value
* 使用 array_merge : 返回数组2的value

###example3 : 键值重置

```js
//code:
	$a = array(
		2 => 'a',
		3 => 'b',
		5 => 'c'
	);
	$b = array(
		3 => 11,
		5 => 22,
		'c' => 33
	);	
	print_r($a + $b);
	$c = array_merge($a,$b);
	print_r($c);
//echo:
Array
(
    [2] => a
    [3] => b
    [5] => c
    [c] => 33
)
Array
(
    [0] => a
    [1] => b
    [2] => c
    [3] => 11
    [4] => 22
    [c] => 33
)
```

* 使用 + : 键值不会重置
* 使用 array_merge : 数字键值会从0开始重置,且返回的合并数组的键值排序是按照先数组1,再数组2的次序来的

####当error_reporting 设置 Notice之上的时候,array_merge函数第一个参数为Null也不会报错,但是确实不会返回任何数据了.
####个人开发本地/测试环境还是把error_reporting设置成E_ALL吧.

岁月蹉跎,遥想当时我来蘑菇街面试的时候,就被问到这个问题了 = , =!!





