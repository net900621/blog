在一个前端群中，一位老大哥风自由提出了一个简单却很奇特的问题：

有一个字符串是这样的：var s = '<html lang="en" class="js no-touch discourse-no-touch">'+ '<head><meta name="csrf-token" content="Oul7WqVh4FBVse2yGeY8ZkqoN5/9/2ImxohJvUYEJYc="/></head><body></body></html>';

怎么通过jquery得到其中content的值？

本来觉得这个问题蛮简单的，既可以通过正则获取，也可以通过jquery的$将这个字符串变成一个jq对象来获取。但是也正是后来这个方法，却带来了问题。

以我的理解，定位content的方法应该为$(s).find("meta").attr("content")，但是这么去获取结果确并不正确。

于是做了简单的尝试：

var s = '<html lang="en" class="js no-touch discourse-no-touch">'+ '<head><meta name="csrf-token" content="Oul7WqVh4FBVse2yGeY8ZkqoN5/9/2ImxohJvUYEJYc="/></head><body></body></html>';
$(s);//[<meta name=​"csrf-token" content=​"Oul7WqVh4FBVse2yGeY8ZkqoN5/​9/​2ImxohJvUYEJYc=">​]
得到的并不是所详细的结果，而是meta标签。

这就有点奇怪了，风大猜测是因为外面有<html></html>，可能jquery不能把<html>标签变为dom。而在字符串外层套一层div则确实能够正确的获取结果。
但是，这个结果真的正确么？
又做了一个简单的尝试：
var s = '<div><html lang="en" class="js no-touch discourse-no-touch">'+
  '<head><meta name="csrf-token" content="Oul7WqVh4FBVse2yGeY8ZkqoN5/9/2ImxohJvUYEJYc="/></head><body></body></html></div>';
$(s);//[<div>​<meta name=​"csrf-token" content=​"Oul7WqVh4FBVse2yGeY8ZkqoN5/​9/​2ImxohJvUYEJYc=">​</div>​]
结果并不是我们所想象的那样，而是仍旧去除了html标签，只不过因为套用了一层div，使$(s).find("meta").attr("content")去正确找寻了节点，并不是保护了html标签。

这个问题的原因，相信只有jq的源码能够给我答案了。写了几行简单的测试代码，去通过断点解读一下jq的处理机制：

(function() {
        debugger;
        $('<html class="123"></html>');
        $('<div></div>');
        $('<html></html>');
    })();
事实证明，除了第一种方式不能正确的获得结果以外，后两种都能正确的获取结果。通过研读jq的源码中对jq对象创建的代码，终于弄清楚了这个原因。

jq对传入的字符串会进行一次正则：

var parsed = rsingleTag.exec( data );//rsingleTag=/^<(\w+)\s*\/?>(?:<\/\1>|)$/
解释一下这个正则吧：

^<(\w+)\s*\/?
以<开头，至少跟着一个字符和任意个空白字符，之后出现0或1次/
>
这个就不说了，前面这些加起来就是<div >这样或者<meta />这样的
(?:<\/\1>|)$
可以匹配<、一个/或者空白并以之为结尾
这些就是</div>或者跟着<br />这种自闭合标签后面的空

这样如果没有任何属性和子节点的字符串（比如'<html></html>'或者'<div></div>'这样）会通过正则的匹配，当通过正则的匹配后则会通过传入的上下文直接创建一个节点：

if ( parsed ) {
            return [ context.createElement( parsed[1] ) ];
        }//context为上下文
而未通过节点的字符串，则通过创建一个div节点，将字符串置入div的innerHTML：

tmp = tmp || safe.appendChild( context.createElement("div") );//创建divdom节点
tmp.innerHTML = wrap[1] + elem.replace( rxhtmlTag, "<$1></$2>" ) + wrap[2];//将字符串置入div对象的innerHTML
而浏览器是不允许div内直接包含<html>的，因此会将html标签过滤。

当过滤了html之后，刚才的字符串变成了：

<head><meta name="csrf-token" content="Oul7WqVh4FBVse2yGeY8ZkqoN5/9/2ImxohJvUYEJYc="/></head><body></body>
jq会从两端向中间去寻找节点的头尾，在这里闭合标签则被寻找为meta标签。

这就是这次所得到的所有结论，总结一下：

如果你的标签是没有子节点和属性的标签，jq会通过正则判定后在你传入的上下文环境直接创建这个节点。
如果你的标签带有子节点或者属性，jq的正则不过，然后会创建一个div，把你的字符串作为div的innerHTML传入，再对div内部的dom节点遍历属性和节点，去获取class啊id啊之类的，然后再创建这个真正的节点。
但是浏览器不允许任何非frame类元素内部有html标签，会将之过滤。
在过滤了html标签后，head标签和body标签就成了两个标签，而jq是从外向内寻找包裹的标签对的，所以这两个标签没有被识别，而meta标签因为是自闭合标签，而且也是从外往内被包裹的最外层闭合标签，因此就成了获取这个标签。
大概就是这样了。
