---
title: "QueryDSL: 문자열이 아닌 코드로 쿼리 작성하기"
date: 2026-03-19 21:00:00 +0900
categories: [spring]
tags: [spring, querydsl, jpa, java, dynamic-query]
---

# QueryDSL: 문자열이 아닌 코드로 쿼리 작성하기 🧑‍💻

JPA와 JPQL을 사용하다 보면 언젠가 한 번은 반드시 마주치게 되는 한계가 있습니다. 바로 **타입 불안정성**과 **동적 쿼리의 복잡성**입니다. 이 글에서는 이 두 가지 문제를 우아하게 해결해 주는 **QueryDSL**에 대해 처음부터 단계별로 정리합니다.

---

## 1. 왜 QueryDSL이 필요한가? (기존 방식의 문제점)

### 문제점 1: 타입 불안정성 (Not Type-Safe)

JPQL은 결국 **문자열(String)**입니다. 따라서 컴파일 시점에는 쿼리의 문법적 오류나 오타를 전혀 잡을 수 없고, 애플리케이션을 실행하고 해당 쿼리가 호출된 후에야 런타임 에러를 마주하게 됩니다.

```java
// "username"을 "usrname"으로 오타를 내도 컴파일은 성공한다.
@Query("SELECT u FROM User u WHERE u.usrname = :name")
List<User> findByName(String name);
```

위 코드는 컴파일도 정상적으로 되고, IDE에서도 아무런 경고가 뜨지 않습니다. 실제로 메서드가 호출되는 순간에야 `No field named usrname` 에러가 발생하며 런타임에 장애를 일으킵니다.

### 문제점 2: 동적 쿼리의 고통

검색 조건이 다양해지는 '동적 쿼리'를 작성할 때, JPQL 문자열을 조건에 따라 `if`문으로 조합하는 방식은 코드를 매우 복잡하고 지저분하게 만듭니다.

```java
// 동적 쿼리를 JPQL 문자열로 처리하는 방식 - 매우 지저분하다
public List<User> searchUsers(String name, Integer age) {
    String jpql = "SELECT u FROM User u WHERE 1=1";
        if (name != null) {
                jpql += " AND u.username = :name";
                    }
                        if (age != null) {
                                jpql += " AND u.age = :age";
                                    }
                                        // ...
                                        }
                                        ```

                                        조건이 늘어날수록 문자열이 복잡해지고, 실수가 생기기 쉬우며 유지보수가 매우 어려워집니다.

                                        ---

                                        ## 2. QueryDSL이란 무엇이며, 어떻게 문제를 해결하는가?

                                        **QueryDSL**은 타입 안전(Type-Safe)한 쿼리를 자바 코드로 작성할 수 있게 해주는 프레임워크입니다. 엔티티 클래스를 기반으로 **Q타입 클래스**를 자동 생성하고, 이 Q타입을 이용해 쿼리를 문자열이 아닌 메서드 체이닝 방식으로 작성합니다.

                                        ```java
                                        // QueryDSL 방식 - 타입 안전, 오타 시 컴파일 에러 발생
                                        QUser user = QUser.user;
                                        List<User> result = queryFactory
                                            .selectFrom(user)
                                                .where(user.username.eq(name)) // 오타 시 컴파일 에러!
                                                    .fetch();
                                                    ```

                                                    핵심 장점은 다음과 같습니다.

                                                    - **컴파일 타임 오류 감지**: 필드명 오타나 타입 불일치를 IDE와 컴파일러가 즉시 잡아줍니다.
                                                    - **동적 쿼리의 간결한 표현**: `BooleanBuilder`나 `BooleanExpression`을 활용해 동적 조건을 깔끔하게 조합할 수 있습니다.
                                                    - **IDE 자동완성 지원**: Q타입 클래스 덕분에 필드명을 자동완성으로 입력할 수 있습니다.

                                                    ---

                                                    ## 3. QueryDSL 기본 설정 (build.gradle)

                                                    QueryDSL을 사용하려면 `build.gradle`에 의존성과 Q타입 생성 설정을 추가해야 합니다.

                                                    ```groovy
                                                    dependencies {
                                                        // QueryDSL
                                                            implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
                                                                annotationProcessor 'com.querydsl:querydsl-apt:5.0.0:jakarta'
                                                                    annotationProcessor 'jakarta.annotation:jakarta.annotation-api'
                                                                        annotationProcessor 'jakarta.persistence:jakarta.persistence-api'
                                                                        }

                                                                        // Q타입 생성 경로 설정
                                                                        def generated = 'src/main/generated'
                                                                        sourceSets {
                                                                            main.java.srcDirs += [ generated ]
                                                                            }
                                                                            tasks.withType(JavaCompile) {
                                                                                options.getGeneratedSourceOutputDirectory().set(file(generated))
                                                                                }
                                                                                clean {
                                                                                    delete file(generated)
                                                                                    }
                                                                                    ```

                                                                                    설정 후 `./gradlew compileJava`를 실행하면 `src/main/generated` 경로에 `QUser`, `QProduct` 등 Q타입 클래스들이 자동으로 생성됩니다.

                                                                                    ---

                                                                                    ## 4. QueryDSL을 활용한 동적 쿼리 작성법 🔧

                                                                                    ### 준비 단계: JPAQueryFactory Bean 등록

                                                                                    QueryDSL의 핵심 객체인 `JPAQueryFactory`를 스프링 빈으로 등록합니다.

                                                                                    ```java
                                                                                    @Configuration
                                                                                    public class QueryDslConfig {

                                                                                        @PersistenceContext
                                                                                            private EntityManager entityManager;

                                                                                                @Bean
                                                                                                    public JPAQueryFactory jpaQueryFactory() {
                                                                                                            return new JPAQueryFactory(entityManager);
                                                                                                                }
                                                                                                                }
                                                                                                                ```

                                                                                                                ### 동적 쿼리 작성의 방법

                                                                                                                `BooleanExpression`을 반환하는 조건 메서드를 분리하면, 동적 쿼리를 매우 깔끔하게 작성할 수 있습니다.

                                                                                                                ```java
                                                                                                                @Repository
                                                                                                                @RequiredArgsConstructor
                                                                                                                public class UserRepositoryImpl {

                                                                                                                    private final JPAQueryFactory queryFactory;

                                                                                                                        public List<User> searchUsers(String name, Integer age) {
                                                                                                                                QUser user = QUser.user;
                                                                                                                                        return queryFactory
                                                                                                                                                    .selectFrom(user)
                                                                                                                                                                .where(
                                                                                                                                                                                usernameEq(name),
                                                                                                                                                                                                ageEq(age)
                                                                                                                                                                                                            )
                                                                                                                                                                                                                        .fetch();
                                                                                                                                                                                                                            }

                                                                                                                                                                                                                                private BooleanExpression usernameEq(String name) {
                                                                                                                                                                                                                                        return name != null ? QUser.user.username.eq(name) : null;
                                                                                                                                                                                                                                            }

                                                                                                                                                                                                                                                private BooleanExpression ageEq(Integer age) {
                                                                                                                                                                                                                                                        return age != null ? QUser.user.age.eq(age) : null;
                                                                                                                                                                                                                                                            }
                                                                                                                                                                                                                                                            }
                                                                                                                                                                                                                                                            ```

                                                                                                                                                                                                                                                            `where()`에 `null`을 전달하면 해당 조건은 자동으로 무시됩니다. 이 방식은 각 조건을 독립적인 메서드로 분리하여 **재사용성**과 **가독성**을 동시에 높여줍니다.

                                                                                                                                                                                                                                                            ---

                                                                                                                                                                                                                                                            ## 5. QueryDSL 심화: Join, 페이징 처리 🚀

                                                                                                                                                                                                                                                            ### 연관된 데이터 함께 가져오기: Join

                                                                                                                                                                                                                                                            QueryDSL에서는 `join()`을 사용하여 엔티티 간의 조인을 직관적으로 표현할 수 있습니다.

                                                                                                                                                                                                                                                            ```java
                                                                                                                                                                                                                                                            QOrder order = QOrder.order;
                                                                                                                                                                                                                                                            QMember member = QMember.member;

                                                                                                                                                                                                                                                            List<Order> orders = queryFactory
                                                                                                                                                                                                                                                                .selectFrom(order)
                                                                                                                                                                                                                                                                    .join(order.member, member)
                                                                                                                                                                                                                                                                        .where(member.username.eq("kim"))
                                                                                                                                                                                                                                                                            .fetch();
                                                                                                                                                                                                                                                                            ```

                                                                                                                                                                                                                                                                            ### 대용량 데이터 효율적으로 다루기: 페이징 처리

                                                                                                                                                                                                                                                                            `offset()`과 `limit()`을 사용해 페이징을 구현하고, count 쿼리를 분리하는 것이 성능상 유리합니다.

                                                                                                                                                                                                                                                                            ```java
                                                                                                                                                                                                                                                                            public Page<Product> searchProducts(String keyword, Pageable pageable) {
                                                                                                                                                                                                                                                                                QProduct product = QProduct.product;

                                                                                                                                                                                                                                                                                    List<Product> content = queryFactory
                                                                                                                                                                                                                                                                                            .selectFrom(product)
                                                                                                                                                                                                                                                                                                    .where(product.name.contains(keyword))
                                                                                                                                                                                                                                                                                                            .offset(pageable.getOffset())
                                                                                                                                                                                                                                                                                                                    .limit(pageable.getPageSize())
                                                                                                                                                                                                                                                                                                                            .fetch();

                                                                                                                                                                                                                                                                                                                                Long total = queryFactory
                                                                                                                                                                                                                                                                                                                                        .select(product.count())
                                                                                                                                                                                                                                                                                                                                                .from(product)
                                                                                                                                                                                                                                                                                                                                                        .where(product.name.contains(keyword))
                                                                                                                                                                                                                                                                                                                                                                .fetchOne();

                                                                                                                                                                                                                                                                                                                                                                    return new PageImpl<>(content, pageable, total);
                                                                                                                                                                                                                                                                                                                                                                    }
                                                                                                                                                                                                                                                                                                                                                                    ```

                                                                                                                                                                                                                                                                                                                                                                    ---

                                                                                                                                                                                                                                                                                                                                                                    ## 6. QueryDSL 프로젝션으로 DTO 깔끔하게 조회하기 ✨

                                                                                                                                                                                                                                                                                                                                                                    ### 왜 DTO로 직접 조회해야 할까?

                                                                                                                                                                                                                                                                                                                                                                    엔티티를 그대로 반환하면 불필요한 필드까지 모두 조회하게 되고, 영속성 컨텍스트 관리 비용도 발생합니다. 필요한 필드만 골라 DTO로 직접 조회하면 성능과 유지보수 면에서 큰 이점을 얻을 수 있습니다.

                                                                                                                                                                                                                                                                                                                                                                    ### @QueryProjection이란 무엇이며, 어떻게 동작하는가?

                                                                                                                                                                                                                                                                                                                                                                    `@QueryProjection`은 DTO의 생성자에 붙이는 어노테이션입니다. 이 어노테이션이 붙은 DTO는 컴파일 시점에 `QDto` 클래스가 생성되어, **타입 안전한 방식으로 DTO를 직접 조회**할 수 있게 됩니다.

                                                                                                                                                                                                                                                                                                                                                                    ### @QueryProjection 사용 방법

                                                                                                                                                                                                                                                                                                                                                                    ```java
                                                                                                                                                                                                                                                                                                                                                                    // DTO 정의
                                                                                                                                                                                                                                                                                                                                                                    @Data
                                                                                                                                                                                                                                                                                                                                                                    public class ProductDto {
                                                                                                                                                                                                                                                                                                                                                                        private String name;
                                                                                                                                                                                                                                                                                                                                                                            private int price;
                                                                                                                                                                                                                                                                                                                                                                                private String categoryName;

                                                                                                                                                                                                                                                                                                                                                                                    @QueryProjection
                                                                                                                                                                                                                                                                                                                                                                                        public ProductDto(String name, int price, String categoryName) {
                                                                                                                                                                                                                                                                                                                                                                                                this.name = name;
                                                                                                                                                                                                                                                                                                                                                                                                        this.price = price;
                                                                                                                                                                                                                                                                                                                                                                                                                this.categoryName = categoryName;
                                                                                                                                                                                                                                                                                                                                                                                                                    }
                                                                                                                                                                                                                                                                                                                                                                                                                    }
                                                                                                                                                                                                                                                                                                                                                                                                                    ```

                                                                                                                                                                                                                                                                                                                                                                                                                    ```java
                                                                                                                                                                                                                                                                                                                                                                                                                    // 쿼리에서 DTO로 직접 조회
                                                                                                                                                                                                                                                                                                                                                                                                                    QProduct product = QProduct.product;
                                                                                                                                                                                                                                                                                                                                                                                                                    QCategory category = QCategory.category;

                                                                                                                                                                                                                                                                                                                                                                                                                    List<ProductDto> results = queryFactory
                                                                                                                                                                                                                                                                                                                                                                                                                        .select(new QProductDto(
                                                                                                                                                                                                                                                                                                                                                                                                                                product.name,
                                                                                                                                                                                                                                                                                                                                                                                                                                        product.price,
                                                                                                                                                                                                                                                                                                                                                                                                                                                category.name
                                                                                                                                                                                                                                                                                                                                                                                                                                                    ))
                                                                                                                                                                                                                                                                                                                                                                                                                                                        .from(product)
                                                                                                                                                                                                                                                                                                                                                                                                                                                            .join(product.category, category)
                                                                                                                                                                                                                                                                                                                                                                                                                                                                .fetch();
                                                                                                                                                                                                                                                                                                                                                                                                                                                                ```

                                                                                                                                                                                                                                                                                                                                                                                                                                                                ### @QueryProjection의 장점과 단점

                                                                                                                                                                                                                                                                                                                                                                                                                                                                **장점**: 타입 안전성이 보장되며, IDE 자동완성을 활용할 수 있습니다. `Projections.constructor()`보다 컴파일 타임에 오류를 잡을 수 있습니다.

                                                                                                                                                                                                                                                                                                                                                                                                                                                                **단점**: DTO가 QueryDSL에 의존하게 됩니다. 순수한 DTO를 원한다면 `Projections.bean()` 또는 `Projections.constructor()`를 사용하는 대안도 있습니다.

                                                                                                                                                                                                                                                                                                                                                                                                                                                                ---

                                                                                                                                                                                                                                                                                                                                                                                                                                                                ## 마무리

                                                                                                                                                                                                                                                                                                                                                                                                                                                                QueryDSL은 단순히 "쿼리를 코드로 쓰는 도구"가 아닙니다. **컴파일 타임 안전성**, **동적 쿼리의 우아한 표현**, **IDE 지원**이라는 세 가지 강력한 이점을 통해 JPA 기반 애플리케이션의 데이터 접근 계층을 한 단계 끌어올려 주는 핵심 도구입니다.

                                                                                                                                                                                                                                                                                                                                                                                                                                                                처음 설정이 다소 번거롭게 느껴질 수 있지만, 한 번 익숙해지면 복잡한 동적 쿼리도 자신감 있게 작성할 수 있게 됩니다. 특히 검색 조건이 많은 실무 프로젝트에서 QueryDSL의 진가를 발휘할 수 있습니다.