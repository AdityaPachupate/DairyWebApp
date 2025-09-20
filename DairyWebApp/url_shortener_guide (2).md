# URL Shortener API - Beginner-Friendly Implementation Guide

## Table of Contents
1. [Understanding What We're Building](#1-understanding-what-were-building)
2. [Project Setup and Structure](#2-project-setup-and-structure)
3. [Building the Foundation - Domain Layer](#3-building-the-foundation---domain-layer)
4. [Creating the Data Access - Infrastructure Layer](#4-creating-the-data-access---infrastructure-layer)
5. [Building Business Logic - Application Layer](#5-building-business-logic---application-layer)
6. [Creating the API - Presentation Layer](#6-creating-the-api---presentation-layer)
7. [Connecting Everything Together](#7-connecting-everything-together)
8. [Testing Your API](#8-testing-your-api)
9. [Adding Authentication (Bonus)](#9-adding-authentication-bonus)
10. [Adding Cloud Storage (Bonus)](#10-adding-cloud-storage-bonus)

---

## 1. Understanding What We're Building

Before we write any code, let's understand exactly what our URL shortener will do. Think of it like building a house - you need to understand what rooms you need before you start laying the foundation.

### Core Functionality
Our URL shortener will work like this:
1. **User submits a long URL**: "https://www.verylongdomainname.com/very/long/path/to/article"
2. **System creates a short code**: "abc123"
3. **User gets back a short URL**: "https://oursite.com/abc123"
4. **When someone visits the short URL**: They get redirected to the original long URL
5. **System tracks clicks**: Every time someone uses the short URL, we count it

### Why Clean Architecture?
Think of Clean Architecture like organizing a well-structured company. Each department (layer) has specific responsibilities and communicates through well-defined channels. This makes the system easier to understand, test, and modify. The beauty is that you can change how one department works without affecting the others, as long as they follow the agreed-upon communication protocols.

---

## 2. Project Setup and Structure

Let's start by creating the foundation of our project. This step is like setting up the folder structure for a big project - you want everything organized from the beginning.

### Step 2.1: Create the Solution and Projects

Think of a solution as a container that holds related projects, like a filing cabinet that contains different folders for different aspects of your business.

```bash
# Create the main container for our application
dotnet new sln -n UrlShortener

# Create individual projects for each layer
# API layer - this is where HTTP requests come in
dotnet new webapi -n UrlShortener.API

# Application layer - this is where business logic lives
dotnet new classlib -n UrlShortener.Application

# Domain layer - this is the heart of our business rules
dotnet new classlib -n UrlShortener.Domain

# Infrastructure layer - this handles databases and external services
dotnet new classlib -n UrlShortener.Infrastructure

# Add all projects to our solution container
dotnet sln add UrlShortener.API/UrlShortener.API.csproj
dotnet sln add UrlShortener.Application/UrlShortener.Application.csproj
dotnet sln add UrlShortener.Domain/UrlShortener.Domain.csproj
dotnet sln add UrlShortener.Infrastructure/UrlShortener.Infrastructure.csproj
```

### Step 2.2: Set Up Project Dependencies

Now we need to tell each project which other projects it can talk to. Think of this like defining which departments in your company can communicate directly with each other. The rule in Clean Architecture is that dependencies flow inward - outer layers can know about inner layers, but inner layers should never know about outer layers.

```bash
# API layer needs to know about Application and Infrastructure
# It's the entry point, so it coordinates everything
cd UrlShortener.API
dotnet add reference ../UrlShortener.Application/UrlShortener.Application.csproj
dotnet add reference ../UrlShortener.Infrastructure/UrlShortener.Infrastructure.csproj

# Application layer only needs to know about Domain
# It implements business use cases but doesn't care about technical details
cd ../UrlShortener.Application
dotnet add reference ../UrlShortener.Domain/UrlShortener.Domain.csproj

# Infrastructure layer knows about Application and Domain
# It implements the technical details that Application needs
cd ../UrlShortener.Infrastructure
dotnet add reference ../UrlShortener.Application/UrlShortener.Application.csproj
dotnet add reference ../UrlShortener.Domain/UrlShortener.Domain.csproj

# Domain layer has no dependencies - it's the pure business logic
# This is intentional - the core business rules shouldn't depend on anything external
```

### Step 2.3: Install Required Packages

Now let's add the external libraries we'll need. Think of these as specialized tools that help us accomplish specific tasks without reinventing the wheel.

```bash
# API Layer - tools for web API functionality
cd ../UrlShortener.API
dotnet add package Serilog.AspNetCore          # For logging what happens in our app
dotnet add package Serilog.Sinks.Console       # To see logs in the console
dotnet add package Serilog.Sinks.File          # To save logs to files
dotnet add package MediatR                     # For organizing our business logic commands

# Application Layer - tools for business logic
cd ../UrlShortener.Application
dotnet add package MediatR                     # For CQRS pattern
dotnet add package FluentValidation            # For validating user input
dotnet add package FluentValidation.DependencyInjectionExtensions
dotnet add package AutoMapper                  # For converting between different object types
dotnet add package AutoMapper.Extensions.Microsoft.DependencyInjection

# Infrastructure Layer - tools for data access
cd ../UrlShortener.Infrastructure
dotnet add package Microsoft.EntityFrameworkCore        # Main database tool
dotnet add package Microsoft.EntityFrameworkCore.SqlServer  # SQL Server database provider
dotnet add package Microsoft.EntityFrameworkCore.Tools      # Tools for database migrations
dotnet add package Microsoft.EntityFrameworkCore.Design     # Design-time tools
```

---

## 3. Building the Foundation - Domain Layer

The Domain layer is the heart of your application. It contains your core business entities and rules. Think of it as defining what a "Short URL" actually is in your business, regardless of how it gets stored or displayed.

### Step 3.1: Create the Base Entity

Every entity in our system will need a unique identifier. Instead of repeating this code in every entity, we create a base class. This is like creating a template that says "every important thing in our system has a unique ID."

**File**: `UrlShortener.Domain/Common/BaseEntity.cs`

```csharp
namespace UrlShortener.Domain.Common;

/// <summary>
/// Base class for all entities in our system
/// Ensures every entity has a unique identifier
/// </summary>
public abstract class BaseEntity
{
    /// <summary>
    /// Unique identifier for this entity
    /// Generated automatically when the entity is created
    /// Protected set means only this class and derived classes can change it
    /// </summary>
    public Guid Id { get; protected set; } = Guid.NewGuid();
}
```

**Why we designed it this way**: The `abstract` keyword means you can't create a `BaseEntity` directly - you must inherit from it. The `protected set` means the ID can only be changed by the entity itself or its children, preventing external code from accidentally changing an entity's identity.

### Step 3.2: Create the Core ShortUrl Entity

This is where we define what a Short URL actually is in our business domain. Every property and method here represents something meaningful to our business.

**File**: `UrlShortener.Domain/Entities/ShortUrl.cs`

```csharp
using UrlShortener.Domain.Common;

namespace UrlShortener.Domain.Entities;

/// <summary>
/// Represents a shortened URL in our system
/// This is our core business entity that contains all the rules about what makes a valid short URL
/// </summary>
public class ShortUrl : BaseEntity
{
    /// <summary>
    /// The original long URL that users want to shorten
    /// Private set ensures it can only be changed through our business methods
    /// </summary>
    public string OriginalUrl { get; private set; } = string.Empty;
    
    /// <summary>
    /// The short code that identifies this URL (e.g., "abc123")
    /// This is what appears in the shortened URL: https://oursite.com/abc123
    /// </summary>
    public string ShortCode { get; private set; } = string.Empty;
    
    /// <summary>
    /// How many times this short URL has been accessed
    /// Starts at 0 and can only be incremented through our business method
    /// </summary>
    public int Clicks { get; private set; }
    
    /// <summary>
    /// When this short URL was created
    /// Set once during creation and never changes
    /// </summary>
    public DateTime CreatedAt { get; private set; }
    
    /// <summary>
    /// Which user created this short URL (if any)
    /// Nullable because we might allow anonymous URL creation
    /// </summary>
    public string? UserId { get; private set; }

    /// <summary>
    /// Private constructor for Entity Framework
    /// EF needs a way to create entities when loading from database
    /// We make it private so application code can't use it accidentally
    /// </summary>
    private ShortUrl() { }

    /// <summary>
    /// Public constructor for creating new short URLs
    /// This is where we enforce our business rules about what makes a valid short URL
    /// </summary>
    /// <param name="originalUrl">The URL to be shortened - must not be empty</param>
    /// <param name="shortCode">The code to use - must not be empty</param>
    /// <param name="userId">Optional user who created this URL</param>
    public ShortUrl(string originalUrl, string shortCode, string? userId = null)
    {
        // Business rule: Original URL is required
        if (string.IsNullOrWhiteSpace(originalUrl))
            throw new ArgumentException("Original URL cannot be empty", nameof(originalUrl));
        
        // Business rule: Short code is required
        if (string.IsNullOrWhiteSpace(shortCode))
            throw new ArgumentException("Short code cannot be empty", nameof(shortCode));

        // Set the properties with validated values
        OriginalUrl = originalUrl;
        ShortCode = shortCode;
        UserId = userId;
        CreatedAt = DateTime.UtcNow;  // Always use UTC for consistency
        Clicks = 0;  // New URLs start with zero clicks
    }

    /// <summary>
    /// Business method to track when someone uses this short URL
    /// We encapsulate this logic to ensure clicks can only increase
    /// </summary>
    public void IncrementClicks()
    {
        Clicks++;
        // In the future, we could add business rules here like:
        // - Maximum clicks allowed
        // - Rate limiting
        // - Analytics tracking
    }
}
```

**Key design decisions explained**: 
- All properties have `private set` to ensure they can only be changed through our controlled business methods
- We validate input in the constructor to ensure we never create invalid entities
- The `IncrementClicks` method encapsulates the business logic of tracking usage
- We separate Entity Framework's needs (parameterless constructor) from our business needs (public constructor)

### Step 3.3: Create Custom Exceptions

Custom exceptions help us communicate specific business problems clearly. Instead of throwing generic exceptions, we create meaningful ones that tell us exactly what went wrong.

**File**: `UrlShortener.Domain/Exceptions/NotFoundException.cs`

```csharp
namespace UrlShortener.Domain.Exceptions;

/// <summary>
/// Thrown when we try to find something that doesn't exist
/// This is a business concern - "not found" is a valid business scenario
/// </summary>
public class NotFoundException : Exception
{
    /// <summary>
    /// Create exception with custom message
    /// Use this when you have a specific error message to communicate
    /// </summary>
    public NotFoundException(string message) : base(message) { }
    
    /// <summary>
    /// Create exception with standardized message format
    /// Use this for consistent error messages across the application
    /// </summary>
    /// <param name="name">Type of entity (e.g., "ShortUrl", "User")</param>
    /// <param name="key">The identifier we were looking for</param>
    public NotFoundException(string name, object key) 
        : base($"Entity \"{name}\" ({key}) was not found.") { }
}
```

**File**: `UrlShortener.Domain/Exceptions/DuplicateShortCodeException.cs`

```csharp
namespace UrlShortener.Domain.Exceptions;

/// <summary>
/// Thrown when someone tries to create a short code that already exists
/// This represents the business rule that short codes must be unique
/// </summary>
public class DuplicateShortCodeException : Exception
{
    public DuplicateShortCodeException(string shortCode) 
        : base($"Short code '{shortCode}' already exists.") { }
}
```

### Step 3.4: Define Repository Interface

This interface defines what our application needs from data storage, without specifying how that storage actually works. Think of it as a contract that says "I need to be able to find and save Short URLs, and I don't care if you use SQL, MongoDB, or even text files to do it."

**File**: `UrlShortener.Domain/Repositories/IShortUrlRepository.cs`

```csharp
using UrlShortener.Domain.Entities;

namespace UrlShortener.Domain.Repositories;

/// <summary>
/// Defines what operations we need to perform on Short URLs
/// This interface lives in Domain because it represents business needs, not technical implementation
/// The actual implementation will be in Infrastructure layer
/// </summary>
public interface IShortUrlRepository
{
    /// <summary>
    /// Find a Short URL by its short code (e.g., "abc123")
    /// Returns null if not found - this is a normal business scenario
    /// </summary>
    Task<ShortUrl?> GetByShortCodeAsync(string shortCode, CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Find a Short URL by its unique ID
    /// Useful for administrative operations
    /// </summary>
    Task<ShortUrl?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get all Short URLs created by a specific user
    /// Returns empty collection if user has no URLs (not null)
    /// </summary>
    Task<IEnumerable<ShortUrl>> GetByUserIdAsync(string userId, CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Save a new Short URL to storage
    /// Returns the Short URL with any changes made during storage (like database-generated timestamps)
    /// </summary>
    Task<ShortUrl> CreateAsync(ShortUrl shortUrl, CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Save changes to an existing Short URL
    /// Used for things like updating click counts
    /// </summary>
    Task UpdateAsync(ShortUrl shortUrl, CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Check if a short code is already taken
    /// More efficient than GetByShortCodeAsync when we only need to know existence
    /// </summary>
    Task<bool> ShortCodeExistsAsync(string shortCode, CancellationToken cancellationToken = default);
}
```

**Why this design works**: The interface is defined from the perspective of what the business logic needs, not from the perspective of how databases work. This means we can test our business logic without involving databases, and we can change database technologies without changing business logic.

---

## 4. Creating the Data Access - Infrastructure Layer

Now that we've defined what our data access should do (through the repository interface), let's implement how it actually works. This layer handles all the technical details of storing and retrieving data.

### Step 4.1: Create the Database Context

Entity Framework Core uses a "DbContext" to manage database connections and translate between our C# objects and database tables. Think of it as a translator that knows how to speak both C# and SQL.

**File**: `UrlShortener.Infrastructure/Data/ApplicationDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using UrlShortener.Domain.Entities;

namespace UrlShortener.Infrastructure.Data;

/// <summary>
/// The bridge between our C# objects and the database
/// This class tells Entity Framework how to map our entities to database tables
/// </summary>
public class ApplicationDbContext : DbContext
{
    /// <summary>
    /// Constructor that receives database configuration from dependency injection
    /// </summary>
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options)
    {
    }

    /// <summary>
    /// This property represents the ShortUrls table in our database
    /// Entity Framework will create and manage this table for us
    /// </summary>
    public DbSet<ShortUrl> ShortUrls { get; set; } = null!;

    /// <summary>
    /// This method is where we configure how our entities map to database tables
    /// It's like giving Entity Framework detailed instructions about our database schema
    /// </summary>
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Configure the ShortUrl entity
        modelBuilder.Entity<ShortUrl>(entity =>
        {
            // Tell EF that Id is the primary key (it usually figures this out automatically)
            entity.HasKey(e => e.Id);
            
            // Configure OriginalUrl column
            entity.Property(e => e.OriginalUrl)
                .IsRequired()                    // Cannot be null in database
                .HasMaxLength(2000);            // Limit URL length (URLs can be long!)
            
            // Configure ShortCode column
            entity.Property(e => e.ShortCode)
                .IsRequired()                    // Cannot be null in database
                .HasMaxLength(20);              // Short codes should be short!
            
            // Create a unique index on ShortCode
            // This ensures no two Short URLs can have the same code
            // The database will enforce this constraint for us
            entity.HasIndex(e => e.ShortCode)
                .IsUnique();
            
            // Configure CreatedAt column
            entity.Property(e => e.CreatedAt)
                .IsRequired();                   // Every Short URL must have a creation date
            
            // Configure Clicks column with default value
            entity.Property(e => e.Clicks)
                .HasDefaultValue(0);            // New Short URLs start with 0 clicks

            // Configure UserId column (nullable)
            entity.Property(e => e.UserId)
                .HasMaxLength(450);             // Standard length for ASP.NET Identity user IDs
        });
    }
}
```

**Why we configure the model explicitly**: While Entity Framework can guess a lot about our database schema, being explicit prevents surprises and documents our intentions. The unique index on ShortCode, for example, ensures our business rule about unique codes is enforced at the database level.

### Step 4.2: Implement the Repository

Now let's implement the actual data access logic. This class takes the interface we defined in the Domain layer and provides a concrete implementation using Entity Framework.

**File**: `UrlShortener.Infrastructure/Repositories/ShortUrlRepository.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using UrlShortener.Domain.Entities;
using UrlShortener.Domain.Repositories;
using UrlShortener.Infrastructure.Data;

namespace UrlShortener.Infrastructure.Repositories;

/// <summary>
/// Concrete implementation of IShortUrlRepository using Entity Framework Core
/// This class handles all the technical details of database access
/// </summary>
public class ShortUrlRepository : IShortUrlRepository
{
    private readonly ApplicationDbContext _context;

    /// <summary>
    /// Constructor receives the database context through dependency injection
    /// </summary>
    public ShortUrlRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    /// <summary>
    /// Find a Short URL by its short code
    /// AsNoTracking() improves performance when we're only reading data
    /// </summary>
    public async Task<ShortUrl?> GetByShortCodeAsync(string shortCode, CancellationToken cancellationToken = default)
    {
        return await _context.ShortUrls
            .AsNoTracking()  // We're not planning to modify this entity, so don't track changes
            .FirstOrDefaultAsync(x => x.ShortCode == shortCode, cancellationToken);
    }

    /// <summary>
    /// Find a Short URL by its unique ID
    /// </summary>
    public async Task<ShortUrl?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        return await _context.ShortUrls
            .AsNoTracking()
            .FirstOrDefaultAsync(x => x.Id == id, cancellationToken);
    }

    /// <summary>
    /// Get all Short URLs created by a specific user
    /// Ordered by creation date (newest first) for better user experience
    /// </summary>
    public async Task<IEnumerable<ShortUrl>> GetByUserIdAsync(string userId, CancellationToken cancellationToken = default)
    {
        return await _context.ShortUrls
            .AsNoTracking()
            .Where(x => x.UserId == userId)
            .OrderByDescending(x => x.CreatedAt)  // Show newest URLs first
            .ToListAsync(cancellationToken);
    }

    /// <summary>
    /// Save a new Short URL to the database
    /// Entity Framework will generate the SQL INSERT statement
    /// </summary>
    public async Task<ShortUrl> CreateAsync(ShortUrl shortUrl, CancellationToken cancellationToken = default)
    {
        _context.ShortUrls.Add(shortUrl);  // Tell EF to track this entity for insertion
        await _context.SaveChangesAsync(cancellationToken);  // Execute the database operation
        return shortUrl;  // Return the entity (might have been modified by database triggers, etc.)
    }

    /// <summary>
    /// Save changes to an existing Short URL
    /// Used for updating click counts and other modifications
    /// </summary>
    public async Task UpdateAsync(ShortUrl shortUrl, CancellationToken cancellationToken = default)
    {
        _context.ShortUrls.Update(shortUrl);  // Tell EF this entity has been modified
        await _context.SaveChangesAsync(cancellationToken);  // Execute the database UPDATE
    }

    /// <summary>
    /// Efficiently check if a short code already exists
    /// More efficient than GetByShortCodeAsync when we only need existence check
    /// </summary>
    public async Task<bool> ShortCodeExistsAsync(string shortCode, CancellationToken cancellationToken = default)
    {
        return await _context.ShortUrls
            .AsNoTracking()  // Don't need to track changes for existence check
            .AnyAsync(x => x.ShortCode == shortCode, cancellationToken);  // AnyAsync is more efficient than FirstOrDefault
    }
}
```

**Performance considerations explained**: The `AsNoTracking()` calls tell Entity Framework not to keep track of changes to these entities. This is more efficient when you're only reading data. The `AnyAsync()` method in `ShortCodeExistsAsync` is more efficient than loading the full entity when we only need to know if something exists.

### Step 4.3: Create Database Migration

Migrations are like version control for your database schema. They contain instructions for how to transform your database from one version to another.

```bash
# Navigate to the solution root
cd ..

# Create the initial migration
# This generates code that will create our database tables
dotnet ef migrations add InitialCreate --project UrlShortener.Infrastructure --startup-project UrlShortener.API

# Apply the migration to create the actual database
dotnet ef database update --project UrlShortener.Infrastructure --startup-project UrlShortener.API
```

**What this does**: The first command creates a migration file that contains C# code to create your database tables. The second command executes that code against your database. If you look in the `UrlShortener.Infrastructure/Migrations` folder, you'll see the generated migration files.

---

## 5. Building Business Logic - Application Layer

The Application layer orchestrates our business logic. It receives requests, validates them, coordinates with the domain entities, and returns appropriate responses. Think of it as the conductor of an orchestra - it doesn't play instruments itself, but it coordinates all the parts to create the final performance.

### Step 5.1: Create Data Transfer Objects (DTOs)

DTOs are like forms that define exactly what information goes in and out of our API. They provide a stable contract for our API while keeping our internal domain entities flexible.

**File**: `UrlShortener.Application/DTOs/ShortUrlDto.cs`

```csharp
namespace UrlShortener.Application.DTOs;

/// <summary>
/// Represents a Short URL as it appears in API responses
/// This is what external clients will see and work with
/// </summary>
public class ShortUrlDto
{
    /// <summary>
    /// Unique identifier for this Short URL
    /// </summary>
    public Guid Id { get; set; }
    
    /// <summary>
    /// The original long URL that was shortened
    /// </summary>
    public string OriginalUrl { get; set; } = string.Empty;
    
    /// <summary>
    /// The short code used in the shortened URL
    /// </summary>
    public string ShortCode { get; set; } = string.Empty;
    
    /// <summary>
    /// Number of times this URL has been accessed
    /// </summary>
    public int Clicks { get; set; }
    
    /// <summary>
    /// When this Short URL was created
    /// </summary>
    public DateTime CreatedAt { get; set; }
}

/// <summary>
/// Represents the data needed to create a new Short URL
/// This is what clients send when they want to shorten a URL
/// </summary>
public class CreateShortUrlDto
{
    /// <summary>
    /// The URL to be shortened (required)
    /// </summary>
    public string OriginalUrl { get; set; } = string.Empty;
    
    /// <summary>
    /// Optional custom code to use instead of generating one
    /// If not provided, system will generate a random code
    /// </summary>
    public string? CustomCode { get; set; }
}

/// <summary>
/// Represents statistics about a Short URL
/// Used for analytics endpoints
/// </summary>
public class ShortUrlStatsDto
{
    public string ShortCode { get; set; } = string.Empty;
    public string OriginalUrl { get; set; } = string.Empty;
    public int Clicks { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

**Why separate DTOs from entities**: DTOs define the external contract (what the API looks like), while entities define the internal business rules. This separation means you can change your internal implementation without breaking existing API clients.

### Step 5.2: Create AutoMapper Configuration

AutoMapper eliminates the tedious code of copying data between similar objects. Instead of writing manual property assignments, we configure the mappings once and use them everywhere.

**File**: `UrlShortener.Application/Mappings/MappingProfile.cs`

```csharp
using AutoMapper;
using UrlShortener.Application.DTOs;
using UrlShortener.Domain.Entities;

namespace UrlShortener.Application.Mappings;

/// <summary>
/// Configures how AutoMapper should convert between different object types
/// This eliminates boilerplate code for copying properties between similar objects
/// </summary>
public class MappingProfile : Profile
{
    public MappingProfile()
    {
        // Configure mapping from ShortUrl entity to ShortUrlDto
        // AutoMapper will automatically match properties with the same name
        CreateMap<ShortUrl, ShortUrlDto>();
        
        // Configure mapping from ShortUrl entity to ShortUrlStatsDto
        // This creates a different "view" of the same entity for statistics
        CreateMap<ShortUrl, ShortUrlStatsDto>();
    }
}
```

**How this works**: When you call `_mapper.Map<ShortUrlDto>(shortUrlEntity)`, AutoMapper uses this configuration to automatically copy matching properties from the entity to the DTO.

### Step 5.3: Create Application Services

These services contain business logic that doesn't belong to any specific entity. The short code generator is a perfect example - it's application logic, not domain logic.

**File**: `UrlShortener.Application/Services/IShortCodeGenerator.cs`

```csharp
namespace UrlShortener.Application.Services;

/// <summary>
/// Service for generating unique short codes
/// This is application logic - it's about how our system works, not core business rules
/// </summary>
public interface IShortCodeGenerator
{
    /// <summary>
    /// Generate a unique short code that doesn't already exist in the system
    /// Will keep trying until it finds one that's not taken
    /// </summary>
    Task<string> GenerateUniqueCodeAsync(CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Generate a random short code (might not be unique)
    /// Use this when you need a code for testing or other purposes
    /// </summary>
    string GenerateCode(int length = 6);
}
```

**File**: `UrlShortener.Application/Services/ShortCodeGenerator.cs`

```csharp
using UrlShortener.Domain.Repositories;

namespace UrlShortener.Application.Services;

/// <summary>
/// Implementation of short code generation logic
/// Generates random alphanumeric codes and ensures they're unique
/// </summary>
public class ShortCodeGenerator : IShortCodeGenerator
{
    private readonly IShortUrlRepository _repository;
    
    // Characters to use in short codes - excludes confusing characters like 0/O and 1/l
    private const string Characters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
    private readonly Random _random = new();

    public ShortCodeGenerator(IShortUrlRepository repository)
    {
        _repository = repository;
    }

    /// <summary>
    /// Generate a code that's guaranteed to be unique in our system
    /// Try up to 10 times with 6-character codes, then switch to 8-character codes
    /// </summary>
    public async Task<string> GenerateUniqueCodeAsync(CancellationToken cancellationToken = default)
    {
        const int maxAttempts = 10;
        
        // Try generating a 6-character code up to 10 times
        for (int attempt = 0; attempt < maxAttempts; attempt++)
        {
            var code = GenerateCode();
            
            // Check if this code is already taken
            if (!await _repository.ShortCodeExistsAsync(code, cancellationToken))
            {
                return code;  // Found a unique code!
            }
        }

        // If we couldn't find a unique 6-character code, try with 8 characters
        // This dramatically reduces collision probability
        return GenerateCode(8);
    }

    /// <summary>
    /// Generate a random code of specified length
    /// Each character is randomly selected from our character set
    /// </summary>
    public string GenerateCode(int length = 6)
    {
        var code = new char[length];
        
        for (int i = 0; i < length; i++)
        {
            code[i] = Characters[_random.Next(Characters.Length)];
        }
        
        return new string(code);
    }
}
```

**Algorithm explanation**: We use a simple retry mechanism for uniqueness. With 62 possible characters (a-z, A-Z, 0-9) and 6 positions, we have 62^6 = over 56 billion possible codes. The chance of collision is extremely low, but we check anyway for absolute certainty.

### Step 5.4: Create