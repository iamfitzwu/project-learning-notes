# 项目创建、开发与 CI/CD 流程总结

## 一、项目创建

### 1. 本地目录结构规划

- 在本地新建项目主文件夹，例如：`PractiseForXXX`
- 在主文件夹下新建 `src` 文件夹，用于存放所有源码相关的 Project

### 2. Rider 中新建 Solution

- 打开 Rider，选择 “New Solution”，路径指向主文件夹
- 创建一个空的 Solution，便于后续灵活添加各类项目模块

### 3. 添加结构目录（Solution Folder）

在 Rider 的 Solution Explorer 中右键添加以下 Solution Folder：

- **Facade**：用于存放 Web API 项目
- **Libraries**：通用库，如实体类、DTO、Service 等
- **Tests**：单元测试或集成测试项目

### 4. 新建 Project 并指定物理路径

> **注意：所有 Project 的物理路径应位于 `src` 文件夹下！**

示例目录结构如下：

```plaintext
PractiseForXXX/
├── src/
│   ├── Facade/
│   │   └── PractiseForXXX.WebApi/
│   ├── Libraries/
│   │   └── PractiseForXXX.Service/
│   └── Tests/
│       └── PractiseForXXX.Tests/
└── PractiseForXXX.sln
```

#### 各目录建议用途

| 目录      | 用途说明                      |
| --------- | -----------------------------|
| Facade    | 存放 Web API 项目             |
| Libraries | 通用库，如实体类、Service 等   |
| Tests     | 单元/集成测试                 |

#### 关键启动文件

- `Startup.cs`：负责所有配置工作
- `Program.cs`：应用程序入口，构建并运行应用程序

---

## 二、项目开发规范

> **目的：确保代码一致性、可维护性和专业性。**

### 1. 通用代码风格

- 使用4空格缩进，不使用Tab
- 每行建议不超120字符
- 操作符前后留空格，方法/属性间留空行
- if语句单行可省略花括号，多行必须加花括号
- 不允许无意义注释，强调自文档化代码，仅公共API写XML注释

### 2. 命名约定

#### - 异步方法以 `Async` 结尾  
```csharp
public async Task<UserDto> GetUserByIdAsync(int id, CancellationToken cancellationToken)
```

#### - 类/接口命名  
PascalCase；接口以I开头；服务类以Service结尾。  
```csharp
public interface IUserService;
public class UserService : IUserService;
```

#### - 方法/属性命名  
PascalCase；布尔属性以Is/Has/Can开头。  
```csharp
public bool IsActive { get; set; }
```

#### - 局部变量/参数命名  
camelCase，私有字段以下划线 `_` 开头。  
```csharp
private readonly IUserService _userService;
```

### 3. 异步编程规范

- 库代码 await 时需 `.ConfigureAwait(false)`；Controller 可省略。
- 所有异步方法接受 CancellationToken 参数，并置于最后。

### 4. 架构模式与分层设计  

采用 DDD 架构分层：

| 层级     | 职责说明                   |
| -------- | -------------------------- |
| Api      | 控制器与Web相关             |
| Core     | 业务逻辑、领域实体          |
| Messages | DTO / Command / Request等   |

采用 CQRS 分离读写，通过 Mediator 中介者处理。

泛型仓储模式统一数据访问接口，实现依赖注入管理服务生命周期。

### 5. 实体与数据模型规范  

实体实现 IEntity 或 IEntity<TId> 接口，并使用 EF Core 特性进行数据库映射。审计字段实现相应接口。

### 6. 服务层设计  

服务接口继承 IScopedDependency，返回具体响应类型。分页返回元组 `(int count, List<T> items)`。

每个聚合根对应一个 DataProvider，负责数据访问逻辑。

### 7. 测试规范  

测试方法 UpperCamelCase 命名，无下划线。例如：

```csharp
public void CanGetAskAnswerByQid()
public void ShouldRemoveNonexistentAskAnswers()
```
使用 xUnit + Shouldly + NSubstitute 模拟。

---

## 三、CI 配置 —— TeamCity 持续集成流程

TeamCity 持续集成流程旨在实现自动化构建、测试与打包，确保每一次代码提交都能被及时验证和交付。主要步骤如下：

### （1）创建 TeamCity 项目

1. 登录 TeamCity 管理后台，新建项目，并关联代码仓库（如 GitHub/GitLab 等）。

### （2）配置 Build Configurations（流水线）

通常包含两个主要阶段：

#### a) Build 阶段

**步骤说明**：
1. `git clean` —— 清理工作目录，避免历史文件干扰。
2. `Restore` —— 还原 NuGet 包依赖。
3. `GitVersion` —— 获取语义化版本号，用于后续打包与发布。
4. `Build` —— 执行 dotnet build 构建项目。
5. `Test` —— 执行 dotnet test 单元测试。

#### b) Containerization Build 阶段

**步骤说明**：
1. `GitVersion`
2. `Docker Login`
3. `Docker Build Image`
4. `Docker Push Image`
5. `Docker Remove`

---

### （3）Dockerfile 创建与结构解读（多阶段构建）

持续集成过程中 TeamCity 会调用 Dockerfile 自动生成容器镜像。推荐多阶段构建示例：

```dockerfile
# base阶段: 基础运行环境，仅包含ASP.NET Core运行时
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 5204                      # 暴露端口5204供外部访问API服务

# build阶段: 编译环境，包括SDK和依赖恢复、编译源码等任务
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src                     # 设置工作目录为/src方便拷贝操作优化缓存利用率 

# 拷贝csproj并还原NuGet依赖，加速缓存利用率，提高CI速度 
COPY ["PractiseForXXX.WebApi/PractiseForXXX.WebApi.csproj", "PractiseForXXX.WebApi/"]
RUN dotnet restore "PractiseForXXX.WebApi/PractiseForXXX.WebApi.csproj"

# 拷贝全部源代码到容器中进行编译 
COPY . .
WORKDIR "/src/PractiseForXXX.WebApi"
RUN dotnet build "PractiseForXXX.WebApi.csproj" -c Release -o /app/build --no-restore 

# publish阶段: 发布编译好的应用到独立目录下（裁剪体积）
FROM build AS publish 
RUN dotnet publish "PractiseForXXX.WebApi.csproj" -c Release -o /app/publish --no-build --no-restore 

# final阶段: 基于base镜像，将发布产物拷贝至最终容器，并设置程序入口点 
FROM base AS final 
WORKDIR /app 
COPY --from=publish /app/publish . 
ENTRYPOINT ["dotnet", "PractiseForXXX.WebApi.dll"]
```

#### 各阶段职责说明：

| 阶段    | 镜像基础         | 职责描述                                         |
| ------- | ---------------- | ------------------------------------------------|
| base    | aspnet:8.0       | 最终运行环境，仅包含 ASP.NET Core 和端口暴露      |
| build   | sdk:8.0          | 编译环境，还原依赖并编译源码                      |
| publish | build            | 发布已编译产物至精简目录                         |
| final   | base             | 提取发布产物作为最终容器内容，并设置程序入口       |

> **优势：多阶段构建极大减少了生产镜像体积，提高了安全性和启动速度。**

---

## 四、CD 配置 —— Octopus 持续交付流程

Octopus 部署流程负责将已完成的应用容器自动化部署到目标 Kubernetes 集群，实现高效、安全的上线交付。

### （1）创建 Octopus 项目及环境准备

在 Octopus 后台新建项目，为不同业务线或服务配置独立项目。同时配置相关变量（如 K8s 命名空间、镜像地址、Ingress 域名等），支持多环境动态切换和参数化部署。

### （2）配置 Process 流程（推荐步骤）

#### a) 部署 API 容器服务至 Kubernetes 集群 

1. **部署目标**
    - 标签：aws-eks  
    - Kubernetes 命名空间：#{K8SNameSpace}（变量化）
2. **Deployment 配置**
    - 类型：Deployment  
    - 名称：deploy-practise-for-shawn-api  
    - 副本数：#{NumOfPodForPractiseForShawnApi}（变量化）
    - 策略：滚动更新 (Rolling Update)
    - 健康检查：自动检测 rollout 成功与否    
3. **容器配置**
    - 镜像地址：solar/practise4shawn   
    - 暴露端口：8080/TCP   
4. **Service 配置**
    - 名称：service-practise4shawn-api   
    - 类型：ClusterIP    
    - 映射端口：http 服务端口8080   
5. **ConfigMap 配置**
    - 名称：configurations-webapi   
    - 键值对示例：
        ```
        PractiseForShawnConnectionString: #{PractiseForShawnConnectionString}
        ```
      （通过 Octopus 注入变量）

---

#### b) 部署 Ingress Resource 至 Kubernetes 

1. **部署目标**
   - 标签 aws-eks   
   - 命名空间同上    
2. **Ingress 配置**
    ```
    Ingress 名称         ：ingress-practise4shawn  
    注解 Annotations     ：cert-manager.io/cluster-issuer: letsencrypt  
                           kubernetes.io/tls-acme: true  
                           nginx.ingress.kubernetes.io/proxy-body-size: 0  
                           kubernetes.io/ingress.class: nginx  
                           nginx.ingress.kubernetes.io/limit-rpm: 200     
                           
    主机规则 Host Rules ：#{IngressBaseDomainName}/ => service-practise4shawn-api:5204      
                          域名支持变量化，多环境切换方便     
                          
    TLS 配置            ：针对 ${IngressBaseDomainName} 启用 TLS 加密      
                          自动申请证书        
    ```

---

### （3）Project Variables 配置   

1. **域名与入口相关配置**
   > IngressBaseDomainName 用于定义每个环境下的对外访问域名，如 example.com，可多环境切换。
2. **命名空间**
   > K8SNameSpace 指定部署到的 K8s 命名空间，各环境隔离管理。
3. **副本数与伸缩**
   > NumOfPodForXXXApi 控制各服务副本数，应对不同负载需求。
4. **数据库连接等外部资源信息**
   > ConnectionString / DbConnectionString 管理数据库或外部服务连接信息，通过变量安全注入。
5. **容器资源限制**
   > CpuResourceLimit/CpuResourceRequest 和 MemoryResourceLimit/MemoryResourceRequest 控制 CPU 与内存请求及上限，有助于合理分配资源、防止资源争抢。


---