---
layout: post
title:  "JPA로 저장 프로시저(Stored procedure) 호출하기"
date:   2023-07-23 22:30:00 +0900
categories: java
tags: dev, spring-boot, jpa
---

## JPA로 저장 프로시저(스토어드 프로시저, Stored procedure) 호출하기

### 1. 개요

스프링부트 개발 프로젝트에서 데이터베이스의 저장 프로시저(Stored procedure) 호출이 요구되어 방법을 구현해 보았다.

여러 가이드에서 JPA에서 프로시저를 호출하려면 `@Entity`클래스에 `@NamedStoredProcedureQuery`를 선언하거나 `Repository` 인터페이스에 `@Procedure` 메서드를 정의하도록 설명하고 있다.
- [Calling Stored Procedures from Spring Data JPA Repositories](https://www.baeldung.com/spring-data-jpa-stored-procedures)
- [[JPA] 저장 프로시저(Stored Procedure) by yu-jin-song](https://velog.io/@yu-jin-song/JPA-%EC%A0%80%EC%9E%A5-%ED%94%84%EB%A1%9C%EC%8B%9C%EC%A0%80Stored-Procedure)

그러나 저장 프로시저는 여러 테이블을 다루거나 테이블과 무관한 필드가 함께 선언되어 있는 경우가 있어 특정 엔티티 또는 레파지터리에 연결하기가 난처해서 별도로 작성하고 싶은 경우가 있다.

프로시저의 입출력 항목에 맞는 가상의 엔티티 클래스를 구현하고 스토어드 프로시저 어노테이션으로 추가하는 방법도 있으나, 가상 엔티티 작성으로 인해 예상치 못한 이슈가 생길 가능성도 있다.
(예를 들면 엔티티 클래스 정보를 읽고 데이터베이스의 메타 정보와 비교해서 잘못 사용되는 컬럼을 찾는 점검 프로그램이 있다면 오류가 발생할 것이다.)

또한 `@NamedStoredProcedureQuery` 사용법을 보면 어노테이션 안에 파라마터별로 어노테이션을 선언하지만 파라마터가 클래스에 선언한 필드와 매치되지 않기에
메서드 파라마터로 `@Param("model_in")` 어노테이션을 붙이거나, 코드에서 `EntityManager`를 주입하고 `createNamedStoredProcedureQuery()` 메서드를 통해
쿼리 객체를 받아서 파라마터를 컨트롤하는 코드를 작성해야 한다.
특히 두번째 방법은 오래 전 `java.sql.Connection`을 받고 `preparedStatement()`를 만들어서 사용하는 방법과 다를 게 없다.

이 어려움을 개선하기 위해 어노테이션과 DTO 클래스를 통해서 프로시저를 호출하는 방법을 구현해 보았다.

아래 코드는 모두 Public domain으로, 자유롭게 가져다 써도 Ok. Spring boot 버전은 3.0

### 2. 어노테이션 정의

`@NamedStoredProcedureQuery` 어노테이션을 대체할 어노테이션을 작성한다.

```java
import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import static java.lang.annotation.ElementType.TYPE;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

@Documented
@Target(TYPE)
@Retention(RUNTIME)
public @interface StoredProcedureSupport {
    /**
     * 데이터베이스의 프로시저 이름
     */
    String name() default "";
}
```

`@StoredProcedureParameter`를 어노테이션을 대체할 어노테이션을 작성한다. `@StoredProcedureParameter`어노테이션을 사용해도 괜찮지만, 나는 파라마터 순서 필드를 추가하기 위해 새로 작성했다.

```java
import jakarta.persistence.ParameterMode;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import static java.lang.annotation.RetentionPolicy.RUNTIME;

/**
 * 데이터베이스 저장 프로시저의 파라마터 정의
 * order 항목 추가.
 * 필드에만 사용 가능. (메서드 불가)
 *
 * @see jakarta.persistence.StoredProcedureParameter
 */
@Documented
@Target(ElementType.FIELD)
@Retention(RUNTIME)
public @interface StoredProcedureParam {

    /**
     *  The name of the parameter as defined by the stored procedure in the database.
     *  If a name is not specified, it is assumed that the stored procedure uses
     *  positional parameters.
     */
    String name();

    /**
     * parameter order.
     * Database ordered
     */
    int order();

    /**
     *  Specifies whether the parameter is an IN, INOUT, OUT, or REF_CURSOR parameter.
     *  REF_CURSOR parameters are used by some databases to return result sets from
     *  a stored procedure.
     */
    ParameterMode mode() default ParameterMode.IN;

    /** JDBC type of the paramter. */
    Class type();

}
```

### 2. 실행 클래스 정의

Runtime 어노테이션을 정의했으면 이를 읽고 실행할 코드가 필요하다.

```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.ParameterMode;
import jakarta.persistence.StoredProcedureQuery;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.util.ReflectionUtils;

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;

@Slf4j
@RequiredArgsConstructor
public class StoredProcedureSupportExecutor {
    private final EntityManager entityManager;

    public boolean execute(Object parameter) {
        var clazz = parameter.getClass();
        // 클래스 어노텐이션
        var supportAnnotation = getSupportAnnotation(clazz);
        // 파라마터 어노테이션
        var paramFields = getStoredProcedureParamFields(clazz);

        log.debug("프로시저 쿼리 생성 - {}", supportAnnotation.name());
        var query = entityManager.createStoredProcedureQuery(supportAnnotation.name());

        // 파라마터 셋
        setRequestParameter(query, paramFields, parameter);

        boolean result = query.execute();
        log.debug("프로시저 실행 결과 - {}", result);

        // OUT 파라마터 처리
        setResponseOutParameter(query, paramFields, parameter);

        return result;
    }

    private void setRequestParameter(StoredProcedureQuery query, List<Field> paramFields, Object parameter) {
        for (var field : paramFields) {
            var paramAn = field.getAnnotation(StoredProcedureParam.class);

            log.debug("파라마터 Info  - {}. {} {} {}", paramAn.order(), paramAn.name(), paramAn.mode().name(), paramAn.type().getSimpleName());
            query.registerStoredProcedureParameter(paramAn.name(), paramAn.type(), paramAn.mode());

            if (paramAn.mode() == ParameterMode.OUT) {
                continue;
            }

            if (log.isDebugEnabled() && !log.isTraceEnabled()) {
                log.debug("파라마터 Value - {}. {}", paramAn.order(), paramAn.name());
            }

            Object value;
            try {
                value = field.get(parameter);
            } catch (IllegalAccessException e) {
                log.error("필드값 접근 오류 - {}", field.getName(),  e);
                throw new RuntimeException(e);
            }

            if (log.isTraceEnabled()) {
                log.trace("파라마터 Value - {}. {} {}", paramAn.order(), paramAn.name(), value);
            }
            query.setParameter(paramAn.name(), value);
        }
    }

    private void setResponseOutParameter(StoredProcedureQuery query, List<Field> paramFields, Object parameter) {
        for (var field : paramFields) {
            var paramAn = field.getAnnotation(StoredProcedureParam.class);
            if (paramAn.mode() == ParameterMode.IN) {
                continue;
            }

            Object value = query.getOutputParameterValue(paramAn.name());
            if (log.isTraceEnabled()) {
                log.trace("파라마터 Ret   - {}. {} {}", paramAn.order(), paramAn.name(), value);
            }

            try {
                field.set(parameter, value);
            } catch (IllegalAccessException e) {
                log.error("필드값 접근 오류 - {}", field.getName(),  e);
                throw new RuntimeException(e);
            }
        }
    }

    private StoredProcedureSupport getSupportAnnotation(Class<?> clazz) {
        var typeAn = clazz.getAnnotation(StoredProcedureSupport.class);
        if (typeAn == null) {
            var clsName = clazz.getName();
            log.error("클래스에 @StoredProcedureSupport 어노테이션이 정의되지 않았습니다. - {}", clsName);
            throw new RuntimeException("Cannot find annotation class.");
        }
        return typeAn;
    }

    private List<Field> getStoredProcedureParamFields(Class<?> clazz) {
        final List<Field> paramInfo = new ArrayList<>();
        ReflectionUtils.doWithFields(clazz, field -> {
            var paramAn = field.getAnnotation(StoredProcedureParam.class);
            if (paramAn == null) {
                return;
            }
            field.setAccessible(true);
            paramInfo.add(field);
        });

        // StoredProcedureParam 어노테이션의 order 값 순서대로 정렬한다.
        return paramInfo.stream().sorted(Comparator.comparingInt(o -> o.getAnnotation(StoredProcedureParam.class).order())).toList();
    }

}
```

### 3. Configuration 클래스 정의

위에서 정의한 `StoredProcedureSupportExecutor` 클래스에 `@Component`어노테이션을 추가하면 빈으로 등록되어 다른 코드에 주입되어 사용할 수 있다.

내 경우 프로젝트의 요구사항에 다중 데이터베이스 컨트롤이 있어서 이를 지원하기 위해 스프링 부트 Configuration bean으로 분리하였다.
요구사항이 단일 데이터베이스인 경우 아래 코드를 생략하고 `@Component`어노테이션으로 붙이거나 아래 코드에서 `@Qualifier("엔티티매니저 이름")`을 삭제하면 된다.

```java
import com.pb.slip.annotation.StoredProcedureSupportExecutor;
import jakarta.persistence.EntityManager;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
//@RequiredArgsConstructor // 다중 데이터베이스가 필요하여 사용하지 않음
public class StoredProcedureSupportConfiguration {
    private final EntityManager appEntityManager;

    public StoredProcedureSupportConfiguration(@Qualifier("엔티티매니저 이름") EntityManager appEntityManager) {
        this.erpEntityManager = erpEntityManager;
    }

    @Bean
    public StoredProcedureSupportExecutor appStoredProcedureSupportExecutor() {
        return new StoredProcedureSupportExecutor(erpEntityManager);
    }
}
```

> JPA 다중 데이터베이스 참고: [Spring JPA – Multiple Databases](https://www.baeldung.com/spring-data-jpa-multiple-databases)
>
> `@Qualifier("엔티티매니저 이름")`은 `LocalContainerEntityManagerFactoryBean`를 생성하여 리턴하는 메소드의 이름을 사용한다.
> 
> 예를 들어 `public LocalContainerEntityManagerFactoryBean userEntityManager()` 데이터베이스를 사용하려면 `@Qualifier("userEntityManager")`로,
> `public LocalContainerEntityManagerFactoryBean productEntityManager()` 데이터베이스를 사용하려면 `@Qualifier("productEntityManager")`를 사용한다.
> 
> 둘 다 필요하면 둘 다 정의하고, 두개의 `public StoredProcedureSupportExecutor 메소드명()`을 작성한다.


### 4. 프로시저 파라마터 DTO 정의

프로시저의 파라마터 구성에 맞는 DTO를 정의한다.

클래스 선언에는 `@StoredProcedureSupport(name="호출할 프로시저 이름")`을 정의하고, 프로시저의 파라마터에 맞춰서 필드 및 `@StoredProcedureParam`을 정의한다.

`@StoredProcedureParameter`와 다른 점은 프로시저의 파라마터 순서를 추가하였다.

```java
//import xx.xx.StoredProcedureSupport
import jakarta.persistence.ParameterMode;
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;

@Getter
@Setter
@Builder
@ToString
@StoredProcedureSupport(name = "DB_PROCEDURE_NAME")
public class ProcParam {

    @StoredProcedureParam(order = 1, name = "o_out1", type = String.class, mode = ParameterMode.OUT)
    private String outCode;

    @StoredProcedureParam(order = 2, name = "o_out2", type = String.class, mode = ParameterMode.OUT)
    private String outMsg;

    @StoredProcedureParam(order = 3, name = "p_user_id", type = String.class, mode = ParameterMode.IN)
    private String userId;

    @StoredProcedureParam(order = 4, name = "p_user_name", type = String.class, mode = ParameterMode.IN)
    private String userName;

    @StoredProcedureParam(order = 5, name = "o_user_seq", type = Long.class, mode = ParameterMode.OUT)
    private Long userSeq;
}
```

### 5. 서비스에서 사용하기

서비스 클래스에서는 `StoredProcedureSupportExecutor`를 주입받고 `execute()` 메서드를 호출하도록 구현한다.

```java
private final StoredProcedureSupportExecutor appStoredProcedureSupportExecutor;

...

public Long callStoredProceduer(String userId, String userName) {

    var param = ProcParam.builder().userId(userId).userName(userName).build();

    var result = appStoredProcedureSupportExecutor.execute(param);
    // 프로시저가 오류를 내는 경우 일반적으로 result = false
    // 그러나 개발 사정에 따라 오류가 발생하더라도 success 로 응답하는 경우도 있으므로 result 는 프로시저 확인해 보고 판단하는게 좋음.

    // result 변수와 무관하게 out 값은 채워져 있음
    log.debug("call result {}, {}, {}, {}", result, param.getOutCode(), param.getOutMsg(), param.getUserSeq());

    return param.getUserSeq();
}
```

코드를 실행하면 `@StoredProcedureSupport(name="")`에 정의한 프로시저를 호출하는 것을 로그를 통해 확인할 수 있다.

> JPA 로그를 보기 위해서는 어플리케이션 설정에 `logging.level.org.hibernate.SQL=DEBUG`를 설정한다.

### 6. 결론

프로시저 호출은 `@Entity` 또는 `Repository`인터페이스와 완전히 분리되었으며, 파라마터 또한 코드화 되어 자동으로 처리하므로 유지보수성도 나아졌다고 생각한다.

끝.
