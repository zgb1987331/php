####数组最后一个元素的 $value 引用在 foreach 循环之后仍会保留
```
<?php
$arr=[1,2,3,4];
    foreach ($arr as &$val) {
       echo $val;
    }
    // 1，2，3，4
    foreach ($arr as $val) {
        echo $val;
    } 
    //1，2，3，3
?>
```
####第一次foreach循环结束, $val指向的$arr数组的最后位置, 第二次循环改变了最后位置的值。$arr[3] 与 $val 指向同一个变量地址，改变$val的值就会改变$arr[3]的值：

```
第一次循环后 $val = $arr[3] ;   
第二次循环:
$arr[3] = $val = $arr[0]; //1
$arr[3] = $val = $arr[1]; //2
$arr[3] = $val = $arr[2]; //3
$arr[3] = $val = $arr[3]; //3    $arr[3]赋值成3了
```
####建议使用 unset() 来将其销

```
<?php
$arr=[1,2,3,4];
    foreach ($arr as &$val) {
       echo $val;
    }
    // 1，2，3，4
    unset($val);
    foreach ($arr as $val) {
        echo $val;
    } 
    //1，2，3，4
?>
```
#####foreach手册地址:https://www.php.net/manual/zh/control-structures.foreach.php
#####参考地址:https://segmentfault.com/q/1010000007996447