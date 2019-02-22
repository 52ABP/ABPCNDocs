## 角色管理

角色实体代表了**该应用的一个角色**。它应该派生自**AbpRole**类，如下所示：
``` csharp

```

``` csharp
public class Role : AbpRole<Tenant, User>
    {
        //add your own role properties here
    }
```
    
角色存储在数据库的AbpRoles表中。您可以将自己的自定义属性添加到R​​ole类（并为更改创建数据库迁移）。

AbpRole定义了一些属性。最重要的是：
-   **Name**: 租户中角色的唯一名称。
-   **DisplayName**: 角色的显示名称。
-   **IsDefault**: 默认情况下是否将此角色分配给新用户？
-   **IsStatic**: 这个角色是静态的吗？（在预构建期间设置，不能删除）。

 
角色用于对权限进行分组。当用户拥有角色时，他/她将拥有该角色的所有权限。用户可以拥有 多个角色。此用户的权限将合并所有已分配角色的所有权限。

 

### 动态与静态角色

在**Module Zero**中，角色可以是动态的或静态的：

 

-   **Static role 静态角色**: 静态角色具有已知名称（如“admin”），无法更改（我们可以更改显示名称）。它存在于系统启动时，无法删除。这样，我们就可以根据静态角色的名称编写代码。
-   **Dynamic (non static) role 动态（非静态）角色**: 我们可以在部署后创建动态角色。然后我们可以为该角色授予权限，我们可以将角色分配给某些用户，我们可以将其删除。我们不知道开发过程中动态角色的名称。

使用**IsStatic**属性为角色设置它。我们还必须 在模块的**PreInitialize**方法中注册静态角色。假设我们对租户有一个“Admin”静态角色：

 
```csharp
    Configuration.Modules.Zero().RoleManagement.StaticRoles.Add(new StaticRoleDefinition("Admin", MultiTenancySides.Tenant));
```
这样，Module Zero就会知道它是静态角色。


### 默认角色

可以将一个或多个角色设置为默认值。默认情况下，默认角色分配给新添加/注册的用户。这不是开发时间属性，可以在部署后进行设置或更改。使用**IsDefault** 属性进行设置。

 

## 角色管理


**RoleManager** 是一种为角色执行**领域逻辑**的服务：

 ```csharp
    public class RoleManager : AbpRoleManager<Tenant, Role, User>
    {
        //...
    }

 ```
 
 您可以[注入](2.1ABP公共结构-依赖注入.md)并使用RoleManager来创建，删除，更新角色，为角色授予权限等等。您也可以在这里添加自己的方法。您还 可以根据自己的需要**覆盖AbpRoleManager**基类的任何方法。

 
 与UserManager一样，RoleManager的某些方法也会返回IdentityResult，而不是抛出异常。有关更多信息，请参阅用户管理文档。



 

## 多租户

与用户管理类似，角色管理也适用于多租户应用程序中的租户。有关 更多信息，请参阅用户管理文档。

 