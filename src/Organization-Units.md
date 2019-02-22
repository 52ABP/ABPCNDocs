## 组织单元--介绍

 

组织单元（Organization Unit【简称OU】）**可以将用户和实体进行合理的分组**。



### OrganizationUnit实体


下文中将用OU来表示**OrganizationUnit**实体。该实体的基本属性如下：

-   **TenantId**: 该OU的租户Id。对于租主的OU可以为null。
-   **ParentId**:父OU的Id。如果该OU是根OU，那么可以是null。
-   **Code**: 对于租户来说是唯一的分层字符串 Code。
-   **DisplayName**: ou 的显示名称。

 
组织单元实体的主键 (id) 是一个**long**类型, OrganizationUnit实体是从提供了审计信息的 [**FullAuditedEntity**](3.1ABP领域层-实体.md) 派生的，而且实现了 [**ISoftDelete**](3.7ABP领域层-数据过滤器.md) 接口 (ou 不会从数据库中删除, 它们只是标记为已删除，也就是我们常说的“软删除”)。

 

### 组织树（Organization Tree）

 

由于 ou 可以有父对象, 因此租户的所有 ou 都是在一个**树形结构**中的。这棵树有一些规则;

- 可以有多个根(root)节点 (其中 parentid 为空)。
- 树的最大深度被定义为一个常数, 如 OrganizationUnit.**MaxDepth**。这个值是**16**。
- 一个OU的第一层的子集数量是有限制的（因为OU Code单元长度是固定的，下面有解释）。
 

### OU 编码

ou  Code由组织单位自动生成和维护的。它是一个类似于以下内容的字符串:

"**00001.00042.00005**"

通过此** Code**可用于轻松地查询数据库中所有 ou 的子级 (递归)。此 Code有一些规则:

- 对于租户来说, 它必须是唯一的。
- 同一 ou 的所有子级都有以父 ou  Code开头的 Code。
- 它是固定长度，基于树中OU的级别，如示例所示。
- 虽然 ou  Code是唯一的, 但如果移动 ou, 它也是会改变的。
- 我们必须通过 id 引用 ou, 而不是 Code。

 

### OrganizationUnit Manager 领域服务

该**OrganizationUnitManager**类可以 通过[依赖注入](2.1ABP公共结构-依赖注入.md)，引用来管理的OU。常见用例是：

- 创建，更新或删除OU
- 在OU树中移动OU。
- 获取有关OU树及其项目有关的信息。

### Multi-Tenancy 多租户

OrganizationUnitManager 目前只能为**单个租户**工作。默认情况下，它适用于**当前租户**。

 

### 常见用例

在这里，我们将看到OU的一些常见用例。您可以在此处找到示例的[源代码](https://github.com/52ABP/52abp-samples) 。
 

### 创建属于组织单位的实体

ou 最明显的用法是将实体分配给 ou。让我们看一个示例实体:

 ```csharp

    public class Product : Entity, IMustHaveTenant, IMustHaveOrganizationUnit
    {
        public virtual int TenantId { get; set; }

        public virtual long OrganizationUnitId { get; set; }
        
        public virtual string Name { get; set; }

        public virtual float Price { get; set; }
    }
```

 
 



我们创建了**OrganizationUnitId**属性将一个实体赋予一个OU。 **IMustHaveOrganizationUnit**定义了OrganizationUnitId属性。我们不必实现该接口，但是建议提供标准化。除此之外，还有一个**IMayHaveOrganizationId**，该接口提供了一个**nullable（可空）**的OrganizationUnitId属性。

现在，我们可以将一个Product关联到一个OU，并且查询一个特定OU的产品。

**注意**：Product实体有 **TenantId**（它是IMustHaveTenant接口中定义的属性）属性来区分多租户应用中不同租户的产品（请看[**多租户**](1.5ABP总体介绍-多租户.md)）。如果你的应用不是多租户，那么你不需要这个接口和属性。





### 获取组织单位中的实体


获取 ou 的产品很简单。让我们看看这个[领域服务](3.4ABP领域层-领域服务.md)的  :
Getting the Products of an OU is simple. Let's see this sample [domain
service](/Pages/Documents/Domain-Services):

``` csharp
    public class ProductManager : IDomainService
    {
        private readonly IRepository<Product> _productRepository;

        public ProductManager(IRepository<Product> productRepository)
        {
            _productRepository = productRepository;
        }

        public List<Product> GetProductsInOu(long organizationUnitId)
        {
            return _productRepository.GetAllList(p => p.OrganizationUnitId == organizationUnitId);
        }
                    
    }
```
如上所示，我们可以简单地针对Product.OrganizationUnitId编写方法。

 
 
 

### 获取组织单元其中包含了它的所有的子项组织单元

我们可能想获取一个包括子组织单元的组织单元的Products。在这种情况下，OU **Code（代码）**可以帮到我们：

 ```csharp
   public class ProductManager : IDomainService
    {
        private readonly IRepository<Product> _productRepository;
        private readonly IRepository<OrganizationUnit, long> _organizationUnitRepository;

        public ProductManager(
            IRepository<Product> productRepository, 
            IRepository<OrganizationUnit, long> organizationUnitRepository)
        {
            _productRepository = productRepository;
            _organizationUnitRepository = organizationUnitRepository;
        }

        [UnitOfWork]
        public virtual List<Product> GetProductsInOuIncludingChildren(long organizationUnitId)
        {
            var code = _organizationUnitRepository.Get(organizationUnitId).Code;

            var query =
                from product in _productRepository.GetAll()
                join organizationUnit in _organizationUnitRepository.GetAll() on product.OrganizationUnitId equals organizationUnit.Id
                where organizationUnit.Code.StartsWith(code)
                select product;

            return query.ToList();
        }
    }
 ```
 

  首先，我们得到了给定OU 的代码。然后我们创建了一个带有join和StartsWith（code）条件的LINQ表达式 （StartsWith 在SQL中创建一个 LIKE查询）。这样我们就可以分层次地获取OU的产品。

首先，我们获得给定OU的**Code**。然后，我们创建了一个具有**join**和**StartsWith(code)**条件（在sql中StartsWith创建一个**Like**查询）的LINQ。这样，我们就可以有层次低获得一个OU的products。


 

### 过滤用户的实体


我们可能获取在OU中的一个特定用户的所有Products，看下面的样例代码：

 ```csharp
 
    public class ProductManager : IDomainService
    {
        private readonly IRepository<Product> _productRepository;
        private readonly UserManager _userManager;

        public ProductManager(
            IRepository<Product> productRepository, 
            UserManager userManager)
        {
            _productRepository = productRepository;
            _organizationUnitRepository = organizationUnitRepository;
            _userManager = userManager;
        }

        public async Task<List<Product>> GetProductsForUserAsync(long userId)
        {
            var user = await _userManager.GetUserByIdAsync(userId);
            var organizationUnits = await _userManager.GetOrganizationUnitsAsync(user);
            var organizationUnitIds = organizationUnits.Select(ou => ou.Id);

            return await _productRepository.GetAllListAsync(p => organizationUnitIds.Contains(p.OrganizationUnitId));
        }
    }
 ```
 
 我们先找到该用户OU的Id，然后获取Products时使用了**Contains**条件。当然，我们可以创建一个具有join的LINQ查询来获得相同的列表。


我们可能希望在用户的OU中获得Products，**包括他们的子OU**：


 
 ```csharp
  public class ProductManager : IDomainService
    {
        private readonly IRepository<Product> _productRepository;
        private readonly IRepository<OrganizationUnit, long> _organizationUnitRepository;
        private readonly UserManager _userManager;

        public ProductManager(
            IRepository<Product> productRepository, 
            IRepository<OrganizationUnit, long> organizationUnitRepository, 
            UserManager userManager)
        {
            _productRepository = productRepository;
            _organizationUnitRepository = organizationUnitRepository;
            _userManager = userManager;
        }

        [UnitOfWork]
        public virtual async Task<List<Product>> GetProductsForUserIncludingChildOusAsync(long userId)
        {
            var user = await _userManager.GetUserByIdAsync(userId);
            var organizationUnits = await _userManager.GetOrganizationUnitsAsync(user);
            var organizationUnitCodes = organizationUnits.Select(ou => ou.Code);

            var query =
                from product in _productRepository.GetAll()
                join organizationUnit in _organizationUnitRepository.GetAll() on product.OrganizationUnitId equals organizationUnit.Id
                where organizationUnitCodes.Any(code => organizationUnit.Code.StartsWith(code))
                select product;

            return query.ToList();
        }
    }
 ```
 我们将**Any**与**StartsWith**条件组合成一个LINQ连接语句。

很可能会有更复杂的需求，但它们都可以使用LINQ或SQL完成。
   



### Settings 设置

您可以注入并使用**IOrganizationUnitSettings**接口来获取组织单位的设置值。目前只有一个设置可以根据您的应用需求进行更改：




-   **MaxUserMembershipCount**: 一个用户最大允许的关系数量。 
    默认值为**int.MaxValue**，它允许用户同时成为无限OU的成员。
设置名称是**AbpZeroSettingNames.OrganizationUnits.MaxUserMembershipCount**中定义的 常量。

您可以使用[**设置管理**](2.5ABP公共结构-设置管理.md)更改设置值。


 