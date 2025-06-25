## Eager Loading , Explicit Loading and Lazy Loading

These three are the strategies, how we retrieve related data from the database using ORM like Entity Framework Core. 

**Eager Loading:** Load the main entity and its related data when query is executed. ORM include related data using join in SQL and it fetching all data at once. In Entity Framework Core.

**Explicit Loading:** Load only main entity when query is executed, for loading related data needs to use separate commands. In Entity Framework Core Load() is used for querying related data explicitly. 

**Lazy Loading:** 


### Understanding the core concept using a simple example.

Suppose the Entity of a simple project is designed like that. Blog one to many relation Post and Other Entity. Post have an one to many Relation with Attachment and one to one relation with Author. 

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

Now I have to make a query for  the Post Title and AuthorName of a Blog. 

In eager loading the query will be like 

```csharp
public async Task<List<object>> GetBlogAuthorsWithPostTitlesAsync(int blogId)
{
    var blogPosts = await _context.Blogs
        .Where(b => b.Id == blogId)
        .Include(b => b.Posts) 
            .ThenInclude(p => p.Author)
        .SelectMany(b => b.Posts, (b, p) => new
        {
            PostTitle = p.Title,
            AuthorName = p.Author.Name
        })
        .ToListAsync();

    return blogPosts.Cast<object>().ToList();
}

```
#### How it work
- Query uses ".Include(b => b.Posts)"  to load Posts 
- ".Include(b => b.Posts)" for loading Author for each post.
- EF Core create a single query using join for fetching all the data
```sql
SELECT p.Title, a.Name
FROM Blogs b
INNER JOIN Posts p ON b.Id = p.BlogId
INNER JOIN Authors a ON p.AuthorId = a.Id
WHERE b.Id = 1
```




### 2. Explicit Loading
Load only main entity when query is executed, for loading related data needs to use seperate commands. In Entity Framework Core Load() is used for quering related data explicitly. 


### 3. Lazy Loading





