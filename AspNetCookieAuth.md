# Cookie Authentication [ASP.NET]
What is a cookie?

According to Wikipedia: HTTP cookies are small blocks of data created by a web server while a user is browsing a website and placed on the user's computer or other device by the user's web browser. Cookies are placed on the device used to access a website, and more than one cookie may be placed on a user's device during a session.
### Setting up the cookie authentication
Configuring the **Startup** class
```c#
/* Startup.cs */
public void ConfigureServices(IServiceCollection services)
{
    // [Default Configuration] register authentication in the service collection
    // default login path is Account/Login
    services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCookie();


    // [With LoginPath configured] register authentication in the service collection
    services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCookie(option => option.LoginPath = "account/signin");


    // [Custom configuration] register authentication in the service collection
    services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCookie(option => 
            option.ExpireTimeSpan = TimeSpan.FromMinutes(20); // cookie expires in 20 mins
            option.SlidingExpiration = true;
            option.AccessDeniedPath = "/Forbidden/"; // path for access denied
            option.LoginPath = "/login/"; // path for logging in
        );
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    /*
        Adding authentication in the middleware pipeline,
        the order of specifying the middleware is important especially with
        authentication and authorization.

        Place the middleware between useRouting and useEndpoints
    */

    app.UseRouting();

    app.UseAuthentication();  // <----
    app.UseAuthorization();   // <----

    app.UseEndpoints(/* endpoints */);
}
```

### Authorization Filter
Using the Authorize attribute
```c#
/*
    [Authorize] filter ensures that nothing in the controller can be
    accessed if the user is not authenticated
*/
[Authorize]
public class SomeController : Controller
{ /* controller routes/endpoints */ }


/*
    [Authorize] can also be placed on the controller's methods ensuring that nothing in the
    route can be accessed if the user is not authenticated
*/
[Authorize]
public async Task HandleOperation()
{ /* Do some process that requires authenticated users */}

/*
    If you configured the AuthorizeFilter to be global in all controllers, all of the routes/
    endpoints cannot be accessed if the user is not authenticated. This will also include the
    Login/Logout route if you have one in your controller. To allow users without authentication
    in accessing a controllers/endpoints, you can use the [AllowAnonymous]
*/
[AllowAnonymous]
public class AuthController : Controller
{ /* controller routes/endpoints */}
```

### Adding Authorization Globally
If you want all controllers to have [Authorize] filter globablly, you can configure the **Startup** class
```c#
/* Startup.cs */
public void ConfigureServices(IServiceCollection services)
{
    // adding the authorize filter
    // now, every users accessing the controllers needs to be logged in / authenticated
    service.AddControllerWithViews(option => option.Filters.Add(new AuthorizeFilter()));

    // register authentication in the service collection
    services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCookie();
}

```
### Logging Users in and out
Create a controller that handles login/logout routes
```cs
/*
    For demo purpose, a UserAccount class will be used as an object passed from
    the UI/Front-End to the Back-End and stored in a UserRepository or a Database
*/
public class UserAccount
{
    public Guid UserId {get; set;}
    public string Username {get; set;}
    public string Password {get; set;}
    public string FullName {get; set;}
    public string FavoriteColor {get; set;}
}

/*
    AccountController handles the the authentication of the users using the application
*/
public class AccountController : Controller
{
    private readonly IUserRepository userRepository;

    // userRepository is injected in the controller's constructor
    public AccountController(IUserRepository userRepository)
    {
        this.userRepository = userRepository;
    }

    /*
        If an Authorization filter is applied globally in the Startup configure
        services collection, you must add the [AllowAnonymous] attribute to allow
        this endpoint from user's that doesn't have authorization from accessing it.
    */
    [HttpPost]
    [AllowAnonymous]
    public async Task<IActionResult> Login(UserAccount model)
    {
        // when a userAccount is passed, we have to check the repository if the account exists.
        var user = userRepository.GetByUsernameAndPassword(model.Username, model.Password);

        // assume that the repository returns 'null' when no user is found based on the
        // username and password provided. Login cannot proceed, we return Unauthorized [401]
        if(user == null)
            return Unauthorized();

        // if a user is found in the database, we'll create a claims for the user's identity
        // which will be used in the cookie identity
        var userClaims = new List<Claim>
        {
            // nameIdentifier uniquely Identifies the user
            new Claim(ClaimTypes.NameIdentifier, user.UserId.ToString()), 

            // we can specify our own claim types
            new Claim("FullName", user.FullName),
            new Claim(ClaimTypes.Role, "Administrator"),
            new Claim("FavoriteColor", user.FavoriteColor);
        }

        // creating the ClaimsIdentity
        var identity = new ClaimsIdentity(claims, 
            CookieAuthenticationDefaults.AuthenticationScheme); // we pass the default scheme

        // creating the claims principal object with the identity
        var principal = new ClaimsPrincipal(identity);

        // setting the authentication properties
        var authProperties = new AuthenticationProperties
        {
            //AllowRefresh = <bool>,
            // Refreshing the authentication session should be allowed.

            //ExpiresUtc = DateTimeOffset.UtcNow.AddMinutes(10),
            // The time at which the authentication ticket expires. A 
            // value set here overrides the ExpireTimeSpan option of 
            // CookieAuthenticationOptions set with AddCookie.

            //IsPersistent = true,
            // Whether the authentication session is persisted across 
            // multiple requests. When used with cookies, controls
            // whether the cookie's lifetime is absolute (matching the
            // lifetime of the authentication ticket) or session-based.

            //IssuedUtc = <DateTimeOffset>,
            // The time at which the authentication ticket was issued.

            //RedirectUri = <string>
            // The full path or absolute URI to be used as an http 
            // redirect response value.
        };

        // now that we have all the necessary objects for our user's identity,
        // we can now sign in the user
        await HttpContext.SignInAsync(
            CookieAuthenticationDefaults.AuthenticationScheme, // default identity cookie
            principal,
            authProperties
        );

        // we can now redirect the user to a route that requires a login authentication
        return LocalRedirect(/* url */);
    }

    /*
        Logging out the user
    */
    public async Task<IActionResult> Logout()
    {
        // simply call the SightOutAsync method in the HttpContext object to sign out user.
        // this clear's the existing external cookie
        await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);

        // redirect the user to a route [like a homepage '/']
        return Redirect(/* url */)
    }
}
```

### Accessing the Claims Principal
To know who is accessing the requests, we have to track the user's claims principal. [In razor view]
```html
@if(User.Identity.IsAuthenticated)
{
    <!-- Show the user claims -->
    <table>
        @foreach(var claim in User.Claims)
        {
            <tr>
                <td>@claim.Type</td>
                <td>@claim.Value</td>    
            </tr>
        }
    </table>
}
```

### Problem with Identity Cookies
Identity Cookies are fine, but they have a problem especially when you use the **Persistent** cookie option. This cookies potentially remains valid for a long time. User has an access to the application as long as the cookie lives.

For example: Lets say that an employee is authenticated and get's fired, he/she will have access to the application until the cookie expires.

[Solution] React to the event that fires upon each incoming request with a cookie. We can then check the database if the user in a cookie is still valid to access the application. This can be done in the *ConfigureService* at startup class

```c#
services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(
        option => option.Events = new CookieAuthenticationEvents
        {
            OnValidatePrincipal = async (context) => {
                /* we can look at the claim's principal if the user is allowed to access or not */
                var claimsPrincipal = context.Principal;

                /*
                    Do something, check for user validation
                */

                // if the user is not allowed to access the app, then call the RejectPrincipal and signout the user
                context.RejectPrincipal();
                await context.HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
            }
        }
    )
```
