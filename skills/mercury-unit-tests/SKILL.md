---
name: mercury-unit-tests
description: Generate comprehensive xUnit unit tests following Mercury C# standards with NSubstitute mocking and FluentAssertions
tools:
  - view
  - create
license: MIT
---

# Mercury Unit Testing Standards

Generate high-quality unit tests for Mercury C# projects following established team conventions.

## Framework Requirements

**Testing Stack:**
- **xUnit** - Testing framework (never MSTest or NUnit)
- **NSubstitute** - Mocking library (never Moq)
- **FluentAssertions** - Assertion library for readable tests

## Test Structure & Naming

### Naming Convention
Format: `MethodName_Scenario_ExpectedResult`

**Examples:**
- `ExtendLoan_ValidLoanId_ReturnsSuccess`
- `ExtendLoan_ExpiredLoan_ReturnsError`
- `ExtendLoan_NullLoanId_ThrowsArgumentNullException`

### Test Organization
1. **Follow AAA Pattern:**
   - `// Arrange` - Setup test data and mocks
   - `// Act` - Execute the method under test
   - `// Assert` - Verify expected outcomes

2. **One Logical Assertion Per Test** - Each test validates one specific behavior

3. **Group Related Tests** - Use nested classes for method-specific test groups:
   ```csharp
   public class LoanServiceTests
   {
       public class ExtendLoanTests { /* tests here */ }
       public class ReturnLoanTests { /* tests here */ }
   }
   ```

## Coverage Requirements

Generate tests for ALL these scenarios:

| Test Type | Minimum Count | Purpose |
|-----------|---------------|---------|
| **Positive Cases** | 1+ | Verify happy path with valid inputs |
| **Negative Cases** | 2+ | Verify error handling and exceptions |
| **Edge Cases** | 2+ | Test null, empty, boundaries, limits |

**Target:** 80%+ code coverage minimum

## Mocking Standards

### Always Mock These Dependencies:
- ✅ **Database Contexts** - `DbContext`, `IRepository<T>`
- ✅ **External APIs** - `HttpClient`, `IApiClient`
- ✅ **File System** - `IFileSystem`, `IFile`, `IDirectory`
- ✅ **Time/Date** - `IDateTimeProvider`, `IClock`
- ✅ **Communication** - `IEmailService`, `INotificationService`
- ✅ **Logging** - `ILogger<T>`

### Mocking Syntax:
```csharp
// Create mock
var mockRepo = Substitute.For<ILoanRepository>();

// Setup return value
mockRepo.GetLoanAsync(1).Returns(loan);

// Verify method called
await mockRepo.Received(1).GetLoanAsync(1);
```

## Complete Example

```csharp
using Xunit;
using NSubstitute;
using FluentAssertions;
using Library.ApplicationCore.Services;
using Library.ApplicationCore.Interfaces;
using Library.ApplicationCore.Entities;
using Library.ApplicationCore.Enums;

namespace UnitTests.ApplicationCore.Services;

public class LoanServiceTests
{
    private readonly ILoanRepository _mockLoanRepo;
    private readonly IDateTimeProvider _mockDateProvider;
    private readonly LoanService _sut; // System Under Test
    
    public LoanServiceTests()
    {
        _mockLoanRepo = Substitute.For<ILoanRepository>();
        _mockDateProvider = Substitute.For<IDateTimeProvider>();
        _sut = new LoanService(_mockLoanRepo, _mockDateProvider);
    }
    
    [Fact]
    public async Task ExtendLoan_ValidLoanId_ReturnsSuccess()
    {
        // Arrange
        var loanId = 1;
        var loan = new Loan { Id = loanId, IsActive = true, DueDate = DateTime.UtcNow.AddDays(7) };
        _mockLoanRepo.GetLoanAsync(loanId).Returns(loan);
        _mockDateProvider.UtcNow.Returns(DateTime.UtcNow);
        
        // Act
        var result = await _sut.ExtendLoan(loanId);
        
        // Assert
        result.Should().Be(LoanExtensionStatus.Success);
        await _mockLoanRepo.Received(1).GetLoanAsync(loanId);
        await _mockLoanRepo.Received(1).UpdateLoanAsync(Arg.Is<Loan>(l => l.Id == loanId));
    }
    
    [Fact]
    public async Task ExtendLoan_ExpiredLoan_ReturnsError()
    {
        // Arrange
        var loanId = 1;
        var loan = new Loan { Id = loanId, IsActive = true, DueDate = DateTime.UtcNow.AddDays(-5) };
        _mockLoanRepo.GetLoanAsync(loanId).Returns(loan);
        _mockDateProvider.UtcNow.Returns(DateTime.UtcNow);
        
        // Act
        var result = await _sut.ExtendLoan(loanId);
        
        // Assert
        result.Should().Be(LoanExtensionStatus.LoanOverdue);
    }
    
    [Fact]
    public async Task ExtendLoan_NullLoanId_ThrowsArgumentNullException()
    {
        // Arrange
        int? loanId = null;
        
        // Act
        Func<Task> act = async () => await _sut.ExtendLoan(loanId.Value);
        
        // Assert
        await act.Should().ThrowAsync<ArgumentNullException>();
    }
}
```

## Generation Workflow

When asked to generate tests, follow these steps:

### 1. Analyze Target
- Read the class/method to understand functionality
- Identify method signature (parameters, return type)
- Note any async/await patterns

### 2. Identify Dependencies
- Examine constructor parameters
- List all interfaces to mock
- Identify external dependencies (DB, APIs, filesystem)

### 3. Plan Test Scenarios
Create test matrix:
- **Positive:** Valid input → expected output (minimum 1)
- **Negative:** Invalid input → error/exception (minimum 2)
- **Edge:** Null, empty string, max values, boundaries (minimum 2)

### 4. Generate Complete Test Class
Include ALL of these:
```csharp
// Required usings
using Xunit;
using NSubstitute;
using FluentAssertions;
using [ProjectNamespace];

// Proper namespace
namespace UnitTests.[Path.To.Class];

// Test class with fixture
public class [ClassName]Tests
{
    // Mock fields
    private readonly IMockInterface _mockDependency;
    
    // System Under Test
    private readonly [ClassName] _sut;
    
    // Constructor for shared setup
    public [ClassName]Tests() { }
    
    // Test methods with [Fact] or [Theory]
}
```

### 5. Verify Quality
Ensure generated tests include:
- ✅ Correct naming convention
- ✅ AAA pattern with comments
- ✅ All dependencies mocked
- ✅ Positive, negative, and edge cases
- ✅ Proper assertions using FluentAssertions
- ✅ Async/await if method is async
- ✅ Received() verification for important calls

## Common Patterns

### Theory with InlineData
For similar tests with different inputs:
```csharp
[Theory]
[InlineData(1, LoanExtensionStatus.Success)]
[InlineData(999, LoanExtensionStatus.NotFound)]
public async Task ExtendLoan_VariousIds_ReturnsExpectedStatus(
    int loanId, 
    LoanExtensionStatus expected)
{
    // Test implementation
}
```

### Exception Testing
```csharp
[Fact]
public async Task Method_InvalidInput_ThrowsException()
{
    // Arrange
    var invalidInput = "";
    
    // Act
    Func<Task> act = async () => await _sut.Method(invalidInput);
    
    // Assert
    await act.Should().ThrowAsync<ArgumentException>()
        .WithMessage("*cannot be empty*");
}
```

### Verify No Calls
```csharp
// Assert no calls were made
await _mockRepo.DidNotReceive().DeleteAsync(Arg.Any<int>());
```

---

**Remember:** Every test should be independent, repeatable, and fast. Always verify the behavior, not the implementation.
