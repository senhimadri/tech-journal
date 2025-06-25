## Eager Loading , Explicit Loading and Lazy Loading

These three are the strategies, how we retrieve related data from the database using ORM like Entity Framework Core. 

### 1. Eager Loading

Load the main entity and its related data when query is executed. ORM include related data using join in SQL and it fatch all data at once. In Entity Framework Core
.Inclide() is used to load the related entities.

Lets explain it using an example. Suppose the Entitys is designed like that.

```csharp
public class Blog
{
    public int Id { get; set; }
    public List<Post> Posts { get; set; } = new List<Post>();
    public List<Other> Others { get; set; } = new List<Other>();
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Description { get; set; }
    public List<Attachment> Attachments { get; set; } = new List<Attachment>();
    public Guid AuthorId { get; set; }
    public Author Author { get; set; }
    public int BlogId { get; set; } // Foreign key to Blog
    public Blog Blog { get; set; } // Navigation property (optional for this example)
}

public class Attachment
{
    public int Id { get; set; }
    public string FileAddress { get; set; }
    public int PostId { get; set; } // Foreign key to Post
    public Post Post { get; set; } // Navigation property
}

public class Author
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public List<Post> Posts { get; set; } = new List<Post>(); // Inverse navigation
}

public class Other
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int BlogId { get; set; } // Foreign key to Blog
    public Blog Blog { get; set; }
} 
```





### 2. Explicit Loading
Load only main entity when query is executed, for loading related data needs to use seperate commands. In Entity Framework Core Load() is used for quering related data explicitly. 


### 3. Lazy Loading

