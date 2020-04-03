在 EntityFramework Core 中，我们可以使用属性或Fluent API来配置模型映射。有一天，我遇到了一个新的需求，有一个系统每天会生成大量数据，每天生成一个新的表存储数据。例如，数据库如下所示：![img](https://mmbiz.qpic.cn/mmbiz_png/ahAATVIdckQ4CwpQmSvyw0hmrDBSIUn2OTzA7ryKGDQx4oiaIX3sgQvfGHqsaibN7pTMWgdAVRUpJ09BogrhCK3g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

所有表都具有相同的结构。那么，如何更改映射以避免创建多个模型呢？

在本文中，我将向您展示如何更改映射以处理这种情况。您也可以使用此方法扩展出更多的用法。

## 创建 .NET Core 3.1 项目

现在，我们可以使用.NET Core 3.1，它是.NET Core的LTS版本，将来可以轻松将其升级到.NET 5。

假设您已经在计算机上安装了最新的.NET Core SDK。如果没有，则可以从https://dotnet.microsoft.com/download下载。然后，您可以使用`dotnet CLI`创建项目。对于此示例，我将使用.NET Core 3.1。

让我们创建一个名为`DynamicModelDemo`的新.NET Core Console项目:

```
dotnet new console --name DynamicModelDemo
```

然后用以下命令创建一个新的解决方案:

```
dotnet new sln --name DynamicModelDemo
```

接下来使用以下命令把刚才创建的项目添加到解决方案：

```
dotnet sln add "DynamicModelDemo/DynamicModelDemo.csproj"
```

接下来可以用Visual Studio打开解决方案了。

## 创建模型

该模型非常简单。在项目中添加一个名为`ConfigurableEntity.cs`的新文件：

```
using System;

namespace DynamicModelDemo
{
    public class ConfigurableEntity
    {
        public int Id { get; set; }
        public string Title { get; set; }
        public string Content { get; set; }
        public DateTime CreateDateTime { get; set; }
    }
}
```

我们将使用`CreateDateTime`属性来确定模型应该映射到哪个表。

## 添加 EntityFramework Core

导航到项目目录并使用以下命令添加所需的EF.Core packages:

```
dotnet add package Microsoft.EntityFrameworkCore.SqlSever
dotnet add package Microsoft.EntityFrameworkCore.Design
```

如果您还没有安装 ef tool，请运行以下命令来安装：

```
dotnet tool install --global dotnet-ef
```

这样您就可以使用 dotnet ef 工具创建迁移或通过应用迁移来更新数据库。

## 创建 DbContext

向项目添加一个名为`DynamicContext.cs`的新类文件。内容如下所示：

```
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Infrastructure;
using System;

namespace DynamicModelDemo
{
    public class DynamicContext : DbContext
    {
        public DbSet<ConfigurableEntity> Entities { get; set; }

        #region OnConfiguring
        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
            => optionsBuilder
                .UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=DynamicContext;Trusted_Connection=True;");
        #endregion

        #region OnModelCreating
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.Entity<ConfigurableEntity>(b =>
            {
                b.HasKey(p => p.Id);
            });
        }
        #endregion
    }
}
```

目前，这只是EF.Core的基本配置。它使用默认映射，这意味着模型将映射到名为`Entities`的表。那么，如果我们想基于其`CreateDateTime`属性将模型映射到不同的表，该怎么办呢？

您可能知道我们可以使用`ToTable()`方法来更改表名，但是如何在`OnModelCreating`方法中更改所有模型的表名呢？EF建立模型时，只会执行一次OnModelCreating。所以这种方式是无法实现的。

对于这种情况，我们需要使用`IModelCacheKeyFactory`来更改默认映射，通过这个接口我们可以定制模型缓存机制，以便EF能够根据其属性创建不同的模型。

## `IModelCacheKeyFactory`是什么？

这是微软官方的文档解释：

> EF uses `IModelCacheKeyFactory` to generate cache keys for models.

默认情况下，EF假定对于任何给定的上下文类型，模型都是相同的。但是对于我们的方案，模型将有所不同，因为它映射到了不同的表。因此，我们需要用我们的实现替换`IModelCacheKeyFactory`服务，该实现会比较缓存键以将模型映射到正确的表。

请注意，该接口通常由数据库提供程序和其他扩展使用，一般不在应用程序代码中使用。但是对于我们的场景来说，这是一种可行的方法。

## 实现`IModelCacheKeyFactory`

我们需要使用`CreateDateTime`来区分表。在`DynamicContext`类中添加一个属性：

```
public DateTime CreateDateTime { get; set; }
```

在项目中添加一个名为`DynamicModelCacheKeyFactory.cs`的新类文件。代码如下所示：

```
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Infrastructure;

namespace DynamicModelDemo
{
    public class DynamicModelCacheKeyFactory : IModelCacheKeyFactory
    {
        public object Create(DbContext context)
            => context is DynamicContext dynamicContext
                ? (context.GetType(), dynamicContext.CreateDateTime)
                : (object)context.GetType();
    }
}
```

在生成模型缓存键时，此实现将考虑`CreateDateTime`属性。

## 应用`IModelCacheKeyFactory`

接下来，我们可以在上下文中注册新的`IModelCacheKeyFactory`：

```
#region OnConfiguring
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
            => optionsBuilder
                .UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=DynamicContext;Trusted_Connection=True;")
                .ReplaceService<IModelCacheKeyFactory, DynamicModelCacheKeyFactory>();
#endregion
```

这样我们就可以在`OnModelCreating`方法中分别映射表名了：

```
#region OnModelCreating
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
      modelBuilder.Entity<ConfigurableEntity>(b =>
            {
                b.ToTable(CreateDateTime.ToString("yyyyMMdd"));
                b.HasKey(p => p.Id);
            });
}
#endregion
```

`CreateDateTime`来自`DynamicContext`的属性。

我们可以在创建`DynamicContext`时指定`CreateDateTime`属性：

```
var context = new DynamicContext { CreateDateTime = datetime };
```

如果`datetime`为`2020/03/27`，则`context`的模型将映射到名为`20200327`的表。

## 创建数据库

在验证代码之前，我们需要首先创建数据库。但是，EF迁移并不是这种情况的最佳解决方案，因为随着时间的流逝，系统将生成更多表。我们只是使用它来创建一些示例表来验证映射。实际上，系统应该具有另一种每天动态生成表的方式。

运行以下命令以创建第一个迁移：

```
dotnet ef migrations add InitialCreate
```

您会看到在`Migrations`文件夹中生成了两个文件。打开`xxx_InitialCreate.cs`文件，并通过以下代码更新Up方法：

```
protected override void Up(MigrationBuilder migrationBuilder)
{
      for (int i = 0; i < 30; i++)
      {
           var index = i;
           migrationBuilder.CreateTable(
               name: DateTime.Now.AddDays(-index).ToString("yyyyMMdd"),
               columns: table => new
               {
                    Id = table.Column<int>(nullable: false)
                            .Annotation("SqlServer:Identity", "1, 1"),
                    Title = table.Column<string>(nullable: true),
                    Content = table.Column<string>(nullable: true),
                    CreateDateTime = table.Column<DateTime>(nullable: false)
               },
               constraints: table =>
               {
                    table.PrimaryKey($"PK_{DateTime.Now.AddDays(-index):yyyyMMdd}", x => x.Id);
               });
        }
    }
```

所做的更改是为了确保数据库中可以有足够的表进行测试。请注意，**我们不应该在生产环境中使用这种方式**。

接下来，我们可以使用此命令来创建和更新数据库：

```
dotnet ef database update
```

您会看到它在数据库中生成了最近30天的表。

## 验证映射

现在该验证新映射了。通过以下代码更新`Program.cs`中的`Main`方法：

```
static void Main(string[] args)
{
    DateTime datetime1 = DateTime.Now;
    using (var context = new DynamicContext { CreateDateTime = datetime1 })
    {
        context.Entities.Add(new ConfigurableEntity { Title = "Great News One", Content = $"Hello World! I am the news of {datetime1}", CreateDateTime = datetime1 });
        context.SaveChanges();
    }
    DateTime datetime2 = DateTime.Now.AddDays(-1);
    using (var context = new DynamicContext { CreateDateTime = datetime2 })
    {
        context.Entities.Add(new ConfigurableEntity { Title = "Great News Two", Content = $"Hello World! I am the news of {datetime2}", CreateDateTime = datetime2 });
        context.SaveChanges();
    }

    using (var context = new DynamicContext { CreateDateTime = datetime1 })
    {
        var entity = context.Entities.Single();
          // Writes news of today
        Console.WriteLine($"{entity.Title} {entity.Content} {entity.CreateDateTime}");
    }

    using (var context = new DynamicContext { CreateDateTime = datetime2 })
    {
        var entity = context.Entities.Single();
        // Writes news of yesterday
        Console.WriteLine($"{entity.Title} {entity.Content} {entity.CreateDateTime}");
    }
}
```

您将会看到如下输出：![img](https://mmbiz.qpic.cn/mmbiz_png/ahAATVIdckQ4CwpQmSvyw0hmrDBSIUn2rKApvOhnTs83Rw4E1DicATdOKx7zev9DD2pls2FTPfic3apnaX4BxgoA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

现在，我们可以通过传递`CreateDateTime`属性来使用相同的`DbContext`来表示不同的模型了。

## 小结

该演示旨在演示如何使用`IModelCacheKeyFactory`更改默认模型映射。请注意，您仍然需要实现分别生成表的方法。托管服务是一种实现方式。有关更多信息，请访问**Background tasks in ASP.NET Core**[1]。

### 参考资料

[1]Background tasks in ASP.NET Core: *https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-3.1&tabs=visual-studio*