# ASP.NET Core Identity

> Official Documentation: https://learn.microsoft.com/aspnet/core/security/authentication/identity

## Overview

ASP.NET Core Identity is a membership system that provides user registration, login, role management, claims, two-factor authentication, external login providers, and account management. It uses Entity Framework Core for data persistence and provides both UI scaffolding for MVC/Razor Pages applications and API endpoints for SPAs and mobile apps. In .NET 8+, Identity includes built-in API endpoints via `MapIdentityApi<T>()`.

---

## 1. Basic Setup

### Installing Identity

```bash
# Core Identity packages
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer   # or .Npgsql for PostgreSQL

# For API endpoints (.NET 8+)
dotnet add package Microsoft.AspNetCore.Identity.UI  # Optional, for scaffolded UI
```

### Configuration

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add DbContext
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("Default")));

// Add Identity with default settings
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    // Password policy
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequiredLength = 8;
    options.Password.RequiredUniqueChars = 4;

    // Lockout policy
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;

    // User settings
    options.User.AllowedUserNameCharacters =
        "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-._@+";
    options.User.RequireUniqueEmail = true;

    // Sign-in settings
    options.SignIn.RequireConfirmedEmail = true;
    options.SignIn.RequireConfirmedPhoneNumber = false;
    options.SignIn.RequireConfirmedAccount = true;

    // Token settings
    options.Tokens.EmailConfirmationTokenProvider = TokenOptions.DefaultEmailProvider;
    options.Tokens.PasswordResetTokenProvider = TokenOptions.DefaultProvider;
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.Run();
```

---

## 2. Custom User Class

```csharp
using Microsoft.AspNetCore.Identity;

namespace MyApp.Models;

public class ApplicationUser : IdentityUser
{
    [MaxLength(100)]
    public required string FirstName { get; set; }

    [MaxLength(100)]
    public required string LastName { get; set; }

    public string FullName => $"{FirstName} {LastName}";

    public DateTime DateOfBirth { get; set; }

    [MaxLength(500)]
    public string? AvatarUrl { get; set; }

    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    public DateTime? LastLoginAt { get; set; }

    public bool IsActive { get; set; } = true;

    [MaxLength(50)]
    public string? RefreshToken { get; set; }

    public DateTime? RefreshTokenExpiryTime { get; set; }

    // Navigation properties
    public ICollection<Order> Orders { get; set; } = new List<Order>();
    public UserProfile? Profile { get; set; }
}

// Custom role with additional properties
public class ApplicationRole : IdentityRole
{
    [MaxLength(500)]
    public string? Description { get; set; }

    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

### DbContext Setup

```csharp
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace MyApp.Data;

public class AppDbContext : IdentityDbContext<ApplicationUser, ApplicationRole, string>
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<UserProfile> UserProfiles => Set<UserProfile>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder); // Required - configures Identity tables

        // Customize table names (optional)
        builder.Entity<ApplicationUser>(entity =>
        {
            entity.ToTable("Users");
            entity.Property(e => e.FirstName).HasMaxLength(100).IsRequired();
            entity.Property(e => e.LastName).HasMaxLength(100).IsRequired();
        });

        builder.Entity<ApplicationRole>(entity =>
        {
            entity.ToTable("Roles");
        });

        builder.Entity<IdentityUserRole<string>>(entity =>
        {
            entity.ToTable("UserRoles");
        });

        builder.Entity<IdentityUserClaim<string>>(entity =>
        {
            entity.ToTable("UserClaims");
        });

        builder.Entity<IdentityUserLogin<string>>(entity =>
        {
            entity.ToTable("UserLogins");
        });

        builder.Entity<IdentityRoleClaim<string>>(entity =>
        {
            entity.ToTable("RoleClaims");
        });

        builder.Entity<IdentityUserToken<string>>(entity =>
        {
            entity.ToTable("UserTokens");
        });
    }
}
```

```bash
# Create migration
dotnet ef migrations add AddIdentity

# Apply migration
dotnet ef database update
```

---

## 3. UserManager<T> Operations

`UserManager<T>` is the primary API for managing users.

### Common Operations

| Method | Purpose | Returns |
|---|---|---|
| `CreateAsync(user, password)` | Register new user | `IdentityResult` |
| `FindByIdAsync(id)` | Find user by ID | `ApplicationUser?` |
| `FindByEmailAsync(email)` | Find user by email | `ApplicationUser?` |
| `FindByNameAsync(username)` | Find user by username | `ApplicationUser?` |
| `UpdateAsync(user)` | Update user | `IdentityResult` |
| `DeleteAsync(user)` | Delete user | `IdentityResult` |
| `CheckPasswordAsync(user, password)` | Verify password | `bool` |
| `ChangePasswordAsync(user, current, new)` | Change password | `IdentityResult` |
| `AddToRoleAsync(user, role)` | Assign role | `IdentityResult` |
| `GetRolesAsync(user)` | Get user roles | `IList<string>` |
| `GenerateEmailConfirmationTokenAsync(user)` | Generate email token | `string` |
| `ConfirmEmailAsync(user, token)` | Confirm email | `IdentityResult` |
| `GeneratePasswordResetTokenAsync(user)` | Generate reset token | `string` |
| `ResetPasswordAsync(user, token, newPassword)` | Reset password | `IdentityResult` |
| `GetClaimsAsync(user)` | Get user claims | `IList<Claim>` |
| `AddClaimAsync(user, claim)` | Add claim | `IdentityResult` |

### Authentication Service Example

```csharp
namespace MyApp.Services;

public interface IAuthService
{
    Task<AuthResult> RegisterAsync(RegisterDto dto);
    Task<AuthResult> LoginAsync(LoginDto dto);
    Task<AuthResult> RefreshTokenAsync(RefreshTokenDto dto);
    Task<IdentityResult> ConfirmEmailAsync(string userId, string token);
    Task<IdentityResult> ForgotPasswordAsync(string email);
    Task<IdentityResult> ResetPasswordAsync(ResetPasswordDto dto);
}

public class AuthService(
    UserManager<ApplicationUser> userManager,
    SignInManager<ApplicationUser> signInManager,
    ITokenService tokenService,
    IEmailSender emailSender,
    ILogger<AuthService> logger) : IAuthService
{
    public async Task<AuthResult> RegisterAsync(RegisterDto dto)
    {
        var existingUser = await userManager.FindByEmailAsync(dto.Email);
        if (existingUser is not null)
        {
            return AuthResult.Failure("Email is already registered");
        }

        var user = new ApplicationUser
        {
            UserName = dto.Email,
            Email = dto.Email,
            FirstName = dto.FirstName,
            LastName = dto.LastName
        };

        var result = await userManager.CreateAsync(user, dto.Password);
        if (!result.Succeeded)
        {
            return AuthResult.Failure(result.Errors.Select(e => e.Description));
        }

        // Assign default role
        await userManager.AddToRoleAsync(user, "User");

        // Add custom claims
        await userManager.AddClaimAsync(user,
            new Claim("registration_date", DateTime.UtcNow.ToString("O")));

        // Generate and send email confirmation
        var token = await userManager.GenerateEmailConfirmationTokenAsync(user);
        var encodedToken = WebEncoders.Base64UrlEncode(Encoding.UTF8.GetBytes(token));
        var confirmationLink = $"https://myapp.com/confirm-email?userId={user.Id}&token={encodedToken}";

        await emailSender.SendEmailAsync(
            user.Email,
            "Confirm your email",
            $"Please confirm your email by clicking: {confirmationLink}");

        logger.LogInformation("User {UserId} registered successfully", user.Id);

        return AuthResult.Success("Registration successful. Please check your email.");
    }

    public async Task<AuthResult> LoginAsync(LoginDto dto)
    {
        var user = await userManager.FindByEmailAsync(dto.Email);
        if (user is null || !user.IsActive)
        {
            return AuthResult.Failure("Invalid email or password");
        }

        // Check if account is locked out
        if (await userManager.IsLockedOutAsync(user))
        {
            var lockoutEnd = await userManager.GetLockoutEndDateAsync(user);
            logger.LogWarning("Locked out user {UserId} attempted login", user.Id);
            return AuthResult.Failure($"Account is locked. Try again at {lockoutEnd}");
        }

        // Verify password
        var result = await signInManager.CheckPasswordSignInAsync(
            user, dto.Password, lockoutOnFailure: true);

        if (!result.Succeeded)
        {
            if (result.IsLockedOut)
                return AuthResult.Failure("Account locked due to too many failed attempts");
            if (result.IsNotAllowed)
                return AuthResult.Failure("Email confirmation required");
            if (result.RequiresTwoFactor)
                return AuthResult.RequiresTwoFactor(user.Id);

            return AuthResult.Failure("Invalid email or password");
        }

        // Generate tokens
        var roles = await userManager.GetRolesAsync(user);
        var accessToken = tokenService.GenerateAccessToken(user, roles);
        var refreshToken = tokenService.GenerateRefreshToken();

        // Store refresh token
        user.RefreshToken = refreshToken;
        user.RefreshTokenExpiryTime = DateTime.UtcNow.AddDays(7);
        user.LastLoginAt = DateTime.UtcNow;
        await userManager.UpdateAsync(user);

        logger.LogInformation("User {UserId} logged in", user.Id);

        return AuthResult.Success(accessToken, refreshToken);
    }

    public async Task<AuthResult> RefreshTokenAsync(RefreshTokenDto dto)
    {
        var principal = tokenService.ValidateExpiredToken(dto.AccessToken);
        if (principal is null)
            return AuthResult.Failure("Invalid access token");

        var userId = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (userId is null)
            return AuthResult.Failure("Invalid token claims");

        var user = await userManager.FindByIdAsync(userId);
        if (user is null ||
            user.RefreshToken != dto.RefreshToken ||
            user.RefreshTokenExpiryTime <= DateTime.UtcNow)
        {
            return AuthResult.Failure("Invalid or expired refresh token");
        }

        var roles = await userManager.GetRolesAsync(user);
        var newAccessToken = tokenService.GenerateAccessToken(user, roles);
        var newRefreshToken = tokenService.GenerateRefreshToken();

        user.RefreshToken = newRefreshToken;
        user.RefreshTokenExpiryTime = DateTime.UtcNow.AddDays(7);
        await userManager.UpdateAsync(user);

        return AuthResult.Success(newAccessToken, newRefreshToken);
    }

    public async Task<IdentityResult> ConfirmEmailAsync(string userId, string token)
    {
        var user = await userManager.FindByIdAsync(userId);
        if (user is null)
            return IdentityResult.Failed(new IdentityError
            {
                Description = "User not found"
            });

        var decodedToken = Encoding.UTF8.GetString(WebEncoders.Base64UrlDecode(token));
        return await userManager.ConfirmEmailAsync(user, decodedToken);
    }

    public async Task<IdentityResult> ForgotPasswordAsync(string email)
    {
        var user = await userManager.FindByEmailAsync(email);
        if (user is null || !await userManager.IsEmailConfirmedAsync(user))
        {
            // Do not reveal that the user does not exist
            return IdentityResult.Success;
        }

        var token = await userManager.GeneratePasswordResetTokenAsync(user);
        var encodedToken = WebEncoders.Base64UrlEncode(Encoding.UTF8.GetBytes(token));
        var resetLink = $"https://myapp.com/reset-password?email={email}&token={encodedToken}";

        await emailSender.SendEmailAsync(
            email,
            "Reset your password",
            $"Reset your password by clicking: {resetLink}");

        return IdentityResult.Success;
    }

    public async Task<IdentityResult> ResetPasswordAsync(ResetPasswordDto dto)
    {
        var user = await userManager.FindByEmailAsync(dto.Email);
        if (user is null)
            return IdentityResult.Failed(new IdentityError
            {
                Description = "Invalid request"
            });

        var decodedToken = Encoding.UTF8.GetString(
            WebEncoders.Base64UrlDecode(dto.Token));

        return await userManager.ResetPasswordAsync(user, decodedToken, dto.NewPassword);
    }
}

// DTOs
public record RegisterDto(
    [Required] string FirstName,
    [Required] string LastName,
    [Required, EmailAddress] string Email,
    [Required, MinLength(8)] string Password);

public record LoginDto(
    [Required, EmailAddress] string Email,
    [Required] string Password);

public record RefreshTokenDto(
    [Required] string AccessToken,
    [Required] string RefreshToken);

public record ResetPasswordDto(
    [Required, EmailAddress] string Email,
    [Required] string Token,
    [Required, MinLength(8)] string NewPassword);

public class AuthResult
{
    public bool IsSuccess { get; init; }
    public string? Message { get; init; }
    public string? AccessToken { get; init; }
    public string? RefreshToken { get; init; }
    public IEnumerable<string>? Errors { get; init; }
    public bool RequiresTwoFactorAuth { get; init; }
    public string? TwoFactorUserId { get; init; }

    public static AuthResult Success(string accessToken, string refreshToken) =>
        new() { IsSuccess = true, AccessToken = accessToken, RefreshToken = refreshToken };

    public static AuthResult Success(string message) =>
        new() { IsSuccess = true, Message = message };

    public static AuthResult Failure(string error) =>
        new() { IsSuccess = false, Errors = new[] { error } };

    public static AuthResult Failure(IEnumerable<string> errors) =>
        new() { IsSuccess = false, Errors = errors };

    public static AuthResult RequiresTwoFactor(string userId) =>
        new() { RequiresTwoFactorAuth = true, TwoFactorUserId = userId };
}
```

---

## 4. Role Management

```csharp
public class RoleService(
    RoleManager<ApplicationRole> roleManager,
    UserManager<ApplicationUser> userManager)
{
    public async Task SeedRolesAsync()
    {
        string[] roles = ["Admin", "Manager", "User", "ReadOnly"];

        foreach (var role in roles)
        {
            if (!await roleManager.RoleExistsAsync(role))
            {
                await roleManager.CreateAsync(new ApplicationRole
                {
                    Name = role,
                    Description = $"{role} role with default permissions"
                });
            }
        }
    }

    public async Task<IdentityResult> AssignRoleAsync(string userId, string roleName)
    {
        var user = await userManager.FindByIdAsync(userId)
            ?? throw new NotFoundException("User", userId);

        if (!await roleManager.RoleExistsAsync(roleName))
            throw new NotFoundException("Role", roleName);

        if (await userManager.IsInRoleAsync(user, roleName))
            return IdentityResult.Success;

        return await userManager.AddToRoleAsync(user, roleName);
    }

    public async Task<IdentityResult> RemoveRoleAsync(string userId, string roleName)
    {
        var user = await userManager.FindByIdAsync(userId)
            ?? throw new NotFoundException("User", userId);

        return await userManager.RemoveFromRoleAsync(user, roleName);
    }

    public async Task<IList<string>> GetUserRolesAsync(string userId)
    {
        var user = await userManager.FindByIdAsync(userId)
            ?? throw new NotFoundException("User", userId);

        return await userManager.GetRolesAsync(user);
    }

    public async Task<IList<ApplicationUser>> GetUsersInRoleAsync(string roleName)
    {
        return await userManager.GetUsersInRoleAsync(roleName);
    }
}

// Seed roles at startup
using (var scope = app.Services.CreateScope())
{
    var roleService = scope.ServiceProvider.GetRequiredService<RoleService>();
    await roleService.SeedRolesAsync();
}
```

---

## 5. Claims-Based Identity

```csharp
public class ClaimsService(UserManager<ApplicationUser> userManager)
{
    public async Task AddUserClaimsAsync(ApplicationUser user)
    {
        var claims = new List<Claim>
        {
            new("full_name", user.FullName),
            new("email_verified", user.EmailConfirmed.ToString().ToLower()),
            new("created_at", user.CreatedAt.ToString("O")),
            new("subscription", "premium"),
            new("department", "Engineering")
        };

        await userManager.AddClaimsAsync(user, claims);
    }

    public async Task<IList<Claim>> GetUserClaimsAsync(string userId)
    {
        var user = await userManager.FindByIdAsync(userId)
            ?? throw new NotFoundException("User", userId);

        return await userManager.GetClaimsAsync(user);
    }

    public async Task ReplaceClaimAsync(string userId, string claimType, string newValue)
    {
        var user = await userManager.FindByIdAsync(userId)
            ?? throw new NotFoundException("User", userId);

        var claims = await userManager.GetClaimsAsync(user);
        var existingClaim = claims.FirstOrDefault(c => c.Type == claimType);

        if (existingClaim is not null)
        {
            await userManager.ReplaceClaimAsync(user, existingClaim,
                new Claim(claimType, newValue));
        }
        else
        {
            await userManager.AddClaimAsync(user, new Claim(claimType, newValue));
        }
    }
}

// Custom claims transformation (add claims on every request)
public class CustomClaimsTransformation(
    UserManager<ApplicationUser> userManager) : IClaimsTransformation
{
    public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        var identity = principal.Identity as ClaimsIdentity;
        if (identity is null || !identity.IsAuthenticated) return principal;

        var userId = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (userId is null) return principal;

        var user = await userManager.FindByIdAsync(userId);
        if (user is null) return principal;

        // Add claims that are not in the JWT
        if (!principal.HasClaim(c => c.Type == "full_name"))
        {
            identity.AddClaim(new Claim("full_name", user.FullName));
        }

        if (!principal.HasClaim(c => c.Type == "is_active"))
        {
            identity.AddClaim(new Claim("is_active", user.IsActive.ToString().ToLower()));
        }

        return principal;
    }
}

builder.Services.AddScoped<IClaimsTransformation, CustomClaimsTransformation>();
```

---

## 6. External Login Providers

```csharp
builder.Services.AddAuthentication()
    .AddGoogle(options =>
    {
        options.ClientId = builder.Configuration["Authentication:Google:ClientId"]!;
        options.ClientSecret = builder.Configuration["Authentication:Google:ClientSecret"]!;
        options.Scope.Add("profile");
        options.Scope.Add("email");
    })
    .AddMicrosoftAccount(options =>
    {
        options.ClientId = builder.Configuration["Authentication:Microsoft:ClientId"]!;
        options.ClientSecret = builder.Configuration["Authentication:Microsoft:ClientSecret"]!;
    })
    .AddGitHub(options =>
    {
        options.ClientId = builder.Configuration["Authentication:GitHub:ClientId"]!;
        options.ClientSecret = builder.Configuration["Authentication:GitHub:ClientSecret"]!;
        options.Scope.Add("user:email");
    });

// External login handling
public class ExternalAuthService(
    UserManager<ApplicationUser> userManager,
    SignInManager<ApplicationUser> signInManager,
    ITokenService tokenService,
    ILogger<ExternalAuthService> logger)
{
    public async Task<AuthResult> HandleExternalLoginAsync(
        ExternalLoginInfo info)
    {
        // Try to sign in with existing external login
        var signInResult = await signInManager.ExternalLoginSignInAsync(
            info.LoginProvider,
            info.ProviderKey,
            isPersistent: false);

        if (signInResult.Succeeded)
        {
            var existingUser = await userManager.FindByLoginAsync(
                info.LoginProvider, info.ProviderKey);
            var roles = await userManager.GetRolesAsync(existingUser!);
            return AuthResult.Success(
                tokenService.GenerateAccessToken(existingUser!, roles),
                tokenService.GenerateRefreshToken());
        }

        // Create new user from external provider
        var email = info.Principal.FindFirst(ClaimTypes.Email)?.Value;
        if (email is null)
            return AuthResult.Failure("Email not provided by external provider");

        var user = await userManager.FindByEmailAsync(email);
        if (user is null)
        {
            user = new ApplicationUser
            {
                UserName = email,
                Email = email,
                FirstName = info.Principal.FindFirst(ClaimTypes.GivenName)?.Value ?? "",
                LastName = info.Principal.FindFirst(ClaimTypes.Surname)?.Value ?? "",
                EmailConfirmed = true // Trust external provider
            };

            var createResult = await userManager.CreateAsync(user);
            if (!createResult.Succeeded)
                return AuthResult.Failure(createResult.Errors.Select(e => e.Description));

            await userManager.AddToRoleAsync(user, "User");
        }

        // Link external login to user
        var addLoginResult = await userManager.AddLoginAsync(user, info);
        if (!addLoginResult.Succeeded)
            return AuthResult.Failure(addLoginResult.Errors.Select(e => e.Description));

        var userRoles = await userManager.GetRolesAsync(user);
        logger.LogInformation("User {UserId} linked {Provider} login", user.Id, info.LoginProvider);

        return AuthResult.Success(
            tokenService.GenerateAccessToken(user, userRoles),
            tokenService.GenerateRefreshToken());
    }
}
```

---

## 7. Two-Factor Authentication (2FA)

```csharp
public class TwoFactorService(
    UserManager<ApplicationUser> userManager,
    ITokenService tokenService)
{
    public async Task<TwoFactorSetupDto> EnableTwoFactorAsync(string userId)
    {
        var user = await userManager.FindByIdAsync(userId)
            ?? throw new NotFoundException("User", userId);

        // Generate authenticator key
        await userManager.ResetAuthenticatorKeyAsync(user);
        var key = await userManager.GetAuthenticatorKeyAsync(user);

        var qrCodeUri = $"otpauth://totp/MyApp:{user.Email}?secret={key}&issuer=MyApp&digits=6";

        return new TwoFactorSetupDto(key!, qrCodeUri);
    }

    public async Task<IdentityResult> VerifyAndEnableTwoFactorAsync(
        string userId, string verificationCode)
    {
        var user = await userManager.FindByIdAsync(userId)
            ?? throw new NotFoundException("User", userId);

        var isValid = await userManager.VerifyTwoFactorTokenAsync(
            user,
            userManager.Options.Tokens.AuthenticatorTokenProvider,
            verificationCode);

        if (!isValid)
            return IdentityResult.Failed(new IdentityError
            {
                Description = "Invalid verification code"
            });

        await userManager.SetTwoFactorEnabledAsync(user, true);

        // Generate recovery codes
        var recoveryCodes = await userManager.GenerateNewTwoFactorRecoveryCodesAsync(user, 10);

        return IdentityResult.Success;
    }

    public async Task<AuthResult> LoginWithTwoFactorAsync(
        string userId, string code)
    {
        var user = await userManager.FindByIdAsync(userId)
            ?? throw new NotFoundException("User", userId);

        var isValid = await userManager.VerifyTwoFactorTokenAsync(
            user,
            userManager.Options.Tokens.AuthenticatorTokenProvider,
            code);

        if (!isValid)
        {
            // Try recovery code
            var redeemResult = await userManager.RedeemTwoFactorRecoveryCodeAsync(user, code);
            if (!redeemResult.Succeeded)
                return AuthResult.Failure("Invalid 2FA code");
        }

        var roles = await userManager.GetRolesAsync(user);
        var accessToken = tokenService.GenerateAccessToken(user, roles);
        var refreshToken = tokenService.GenerateRefreshToken();

        user.RefreshToken = refreshToken;
        user.RefreshTokenExpiryTime = DateTime.UtcNow.AddDays(7);
        await userManager.UpdateAsync(user);

        return AuthResult.Success(accessToken, refreshToken);
    }

    public async Task<IdentityResult> DisableTwoFactorAsync(string userId)
    {
        var user = await userManager.FindByIdAsync(userId)
            ?? throw new NotFoundException("User", userId);

        return await userManager.SetTwoFactorEnabledAsync(user, false);
    }
}

public record TwoFactorSetupDto(string SharedKey, string QrCodeUri);
```

---

## 8. Password Hashing and Policies

```csharp
// Custom password validator
public class CustomPasswordValidator : IPasswordValidator<ApplicationUser>
{
    public Task<IdentityResult> ValidateAsync(
        UserManager<ApplicationUser> manager,
        ApplicationUser user,
        string? password)
    {
        var errors = new List<IdentityError>();

        if (password is null)
        {
            errors.Add(new IdentityError
            {
                Code = "PasswordRequired",
                Description = "Password is required"
            });
            return Task.FromResult(IdentityResult.Failed(errors.ToArray()));
        }

        // No common passwords
        var commonPasswords = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
        {
            "password", "123456", "qwerty", "admin", "letmein"
        };

        if (commonPasswords.Contains(password))
        {
            errors.Add(new IdentityError
            {
                Code = "CommonPassword",
                Description = "This password is too common"
            });
        }

        // Cannot contain username
        if (user.UserName is not null &&
            password.Contains(user.UserName, StringComparison.OrdinalIgnoreCase))
        {
            errors.Add(new IdentityError
            {
                Code = "PasswordContainsUsername",
                Description = "Password cannot contain your username"
            });
        }

        // Cannot contain email prefix
        if (user.Email is not null)
        {
            var emailPrefix = user.Email.Split('@')[0];
            if (password.Contains(emailPrefix, StringComparison.OrdinalIgnoreCase))
            {
                errors.Add(new IdentityError
                {
                    Code = "PasswordContainsEmail",
                    Description = "Password cannot contain your email"
                });
            }
        }

        return Task.FromResult(errors.Count > 0
            ? IdentityResult.Failed(errors.ToArray())
            : IdentityResult.Success);
    }
}

// Register custom validator
builder.Services.AddIdentity<ApplicationUser, ApplicationRole>()
    .AddEntityFrameworkStores<AppDbContext>()
    .AddDefaultTokenProviders()
    .AddPasswordValidator<CustomPasswordValidator>();

// Custom password hasher (if needed)
builder.Services.AddScoped<IPasswordHasher<ApplicationUser>, Argon2PasswordHasher>();
```

---

## 9. Identity API Endpoints (.NET 8+)

```csharp
// .NET 8 provides built-in Identity API endpoints
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddIdentityApiEndpoints<ApplicationUser>(options =>
{
    options.Password.RequiredLength = 8;
    options.SignIn.RequireConfirmedEmail = true;
})
.AddRoles<IdentityRole>()
.AddEntityFrameworkStores<AppDbContext>();

builder.Services.AddAuthorization();

var app = builder.Build();

// Map all identity endpoints: /register, /login, /refresh, /confirmEmail, etc.
app.MapIdentityApi<ApplicationUser>();

// The following endpoints are automatically created:
// POST /register          - Register new user
// POST /login             - Login (returns access + refresh tokens)
// POST /refresh           - Refresh access token
// GET  /confirmEmail      - Confirm email
// POST /resendConfirmationEmail
// POST /forgotPassword
// POST /resetPassword
// POST /manage/2fa        - Two-factor setup
// GET  /manage/info       - Get user info
// POST /manage/info       - Update user info

// Add custom endpoints alongside identity endpoints
app.MapGet("/api/me", async (HttpContext context, UserManager<ApplicationUser> userManager) =>
{
    var user = await userManager.GetUserAsync(context.User);
    if (user is null) return Results.Unauthorized();

    return Results.Ok(new
    {
        user.Id,
        user.Email,
        user.FirstName,
        user.LastName,
        Roles = await userManager.GetRolesAsync(user)
    });
}).RequireAuthorization();

app.Run();
```

---

## 10. Custom User Store

```csharp
// Implement custom stores for non-EF data sources
public class MongoUserStore :
    IUserStore<ApplicationUser>,
    IUserPasswordStore<ApplicationUser>,
    IUserEmailStore<ApplicationUser>,
    IUserRoleStore<ApplicationUser>
{
    private readonly IMongoCollection<ApplicationUser> _users;

    public MongoUserStore(IMongoDatabase database)
    {
        _users = database.GetCollection<ApplicationUser>("users");
    }

    public async Task<IdentityResult> CreateAsync(
        ApplicationUser user, CancellationToken ct)
    {
        await _users.InsertOneAsync(user, cancellationToken: ct);
        return IdentityResult.Success;
    }

    public async Task<ApplicationUser?> FindByIdAsync(
        string userId, CancellationToken ct)
    {
        return await _users.Find(u => u.Id == userId)
            .FirstOrDefaultAsync(ct);
    }

    public async Task<ApplicationUser?> FindByEmailAsync(
        string normalizedEmail, CancellationToken ct)
    {
        return await _users.Find(u => u.NormalizedEmail == normalizedEmail)
            .FirstOrDefaultAsync(ct);
    }

    // ... implement all interface methods

    public void Dispose() { }
}

// Register custom store
builder.Services.AddIdentity<ApplicationUser, ApplicationRole>()
    .AddUserStore<MongoUserStore>()
    .AddRoleStore<MongoRoleStore>()
    .AddDefaultTokenProviders();
```

---

## Best Practices

1. **Always require email confirmation** - Set `options.SignIn.RequireConfirmedEmail = true` to prevent account abuse.
2. **Use strong password policies** - Require minimum 8 characters with complexity rules and add custom validators for common passwords.
3. **Implement account lockout** - Configure `MaxFailedAccessAttempts` and `DefaultLockoutTimeSpan` to prevent brute force.
4. **Store refresh tokens securely** - Hash refresh tokens before storing and set expiration times.
5. **Use claims transformation sparingly** - It runs on every request; cache results if accessing the database.
6. **Separate Identity DbContext** - Consider a dedicated DbContext for Identity tables in large applications.
7. **Implement proper token refresh** - Validate the expired access token and rotate refresh tokens on each use.
8. **Never reveal user existence** - Return the same response for "user not found" and "wrong password" on login.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---|---|---|
| Missing `AddEntityFrameworkStores` | Runtime error: no `IUserStore` registered | Always call `.AddEntityFrameworkStores<TContext>()` |
| Missing `base.OnModelCreating` | Identity tables not created in migration | Always call `base.OnModelCreating(builder)` in DbContext |
| Token in URL without encoding | Tokens with `+` and `/` break URLs | Use `WebEncoders.Base64UrlEncode/Decode` |
| Not seeding roles | `AddToRoleAsync` fails with "Role not found" | Seed roles at startup before assigning |
| `RequireConfirmedEmail` without email service | Users cannot log in after registration | Implement `IEmailSender` or disable for development |
| Returning user existence info | Attackers can enumerate emails | Return generic "invalid credentials" message |
| Not rotating refresh tokens | Stolen refresh token remains valid forever | Issue new refresh token on each refresh and invalidate old one |
| Password reset without rate limiting | Abuse of password reset emails | Add rate limiting to forgot-password endpoint |
