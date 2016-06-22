
https://msdn.microsoft.com/en-us/library/system.security.claims.claimsprincipal%28v=vs.110%29.aspx

ClaimsPrincipal
-----------------

Beginning with .NET Framework 4.5, Windows Identity Foundation (WIF) and claims-based identity have 
been fully integrated into the .NET Framework. This means that many classes that represent a 
principal in the .NET Framework now derive from *ClaimsPrincipal* rather than simply implementing the 
*IPrincipal interface*.

In addition to implementing the IPrincipal interface, ClaimsPrincipal exposes properties and methods 
that are useful for working with claims. ClaimsPrincipal exposes a collection of identities, each of 
which is a *ClaimsIdentity*.

In the common case, this collection, which is accessed through the Identities property, will only 
have a single element. The introduction of ClaimsPrincipal in .NET 4.5 as the principal from which 
most principal classes derive does not force you to change anything in the way in which you deal 
with identity. It does, however open up more possibilities and offer more chances to exercise 
finer access control. 

ClaimsAuthenticationManager
-----------------------------

### Authentication

For example, you can configure a web-based application or service with a custom claims authentication 
manager, an instance of a class that derives from the *ClaimsAuthenticationManager* class. When so 
configured, the request processing pipeline invokes the Authenticate method on your claims 
authentication manager passing it a ClaimsPrincipal that represents the context of the incoming request. 
Your claims authentication manager can then perform authentication based on the values of the incoming 
claims. It can also filter, transform, or add claims to the incoming claim set. For example, it could be 
used to enrich the incoming claim set with new claims created from a local data source such as a local 
user profile

### Authorization

You can configure a web-based application with a custom claims authorization manager, an instance of a 
class that derives from the ClaimsAuthorizationManager class. When so configured, the request processing 
pipeline packages the incoming ClaimsPrincipal in an AuthorizationContext and invokes the CheckAccess 
method on your claims authorization manager. Your claims authorization manager can then enforce 
authorization based on the incoming claims.

Inline claims-based code access checks can be performed by configuring your application with a custom 
claims authorization manager and using either the ClaimsPrincipalPermission class to perform imperative 
access checks or the ClaimsPrincipalPermissionAttribute to perform declarative access checks. Claims-based 
code access checks are performed inline, outside of the processing pipeline, and so are available to all 
applications as long as a claims authorization manager is configured.

http://benfoster.io/blog/aspnet-identity-stripped-bare-mvc-part-1

Inline claims-based code access checks can be performed by configuring your application with a custom claims 
authorization manager and using either the ClaimsPrincipalPermission class to perform imperative access 
checks or the ClaimsPrincipalPermissionAttribute to perform declarative access checks. Claims-based code 
access checks are performed inline, outside of the processing pipeline, and so are available to all 
applications as long as a claims authorization manager is configured.

In a future post we'll make use of the new ASP.NET Identity UserManager to authenticate against user information 
stored in a database. The intestesting code is what happens when we've validated the credentials.


Setup

1. The UseCookieAuthentication extension tells the ASP.NET Identity framework to use cookie based authentication. We need to set 2 properties:

2. AuthenticationType - This is a string value that identifies the the cookie. This is necessary since we may have several instances of the Cookie middleware. For example, when using external auth servers (OAuth/OpenID) the same cookie middleware is used to pass claims from the external provider. If we'd pulled in the Microsoft.AspNet.Identity NuGet package we could just use the constant DefaultAuthenticationTypes.ApplicationCookie which has the same value - "ApplicationCookie".

3. LoginPath - The path to which the user agent (browser) should be redirected to when your application returns an unauthorized (401) response. This should correspond to your "login" controller. In this case I have an AuthContoller with a LogIn action.

            app.UseCookieAuthentication(new CookieAuthenticationOptions
            {
                AuthenticationType = "ApplicationCookie",
                LoginPath = new PathString("/auth/login")
            });
            

if (model.Email == "admin@admin.com" && model.Password == "password")

Steps

1. First we create a ClaimsIdentity object that contains information (Claims) about the current user. What's particularly interesting about the new Claims based approach is that the Claims are persisted to the client inside the authentication cookie. This means that you can add claims for frequently accessed user information rather than having to load it from a database on every request. This is similar to what we had with Federated Identity's Session Authentication Module (SAM). Note that we also provide the authentication type. This should match the value you provided in the Startup class.

var identity = new ClaimsIdentity(new[] {
                new Claim(ClaimTypes.Name, "Ben"),
                new Claim(ClaimTypes.Email, "a@b.com"),
                new Claim(ClaimTypes.Country, "England")
            },    
            "ApplicationCookie");

2. Next we get obtain an IAuthenticationManager instance from the current OWIN context. This was automatically registered for you during startup.

        var authManager = ctx.Authentication;
        authManager.SignIn(identity);
        
3. Then we call IAuthenticationManager.SignIn passing the claims identity. This sets the authentication cookie on the client.

4. Finally we redirect the user agent to the resource they attempted to access. We also check to ensure the return URL is local to the application to prevent Open Redirection attacks.

Signout

Again we obtain the IAuthenticationManager instance from the OWIN context, this time calling SignOut passing the authentication type (so the manager knows exactly what cookie to remove).

    var ctx = Request.GetOwinContext();
    var authManager = ctx.Authentication;

    authManager.SignOut("ApplicationCookie");
    


Accessing custom claim data

Whilst claims based principals/identities have been used in ASP.NET for a while, the current user is still exposed as IPrincipal/IIdentity this means all we can really get from the User property in our controllers and views is the Name claim.

To access other claims else we can cast the current user identity as a ClaimIdentity. Update the Index action in HomeController to the following:

public ActionResult Index()
{
    var claimsIdentity = User.Identity as ClaimsIdentity;
    ViewBag.Country = claimsIdentity.FindFirst(ClaimTypes.Country).Value;


SMART WAY

The smarter way

Having to do this every time you need access to user claims is a bit of a smell. Wouldn't it be nice if you had strongly typed access to your user claims?

To do this we'll create our own Principal class that wraps the current ClaimsPrincipal and provides strongly typed properties to it's underlying claims:

public class AppUser : ClaimsPrincipal
{
    public AppUser(ClaimsPrincipal principal)
        : base(principal)
    {
    }

    public string Name
    {
        get
        {
            return this.FindFirst(ClaimTypes.Name).Value;
        }
    }

    public string Country
    {
        get
        {
            return this.FindFirst(ClaimTypes.Country).Value;
        }
    }
}
Next we'll add a base controller that provides access to AppUser:

public abstract class AppController : Controller
{       
    public AppUser CurrentUser
    {
        get
        {
            return new AppUser(this.User as ClaimsPrincipal);
        }
    }
}
Finally we can update HomeController like so:

public class HomeController : AppController
{
    public ActionResult Index()
    {
        ViewBag.Country = CurrentUser.Country;
        return View();
    }
}


https://msdn.microsoft.com/en-us/library/system.security.claims.claimsauthenticationmanager(v=vs.110).aspx

# Extending ClaimsAuthenticationManager

    class SimpleClaimsAuthenticatonManager : ClaimsAuthenticationManager
    {
        public override ClaimsPrincipal Authenticate(string resourceName, ClaimsPrincipal incomingPrincipal)
        {
            if (incomingPrincipal != null && incomingPrincipal.Identity.IsAuthenticated == true)
            {
                ((ClaimsIdentity)incomingPrincipal.Identity).AddClaim(new Claim(ClaimTypes.Role, "User"));
            }
            return incomingPrincipal; 
        }
    }
    
    
https://docs.asp.net/en/latest/security/data-protection/configuration/overview.html

Data protection and keys
