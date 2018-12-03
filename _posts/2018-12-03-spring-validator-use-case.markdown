---
layout:     post
title:      "自定义Spring validator（跨属性校验）应用实例"
subtitle:   "如何自定义Spring validator（跨属性校验）?"
date:       2018-12-03 12:00:00
author:     "王磊磊"
header-img: "img/home-bg-o.jpg"
tags:
    - Web
    - Spring
---

最近使用spring开发API，涉及到很多跨属性校验的问题，每次都写逻辑判断（如：判断两个日期属性，申请日期必须早于获取日期）。研究了下Spring 的validator，通过自定义注解实现功能。
<br>
<br>自定义Validator
```java
package com.test.web.validator;

import com.test.web.aop.annotation.DateFieldValidation;
import java.util.Date;
import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.beanutils.PropertyUtils;

/**
 * 两个日期类型属性检验：first must before second
 *
 * @author wade0817wang
 */
@Slf4j
public class DateFieldMatchValidator implements ConstraintValidator<DateFieldValidation, Object> {

  private String first;

  private String second;

  private String errMsg;

  @Override
  public boolean isValid(Object value, ConstraintValidatorContext context) {
    boolean toReturn = true;
    try {
      final Object startDate = PropertyUtils.getProperty(value, first);
      final Object endDate = PropertyUtils.getProperty(value, second);
      if (startDate != null && endDate != null && ((Date) endDate).before((Date) startDate)) {
        toReturn = false;
      }
    } catch (Exception e) {
      log.error("[DateFieldMatchValidator] execute error");
    }
    if (!toReturn) {
      context.disableDefaultConstraintViolation();
      context.buildConstraintViolationWithTemplate(errMsg).addPropertyNode(second).addConstraintViolation();
    }
    return toReturn;
  }

  @Override
  public void initialize(DateFieldValidation constraintAnnotation) {
    this.first = constraintAnnotation.first();
    this.second = constraintAnnotation.second();
    this.errMsg = constraintAnnotation.message();
  }
}

```
<br>
<br>自定义校验注解
```java
package com.test.web.aop.annotation;

import com.test.web.validator.DateFieldMatchValidator;
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import javax.validation.Constraint;
import javax.validation.Payload;

/**
 * 跨属性校验：第一个日期属性必须早于第二个日期属性
 * @author wade0817wang
 */
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = DateFieldMatchValidator.class)
@Documented
public @interface DateFieldValidation {

  /**
   * 第一个日期属性名称
   */
  String first();

  /**
   * 第二个日期属性名称
   */
  String second();

  String message() default "校验失败";
  Class<?>[] groups() default {};
  Class<? extends Payload>[] payload() default {};


  /**
   * Defines several <code>@FieldMatch</code> annotations on the same element
   *
   * @see DateFieldValidation
   */
  @Target({ElementType.TYPE, ElementType.ANNOTATION_TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  public @interface List{
    DateFieldValidation[] value();
  }
}

```
<br>
<br>实体DTO

```java
package com.test.web.dto.company;

import com.test.web.aop.annotation.DateFieldValidation;
import com.test.web.dto.common.BaseDTO;
import java.util.Date;
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;
import org.hibernate.validator.constraints.Length;

/**
 * Dto
 *
 * @author wade0817wang
 */
@Getter
@Setter
@ToString
@DateFieldValidation(first = "apply_date", second = "release_date", message = "发布日期必须在申请日期之后")
public class ObjectDTO extends BaseDTO {

  /**
   * id
   */
  private Integer id;

  /**
   * 名称
   */
  @Length(max = 50, message = "名称不能超过50个字符")
  private String name;
  
  /**
   * 发布日期
   */
  private Date release_date;

  /**
   * 申请日期
   */
  private Date apply_date;
}

```

<br>
<br>测试结果
```json
{
  "error":[
    "名称不能超过50个字符",
    "发布日期必须在申请日期之后"
  ]
}
```



<br>注意：在定义DateFieldMatchValidator时， 一定要添加代码：
```java
    if (!toReturn) {
      //设置默认的校验规则失效
      context.disableDefaultConstraintViolation();
      //设置新的Vilation模板，设置相关message，设置属性名
      context.buildConstraintViolationWithTemplate(errMsg).addPropertyNode(second).addConstraintViolation();
    }
```
<br>否则校验代码可以正常执行，但是无法打印错误信息。







