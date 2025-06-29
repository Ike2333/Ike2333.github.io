+++
date = '2025-06-28T14:56:25+08:00'
categories = ["hibernate", "spring"]
title = 'JPA Hibernate Query Optimization'
description = "JPA Hibernate 查询优化"
+++

## 1. 概述
`spring data jpa 3.x`版本中对Specification查询相关api进行了一些增强, 使得我们可以用更少的代码量来实现复杂的动态查询. 本文将基于最新版本的jpa介绍如何优化Hibernate查询, 将抛弃原始的HQL/JPQL/SQL, 以代码的方式实现各种查询.
在开始之前, 你应该已经了解了JPA/Hibernate/Criteria的基础知识, 你可以结合这个仓库快速理解接下来的内容: `https://github.com/Ike2333/specification-demo.git`


## 2. 可以先了解`src/main/kotlin/com/ike/specificationdemo/entity`包下的实体类建模


## 3. 简单查询优化
**假设我们需要`findAll`查询`UserEntity`实体中的id, username, email字段:**
* 最简单的方式是直接`userRepository.findAll()`,然后用一个只包含需要查询的字段的DTO去接收, 问题显而易见, 生成的SQL始终是`SELECT *`

* 为了避免直接`SELECT *`, 我们曾经需要先定义一个包含要查询的字段的投影接口
  ```kotlin
  interface UserProj(
      fun getUsername(): String?,
      fun getId(): Long?,
      fun getEmail(): String?,
  )
  ```
  
* 在新版本中, 接口投影可以更简单, 直接使用数据类, 这样也避免了额外的数据转换成本:
  ```kotlin
  data class SimplerUserDTO(
      val id: Long?,
      val username: String?,
      val email: String?,
  )
  ```
  
* 随后我们可以直接在UserRepository中增加如下的方法:
  ```kotlin
  // 方法签名必须遵循 `find{OBJ_NAME}By() : *<OBJ_NAME>` 这样的命名规则;
  fun findUserProjBy(): List<UserProj>
  fun findSimplerUserDTOBy(): List<SimplerUserDTO>
  ```


## 4. 复杂查询优化

### 4.1 假设我们要动态模糊查询`PermissionEntity.name`和`RoleEntity.name`与传入关键字相关的数据, 并返回需要的字段:

* 常见且最通用的做法如下:
  ```kotlin
    @Transactional
    fun findByRoleAndPermissionName(): Page<UserDTO> {
        val roleName = "first"
        val permissionName = "first"
        val pageable = PageRequest.of(0, 5)

        println("$roleName---------$permissionName")

        val spec = Specification<UserEntity> { root, query, cb ->
            val urj = root.join<UserEntity, RoleEntity>("roles", JoinType.LEFT)
            val rpj = urj.join<RoleEntity, PermissionEntity>("permissions", JoinType.LEFT)
            val predicates = mutableListOf<Predicate>()
            if (roleName.isNotBlank()) {
                predicates.add(cb.like(urj["name"], "%$roleName%"))
            }
            if (permissionName.isNotBlank()) {
                predicates.add(cb.like(rpj["name"], "%$permissionName%"))
            }

            cb.and(*predicates.toTypedArray())
        }

        return userRepository.findAll(spec, pageable).map {
            it.roles?.let { it1 ->
                UserDTO(
                    it.id,
                    it.username,
                    it.password,
                    it.createdAt,
                    it.updatedAt,
                    // 如果加上下面两行, 将会导致N+1查询, 会看到输出了多行sql
                    it1.map { x -> x.name }.toSet(),
                    it1.map { x -> x.path }.toSet()
                )
            }
        }
    }
  ```
  _直接findAll会查询所有的字段, 并且当我们尝试读取映射关系中的其他实体的字段时, hibernate会为了我们想要的值多次查询数据库(包含分页只会输出两行sql); 当然, 如果不尝试获取其他实体中的属性, 就不会有这些问题_


* 为了确保性能, 我们必须使用JPA原生的criteria api, 通过自定义子查询进行优化
  ```kotlin
  fun findByRoleAndPermissionNameOptimized(): Page<UserDTO> {
    val roleNameQuery = "first"
    val permissionNameQuery = "first"
    val pageable = PageRequest.of(0, 5)

    println("$roleNameQuery---------$permissionNameQuery")

    val cb = entityManager.criteriaBuilder
    // 此子查询将识别结果应包含哪些用户
    val userIdsSubquery: Subquery<Long> = cb.createQuery(UserEntity::class.java).subquery(Long::class.java)
    val subqueryRoot: Root<UserEntity> = userIdsSubquery.from(UserEntity::class.java)
    // 使用内连接查询具有角色和符合条件的权限的用户
    val subqueryUserRoleJoin = subqueryRoot.join<UserEntity, RoleEntity>("roles", JoinType.INNER)
    val subqueryRolePermissionJoin =
        subqueryUserRoleJoin.join<RoleEntity, PermissionEntity>("permissions", JoinType.INNER)

    val subqueryPredicates = mutableListOf<Predicate>()
    if (roleNameQuery.isNotBlank()) {
        subqueryPredicates.add(cb.like(subqueryUserRoleJoin.get("name"), "%$roleNameQuery%"))
    }
    if (permissionNameQuery.isNotBlank()) {
        subqueryPredicates.add(cb.like(subqueryRolePermissionJoin.get("name"), "%$permissionNameQuery%"))
    }

    userIdsSubquery.select(subqueryRoot.get("id")).where(*subqueryPredicates.toTypedArray()).distinct(true)


    // 针对即将查询的用户, 获取其角色和权限进行过滤
    val mainQuery = cb.createQuery(UserFlatDTO::class.java)
    val mainRoot = mainQuery.from(UserEntity::class.java)
    val mainUserRoleJoin = mainRoot.join<UserEntity, RoleEntity>("roles", JoinType.LEFT)

    mainQuery.select(
        cb.construct(
            UserFlatDTO::class.java,
            mainRoot.get<Long>("id"),
            mainRoot.get<String>("username"),
            mainRoot.get<String>("password"),
            mainRoot.get<Instant>("createdAt"),
            mainRoot.get<Instant>("updatedAt"),
            mainUserRoleJoin.get<String>("name"),
            mainUserRoleJoin.get<String>("path"),
        )
    )

    // 使用子查询中的ID过滤主查询
    mainQuery.where(mainRoot.get<Long>("id").`in`(userIdsSubquery))

    // 针对主查询实体(UserEntity)排序
    if (pageable.sort.isSorted) {
        val orders = pageable.sort.map { order ->
            if (order.isAscending) cb.asc(mainRoot.get<Long>("id")) else cb.desc(mainRoot.get<Long>("id"))
        }.toList()
        mainQuery.orderBy(orders)
    }

    // UserFlatDTO分页查询
    val flatTypedQuery = entityManager.createQuery(mainQuery)
    flatTypedQuery.firstResult = pageable.offset.toInt()
    flatTypedQuery.maxResults = pageable.pageSize

    val flatResults: List<UserFlatDTO> = flatTypedQuery.resultList

    // 将UserFlatDTO聚合为userDTO
    val userDtos = flatResults
        .groupBy { it.id } // 以user id分组
        .map { (userId, group) ->
            val firstFlatDto = group.first()
            UserDTO(
                userId,
                firstFlatDto.username,
                firstFlatDto.password,
                firstFlatDto.createdAt,
                firstFlatDto.updatedAt,
                group.mapNotNull { it.roleName }.toSet(),
                group.mapNotNull { it.rolePath }.toSet()
            )
        }

    // 获取命中查询条件的总用户数
    val countCq = cb.createQuery(Long::class.java)
    val countRoot = countCq.from(UserEntity::class.java)
    countCq.select(cb.countDistinct(countRoot.get<Long>("id")))
    countCq.where(countRoot.get<Long>("id").`in`(userIdsSubquery))
    val totalElements = entityManager.createQuery(countCq).singleResult

    return PageImpl(userDtos, pageable, totalElements)
  }
  ```
  _即便要获取其他实体的属性, 输出的sql也只会是两行_