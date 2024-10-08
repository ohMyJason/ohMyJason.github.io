---
layout:     post
title:      "轮子工程|基于注解式的参数检查"
subtitle:   "永远不要信任前端的传参"
date:       2024-10-8 16:58:06
author:     "Hux"
header-img: "img/post-bg-alibaba.jpg"
tags:
    - 轮子
    - Java
---

```java
package com.hikvision.fj.util;

import com.hikvision.fj.entity.param.ApprovalAppointmentWebParam;
import io.swagger.annotations.ApiModelProperty;
import org.apache.commons.lang3.StringUtils;
import org.springframework.util.CollectionUtils;

import javax.validation.constraints.Max;
import javax.validation.constraints.Min;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.*;
import java.util.function.Function;
import java.util.function.Supplier;
import java.util.stream.Collectors;

/**
 * @Author: liujiasheng5
 * @Date: 2024/9/20 11:36
 */
public class ParamCheck {

    /**
     * 检查参数是否合法
     *
     * @param param 传入的对象
     * @throws IllegalArgumentException 如果校验不通过，抛出异常
     */
    public static void paramsCheck(Object param) throws IllegalArgumentException {
        if (param == null) {
            throw new IllegalArgumentException("参数对象不能为空");
        }

        // 递归检查对象字段
        checkFields(param, param.getClass());
    }

    public static void paramsCheckPrimitive(Object param, String paramName) throws IllegalArgumentException {
        if (param == null) {
            throw new IllegalArgumentException(paramName + "不能为空");
        } else if (param instanceof String && StringUtils.isBlank(paramName)) {
            throw new IllegalArgumentException(paramName + "不能为空字符串");
        }
    }

    /**
     * 递归检查对象及其父类的字段
     *
     * @param obj   需要校验的对象
     * @param clazz 对象的类
     * @throws IllegalArgumentException 如果校验不通过，抛出异常
     */
    private static void checkFields(Object obj, Class<?> clazz) throws IllegalArgumentException {
        // 如果类是Object，则停止递归
        if (clazz == Object.class) {
            return;
        }

        // 获取类的所有字段，包括私有字段
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            // 忽略静态字段和final字段
            if (Modifier.isStatic(field.getModifiers()) || Modifier.isFinal(field.getModifiers())) {
                continue;
            }

            // 允许访问私有字段
            field.setAccessible(true);

            try {
                // 获取字段的值
                Object fieldValue = field.get(obj);

                // 检查@ApiModelProperty注解
                ApiModelProperty apiModelProperty = field.getAnnotation(ApiModelProperty.class);
                if (apiModelProperty == null) {
                    continue;
                }
                if (apiModelProperty.required()) {
                    if (fieldValue == null) {
                        throw new IllegalArgumentException(getFieldName(field, apiModelProperty) + "不能为空");
                    } else if (fieldValue instanceof String && ((String) fieldValue).isEmpty()) {
                        throw new IllegalArgumentException(getFieldName(field, apiModelProperty) + "不能为空字符串");
                    } else if (fieldValue instanceof Number) {
                        // 检查@Min和@Max注解
                        Min minAnnotation = field.getAnnotation(Min.class);
                        Max maxAnnotation = field.getAnnotation(Max.class);
                        long numberValue = ((Number) fieldValue).longValue();
                        if (minAnnotation != null && numberValue < minAnnotation.value()) {
                            throw new IllegalArgumentException(getFieldName(field, apiModelProperty) + "不能小于" + minAnnotation.value());
                        }
                        if (maxAnnotation != null && numberValue > maxAnnotation.value()) {
                            throw new IllegalArgumentException(getFieldName(field, apiModelProperty) + "不能大于" + maxAnnotation.value());
                        }
                    }
                    Limit limitAnnotation = field.getAnnotation(Limit.class);
                    if (limitAnnotation != null) {
//                        if (limitAnnotation.type() == int.class || limitAnnotation.type() == Integer.class) {
//                            Set<Integer> collect = Arrays.stream(limitAnnotation.value()).map(Integer::valueOf).collect(Collectors.toSet());
//                            if (!collect.contains(fieldValue)) {
//                                throw new IllegalArgumentException(getFieldName(field, apiModelProperty) + "只能在如下范围:" + getLimitMsg(collect));
//                            }
//                        }
//                        if (limitAnnotation.type() == String.class) {
//                            Set<String> collect = Arrays.stream(limitAnnotation.value()).collect(Collectors.toSet());
//                            if (!collect.contains(fieldValue)) {
//                                throw new IllegalArgumentException(getFieldName(field, apiModelProperty) + "只能在如下范围:" + getLimitMsg(collect));
//                            }
//                        }
                        Set<String> collect = Arrays.stream(limitAnnotation.value()).collect(Collectors.toSet());
                        if (!collect.contains("" + fieldValue)) {
                            throw new IllegalArgumentException(getFieldName(field, apiModelProperty) + "只能在如下范围:" + getLimitMsg(collect));
                        }
                    }
                }


                // 处理嵌套的自定义对象，递归调用检查
                if (fieldValue != null && !isPrimitiveOrWrapperOrString(field.getType())) {
                    checkFields(fieldValue, fieldValue.getClass());
                }

            } catch (IllegalAccessException e) {
                throw new RuntimeException("无法访问字段：" + field.getName(), e);
            }
        }

        // 递归检查父类的字段
        checkFields(obj, clazz.getSuperclass());
    }

    /**
     * 判断字段是否是基本类型、包装类或字符串
     *
     * @param clazz 字段类型
     * @return 如果是基本类型或包装类或字符串，返回true；否则返回false
     */
    private static boolean isPrimitiveOrWrapperOrString(Class<?> clazz) {
        return clazz.isPrimitive() ||
                clazz == Integer.class ||
                clazz == Long.class ||
                clazz == Double.class ||
                clazz == Float.class ||
                clazz == Boolean.class ||
                clazz == Byte.class ||
                clazz == Character.class ||
                clazz == Short.class ||
                clazz == String.class ||
                clazz == Date.class;
    }

    private static String getLimitMsg(Set limit) {
        StringBuilder s = new StringBuilder();
        s.append("[");
        for (Object o : limit) {
            s.append(o).append(",");
        }
        if (s.charAt(s.length() - 1) == ',') {
            s.deleteCharAt(s.length() - 1);
        }
        s.append("]");
        return s.toString();
    }

    /**
     * 获取@ApiModelProperty的字段名称，如果没有name属性，则使用字段的名称
     *
     * @param field            需要检查的字段
     * @param apiModelProperty 字段上的@ApiModelProperty注解
     * @return 返回注解中的name值或字段名称
     */
    private static String getFieldName(Field field, ApiModelProperty apiModelProperty) {
        if (apiModelProperty != null && StringUtils.isNotBlank(apiModelProperty.value())) {
            return apiModelProperty.value();
        }
        return field.getName();
    }


    //------------------------------------------


    /**
     * 多转1工具
     *
     * @param list
     * @param nullTask
     * @param outOfOneTask
     * @param <T>
     * @return
     */
    public static <T> T listToOne(List<T> list, Runnable nullTask, Runnable outOfOneTask) {
        if (CollectionUtils.isEmpty(list) || list.get(0) == null) {
            Optional.ofNullable(nullTask).orElse(() -> {
            }).run();
        } else if (list.size() > 1) {
            Optional.ofNullable(outOfOneTask).orElse(() -> {
            }).run();
        } else {
            return list.get(0);
        }
        return null;
    }

    //TEST
    public static void main(String[] args) {
        paramsCheck(ApprovalAppointmentWebParam.builder().approvalAction(2).applyId("1").build());

//        System.out.println(getLimitMsg(new HashSet() {{
//            add(1);
//            add(2);
//            add(3);
//        }}));
    }


    /**
     * 只支持int和String类型
     */
    @Target({ElementType.FIELD})
    @Retention(RetentionPolicy.RUNTIME)
    public @interface Limit {
        String[] value();

    }

}

```


```java
    @ParamCheck.Limit({"1", "3"})
    @ApiModelProperty(value = "审批动作 1:通过,3:退回 ", required = true)
    Integer approvalAction;
```
