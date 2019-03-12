


## 自动换行

Settings -> editor -> code style -> JAVA -> Wrapping and Braces -> 勾选 Line breaks 和 Ensure right margin is not exceeded -> 设置 Right margin 数值（每行的字符数），默认为 100，建议设置为 110 或 120。

## bean 类变量自动判 null

主要用于容错处理。
在 bean 类右键鼠标 -> Generate -> Setter -> Template 模板按钮“...” -> “+” 添加新模板 -> 添加成功后选择新模板

模板代码：

```
#if($field.modifierStatic)
static ##
#end
$field.type ##
#set($name = $StringUtil.capitalizeWithJavaBeanConvention($StringUtil.sanitizeJavaIdentifier($helper.getPropertyName($field, $project))))
#if ($field.boolean && $field.primitive)
  #if ($StringUtil.startsWithIgnoreCase($name, 'is'))
    #set($name = $StringUtil.decapitalize($name))
  #else
    is##
#end
#else
  get##
#end
${name}() {
  #if ($field.string)
     return $field.name == null ? "" : $field.name;
  #else 
    #if ($field.list)
    if ($field.name == null) {
        return new ArrayList<>();
    }
    return $field.name;
    #else 
    return $field.name;
    #end
  #end
}
```

## 代码模板

https://mp.weixin.qq.com/s?__biz=MzI3NTYwODgwMA==&mid=2247483726&idx=1&sn=069cca8770f05c235774f77b0430e770&chksm=eb0364c1dc74edd7e6980adca049291140a2f8b365d8895bc6293d3189626b57d2cd52716b6a&mpshare=1&scene=23&srcid=#rd




