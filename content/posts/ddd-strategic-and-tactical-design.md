+++
date = '2026-01-05T19:03:02+08:00'
draft = false
title = 'DDD Strategic and Tactical Design'
tags = ['DDD', '领域驱动设计', '架构设计']
categories = ['软件架构']
+++

# 领域驱动设计(DDD)

领域驱动设计（Domain-Driven Design, DDD）是一种以领域模型为核心的软件开发方法论，旨在应对复杂业务系统的设计与演进。DDD 分为**战略设计**和**战术设计**两个层面：战略设计关注系统整体的边界划分与团队协作模式，战术设计则聚焦于领域模型的具体实现模式。

## 1. 战略设计

战略设计的目标是从整体上划分系统的**限界上下文（Bounded Context）**，并在每个上下文中发展出清晰一致的**通用语言（Ubiquitous Language）**。

### 1.1 领域与子域

- **领域（Domain）**：系统所关注的问题范围，代表业务活动和规则的集合。
- **子域（Subdomain）**：领域的组成部分，将复杂业务划分为更小、更聚焦的单元。

子域通常分为三类：
| 类型 | 说明 |
|------|------|
| **核心域（Core Domain）** | 业务的核心竞争力所在，需要重点投入 |
| **支撑域（Supporting Subdomain）** | 支撑核心业务运作，但非核心竞争力 |
| **通用域（Generic Subdomain）** | 通用功能，可采用现成方案 |

领域分析发生在**问题空间（Problem Space）**，而领域模型的设计与实现属于**解决方案空间（Solution Space）**：

- **问题空间**：识别业务目标和边界，关注"做什么"。
- **解决方案空间**：将需求转化为可实现的设计和模型，关注"怎么做"。

### 1.2 限界上下文（Bounded Context）

**限界上下文**是领域模型的语义边界。在该边界内，所有的概念、对象和规则都有明确且一致的含义。

> 同一个术语在不同上下文中可能代表不同含义，而在同一上下文中则保持语义一致。

例如，"账户"在用户上下文中可能指用户登录凭证，而在财务上下文中则指资金账户。限界上下文的划分有助于避免模型的混淆和污染。

### 1.3 通用语言（Ubiquitous Language）

**通用语言**是限界上下文内部领域专家与开发人员共享的统一语言。它通过领域模型来表达业务规则、行为和约束。

- 每个限界上下文都有自己独立的通用语言
- 通用语言不能跨上下文混用
- 代码、文档、沟通都应使用通用语言

### 1.4 上下文映射（Context Mapping）

当不同的限界上下文需要交互时，必须通过**上下文映射**来完成语言的"翻译"和语义对齐。上下文映射定义了上下文之间的关系、通信模式以及模型转换方式。

常见的上下文映射模式包括：

| 模式 | 说明 |
|------|------|
| **合作关系（Partnership）** | 两个团队共同协调开发，相互依赖 |
| **共享内核（Shared Kernel）** | 共享部分模型代码，需谨慎管理 |
| **客户-供应商（Customer-Supplier）** | 上游供应、下游消费，下游可提需求 |
| **遵奉者（Conformist）** | 下游完全遵从上游模型 |
| **防腐层（ACL）** | 下游建立转换层隔离上游模型 |
| **开放主机服务（OHS）** | 上游提供标准化协议供多方使用 |
| **发布语言（Published Language）** | 使用标准化的数据交换格式 |

### 1.5 防腐层（Anti-Corruption Layer）

下游上下文在使用上游上下文的数据或服务时，应建立一个**防腐层（ACL）**。防腐层负责：

- 隔离下游模型与上游模型
- 通过转换适配，防止外部模型污染内部语义
- 使下游上下文保持独立演进的能力

---

## 2. 战术设计

战术设计提供了一系列构建领域模型的模式，用于在限界上下文内部实现业务逻辑。

### 2.1 聚合（Aggregate）与聚合根（Aggregate Root）

- **聚合**：一组相关对象的集合，作为数据修改的单元，由聚合根统一管理。
- **聚合根**：聚合的唯一入口点，负责保护内部业务规则的一致性。

**设计原则**：
1. 设计小聚合，避免过大的聚合边界
2. 聚合内部保持强一致性
3. 跨聚合通过标识符引用，而非对象引用
4. 聚合边界之外使用最终一致性

```java
// 聚合根
public class User { 
    private Long id;
    private String username;
    private UserProfile profile;      // 值对象
    private UserCredential credential; // 实体
    private UserSetting setting;       // 值对象

    // 业务行为由聚合根协调
    public void changePassword(String newPassword) {
        credential.changePassword(newPassword);
    }

    public void deactivate() {
        credential.deactivate();
    }
}
```

### 2.2 实体（Entity）与值对象（Value Object）

- **实体**：具有唯一标识，生命周期内标识不变，状态可变。
- **值对象**：无唯一标识，不可变，通过属性值判断相等性，用于描述或量化实体的属性。

```java
// 实体：有标识，可变
public class UserCredential {
    private String email;
    private String phone;
    private String passwordHash;
    private boolean active;

    public void changePassword(String newPassword) {
        this.passwordHash = hash(newPassword);
    }

    public void deactivate() {
        this.active = false;
    }

    private String hash(String password) {
        // 哈希逻辑省略
        return password;
    }
}

// 值对象：无标识，不可变（Java 17+ record）
public record UserProfile(String nickname, String avatarUrl, String gender) {
    public UserProfile {
        if (nickname == null || nickname.isBlank()) {
            throw new IllegalArgumentException("昵称不能为空");
        }
    }
}

public record UserSetting(boolean notificationsEnabled, String language) {
    public UserSetting {
        if (language == null || language.isBlank()) {
            throw new IllegalArgumentException("语言不能为空");
        }
    }
}
```

### 2.3 领域服务（Domain Service）

当业务逻辑**跨越多个实体或聚合**，不适合放在单个实体中时，由领域服务承担。领域服务是无状态的，专注于领域逻辑的实现。

```java
public class GroupDomainService {

    private final GroupMemberRepository groupMemberRepository;

    public GroupDomainService(GroupMemberRepository groupMemberRepository) {
        this.groupMemberRepository = groupMemberRepository;
    }

    public GroupMember addMemberToGroup(ChatGroup group, User operator, User userToAdd) {
        // 业务规则校验
        if (!operator.isActive()) {
            throw new IllegalArgumentException("操作者账号不可用");
        }
        if (!userToAdd.isActive()) {
            throw new IllegalArgumentException("被添加用户账号不可用");
        }
        if (!group.isOperator(operator.getId())) {
            throw new IllegalArgumentException("操作者没有权限");
        }
        if (operator.getId().equals(userToAdd.getId())) {
            throw new IllegalArgumentException("不能添加自己到群组");
        }
        if (groupMemberRepository.exists(group.getId(), userToAdd.getId())) {
            throw new IllegalArgumentException("成员已存在群组中");
        }

        return GroupMember.builder()
                .groupId(group.getId())
                .userId(userToAdd.getId())
                .role(GroupMember.Role.MEMBER.getValue())
                .build();
    }
}
```

### 2.4 应用服务（Application Service）

应用服务是领域模型的客户端，负责：
- 协调领域对象完成用例
- 管理事务边界
- 处理安全认证、日志等横切关注点
- **不承担核心业务规则**

```java
@Service
public class GroupApplicationService {

    private final GroupRepository groupRepository;
    private final GroupMemberRepository groupMemberRepository;
    private final GroupDomainService groupDomainService;
    private final UserApplicationService userApplicationService;
    private final EventPublisher eventPublisher;

    @Transactional
    public void addMember(AddMemberCmd cmd) {
        // 1. 获取聚合
        ChatGroup chatGroup = groupRepository.getById(cmd.getGroupId())
                .orElseThrow(() -> new IllegalArgumentException("Group not found"));

        // 2. 跨上下文校验（通过应用服务）
        userApplicationService.validateUserExists(cmd.getOperatorId());
        userApplicationService.validateUserExists(cmd.getMemberId());

        // 3. 调用领域服务处理业务逻辑
        GroupMember groupMember = groupDomainService.addMemberToGroup(
                chatGroup, cmd.getOperatorId(), cmd.getMemberId()
        );

        // 4. 持久化
        groupMemberRepository.save(groupMember);

        // 5. 发布领域事件
        eventPublisher.publish(new UserJoinedGroupEvent(
                chatGroup.getId(),
                cmd.getMemberId(),
                cmd.getOperatorId(),
                LocalDateTime.now()));
    }
}
```

### 2.5 领域事件（Domain Event）

领域事件表示领域中发生的有意义的业务事件，基于发布-订阅模式实现模块间解耦。

**特点**：
- 事件名称使用过去时态（如 `UserJoinedGroupEvent`）
- 事件是不可变的
- 包含事件发生时的相关数据

```java
// 定义领域事件
public class UserJoinedGroupEvent extends ApplicationEvent {
    private final Long groupId;
    private final Long userId;
    private final Long operatorId;
    private final LocalDateTime occurredAt;

    public UserJoinedGroupEvent(Long groupId, Long userId, 
                                 Long operatorId, LocalDateTime occurredAt) {
        super(groupId);
        this.groupId = groupId;
        this.userId = userId;
        this.operatorId = operatorId;
        this.occurredAt = occurredAt;
    }

    // getters...
}

// 事件监听器
@Component
public class GroupEventListener {

    @EventListener
    public void handleUserJoined(UserJoinedGroupEvent event) {
        // 发送通知、记录日志、更新统计等
        log.info("用户 {} 加入群组 {}", event.getUserId(), event.getGroupId());
    }
}
```

### 2.6 工厂（Factory）

工厂用于封装复杂对象的创建逻辑，确保创建出的对象满足业务规则。

- **聚合根工厂方法**：用于创建聚合，封装创建时的业务校验
- **领域服务工厂**：在上下文集成时，将外部模型转换为本地模型

```java
public class User {
    private Long id;
    private String username;
    private final UserProfile profile;
    private final UserCredential credential;
    private final UserSetting setting;

    // 私有构造函数
    private User(String username, UserProfile profile, 
                 UserCredential credential, UserSetting setting) {
        this.username = username;
        this.profile = profile;
        this.credential = credential;
        this.setting = setting;
    }

    // 工厂方法：封装创建逻辑与业务校验
    public static User create(String username, UserProfile profile, 
                               UserCredential credential, UserSetting setting) {
        // 聚合级别的业务校验
        if (username == null || username.isBlank()) {
            throw new IllegalArgumentException("用户名不能为空");
        }
        return new User(username, profile, credential, setting);
    }
}
```

### 2.7 资源库（Repository）

资源库是聚合的持久化抽象，提供面向集合的接口来存取聚合。

**与 DAO 的区别**：
| 资源库（Repository） | DAO |
|---------------------|-----|
| 面向聚合和业务模型 | 面向数据表和实体 |
| 隐藏持久化细节 | 暴露数据访问操作 |
| 一次操作整个聚合 | 操作单个表或实体 |

```java
// 资源库接口（领域层）
public interface UserRepository {
    void save(User user);
    Optional<User> findById(Long id);
    void delete(User user);
}

// 资源库实现（基础设施层）
@Repository
public class UserRepositoryImpl implements UserRepository {

    private final UserMapper userMapper;
    private final UserProfileMapper profileMapper;
    private final UserCredentialMapper credentialMapper;
    private final UserSettingMapper settingMapper;

    @Transactional
    @Override
    public void save(User user) {
        if (user.getId() == null) {
            // 新增
            userMapper.insert(UserAssembler.toPO(user));
            Long userId = user.getId();
            profileMapper.insert(UserAssembler.toPO(user.getProfile(), userId));
            credentialMapper.insert(UserAssembler.toPO(user.getCredential(), userId));
            settingMapper.insert(UserAssembler.toPO(user.getSetting(), userId));
        } else {
            // 更新
            userMapper.update(UserAssembler.toPO(user));
            Long userId = user.getId();
            profileMapper.update(UserAssembler.toPO(user.getProfile(), userId));
            credentialMapper.update(UserAssembler.toPO(user.getCredential(), userId));
            settingMapper.update(UserAssembler.toPO(user.getSetting(), userId));
        }
    }

    @Override
    public Optional<User> findById(Long id) {
        UserPO userPO = userMapper.selectById(id);
        if (userPO == null) {
            return Optional.empty();
        }
        UserProfilePO profilePO = profileMapper.selectByUserId(id);
        UserCredentialPO credentialPO = credentialMapper.selectByUserId(id);
        UserSettingPO settingPO = settingMapper.selectByUserId(id);

        return Optional.of(UserAssembler.toAggregate(userPO, profilePO, credentialPO, settingPO));
    }
}
```

---

## 总结

| 层面 | 关注点 | 核心概念 |
|------|--------|----------|
| **战略设计** | 系统边界与团队协作 | 限界上下文、通用语言、上下文映射、防腐层 |
| **战术设计** | 领域模型实现 | 聚合、实体、值对象、领域服务、领域事件、资源库 |

DDD 的价值在于让技术实现与业务语义保持一致，通过清晰的边界划分和统一的语言降低系统复杂度，使软件能够持续演进以适应业务变化。

## 参考资料

- Vaughn Vernon. *"Implementing Domain-Driven Design"*（《实现领域驱动设计》）
- Vaughn Vernon. *"Domain-Driven Design Distilled"*（《领域驱动设计精粹》）
- Eric Evans. *"Domain-Driven Design: Tackling Complexity in the Heart of Software"*（《领域驱动设计：软件核心复杂性应对之道》）
