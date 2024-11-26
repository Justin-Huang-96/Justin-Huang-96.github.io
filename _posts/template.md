---
layout: post
title: template page
categories: [ cate1, cate2 ]
description: some word here
keywords: keyword1, keyword2
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# 动态调用Spring任意Bean的任意public方法

## 核心思路

动态调用方法时，需要解决两个核心问题：

1. 找到目标方法：根据方法名和参数个数定位目标方法。
2. 解析参数类型：确保将请求中的参数转换为目标方法所需的实际类型（包括普通类型和复杂泛型类型）。

## 核心逻辑解析

1. 获取目标方法
   我们需要动态获取指定的 Bean，然后通过反射获取其方法。代码中的逻辑如下：
   Method method = getMethodWithParameters(bean, request.getMethodName(), request.getParams());
   ● bean：通过 Spring 的工具类 SpringUtil.getBean(request.getBeanName()) 动态获取。
   ● request.getMethodName()：方法名称。
   ● request.getParams()：参数列表，判断方法是否匹配参数数量。
   方法定位逻辑

private Method getMethodWithParameters(Object bean, String methodName, Object[] params) {
Method[] methods = bean.getClass().getMethods();
for (Method method : methods) {
if (method.getName().equals(methodName) &&
method.getParameterCount() == (params != null ? params.length : 0)) {
return method;
}
}
return null;
}
● 遍历类的所有方法。
● 匹配方法名，并根据参数数量过滤方法。

2. 参数解析逻辑
   核心问题：请求参数是 JSON 数据，目标方法的参数类型可能是泛型集合或复杂对象，必须进行类型转换。
   代码核心：

```java


Object[] paramValues = parseParams(request.getParams(), method);

//参数解析方法
private Object[] parseParams(Object[] params, Method method) throws Exception {
    if (params == null || method.getParameterCount() == 0) {
        return new Object[0]; // 无参数
    }

    Object[] parsedParams = new Object[params.length];
    Type[] genericParameterTypes = method.getGenericParameterTypes(); // 获取每个参数的完整类型（包括泛型）

    for (int i = 0; i < params.length; i++) {
        Type genericType = genericParameterTypes[i];

        if (genericType instanceof ParameterizedType) {
            // 处理泛型类型（如 List<ItemCreateVO>）
            ParameterizedType parameterizedType = (ParameterizedType) genericType;
            Class<?> rawType = (Class<?>) parameterizedType.getRawType(); // 原始类型（List）
            Type[] actualTypeArguments = parameterizedType.getActualTypeArguments(); // 泛型类型（ItemCreateVO）
            if (Collection.class.isAssignableFrom(rawType) && actualTypeArguments.length > 0) {
                Class<?> itemType = (Class<?>) actualTypeArguments[0]; // 获取集合元素的类型
                CollectionType collectionType = objectMapper.getTypeFactory()
                        .constructCollectionType((Class<? extends Collection>) rawType, itemType);
                parsedParams[i] = objectMapper.convertValue(params[i], collectionType);
            } else {
                throw new UnsupportedOperationException("Unsupported generic type: " + genericType.getTypeName());
            }
        } else {
            // 非泛型参数直接转换
            parsedParams[i] = objectMapper.convertValue(params[i], (Class<?>) genericType);
        }
    }
    return parsedParams;
}
```

解析逻辑详解

1. 获取参数的完整类型：
   ○ 使用 method.getGenericParameterTypes() 获取目标方法的参数类型，包括泛型信息。
   ○ 普通类型直接通过 Class<?> 转换；泛型类型（如 List<ItemCreateVO>）需要特殊处理。
2. 处理泛型参数：
   ○ 判断类型是否是 ParameterizedType。
   ○ 获取原始类型（如 List）和泛型类型（如 ItemCreateVO）。
   ○ 使用 Jackson 的 CollectionType 来构造类型映射，完成 JSON 转对象的工作。
3. 处理普通参数：
   ○ 直接通过 Jackson 的 convertValue 方法完成类型转换。

3. 调用目标方法
   目标方法的参数解析完成后，使用反射调用：

return method.invoke(bean, paramValues);
● method 是目标方法对象。
● bean 是目标对象。
● paramValues 是解析后的参数。

示例解析
假设调用目标方法 createItem：

```java
public Boolean createItem(String tenantId, List<ItemCreateVO> itemCreateVOList) {
    System.out.println("Tenant ID: " + tenantId);
    System.out.println("Items: " + itemCreateVOList);
    return true;
}
```

请求 JSON：

{
"beanName": "itemHttp",
"methodName": "createItem",
"params": [
"SBFH",
[
{
"sku": "iphone16",
"customerId": "test_customerId",
"uom": "g",
"weight": "200",
"length": "1",
"width": "1",
"height": "1"
}
]
]
}
执行流程：

1. 定位方法：
   ○ beanName：动态获取 itemHttp Bean。
   ○ methodName：匹配方法 createItem。
   ○ 参数数量：匹配参数数量为 2。
2. 解析参数：
   ○ 第一个参数为普通类型 String，直接转换为 "SBFH"。
   ○ 第二个参数为 List<ItemCreateVO>：
   ■ 原始类型：List。
   ■ 泛型类型：ItemCreateVO。
   ■ JSON 数据转换为 List<ItemCreateVO>。
3. 调用方法： 使用反射调用 createItem，传入解析后的参数。

输出结果
控制台打印：

Tenant ID: SBFH
Items: [ItemCreateVO{sku='iphone16', customerId='test_customerId', uom='g', weight='200', length='1', width='1', height='1'}]
目标方法成功被调用，List<ItemCreateVO> 被正确解析。

核心点总结

1. 方法定位：通过反射动态获取指定的方法。
2. 参数解析：区分普通参数和泛型参数，使用 Jackson 动态构造目标类型。
3. 反射调用：通过 Method.invoke 实现动态调用，传入解析后的参数。
