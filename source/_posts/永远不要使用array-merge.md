---
title: 永远不要使用array_merge
date: 2018-12-12 17:02:15
tags: php
---

array_merge是php中的一个常用函数，我在之前也经常使用这个函数，知道有一天我发现我的程序运行的比预想中要慢的多，经过逐行排查发现竟然是array_merge函数惹得祸

```php
$primes = array();
while (xxx)
{
    $data = get_some_data();
	$primes = array_merge($primes, $data);
}
```

当你使用类似上面的函数去不断的合并数组，尤其是最终结果较大时你会发现时间变得不可忍受。

解决的方法也很简单

```php
$primes = array();
while (xxx)
{
    $data = get_some_data();
    foreach($data as $val) {
        $primes[] = $val;
    }
}
```

差别大到你无法想象。