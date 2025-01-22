# WatchNest Main Application
This Project contains the main application. To use this, you will need to download this project and the API for this project (which is named WatchNest - API). The explanation for how each action in the API work is explained in the README for the API repository. This README will focus on the front-end and back-end of this application.

# Table of Contents
- [Overview](#overview)
- [Packages Used](#packages-used)
- [Program.cs](#programcs)
  - [Distributed SQL Caching](#distributed-sql-caching)
  - [Cookie Authentication](#cookie-authentication)
  - [HTTP Configuration](#http-configuration)
  - [Middleware: JwtClaimsMiddleware](#middleware-jwtclaimsmiddleware)
- [Models](#models)
  - [Account Services](#account-services)
  - [Admin Services](#admin-services)
  - [WatchList Service](#watchlist-service)
- [Views](#views)
  - [Filter.cshtml](#filtercshtml)
- [Controllers](#controllers)
  - [Admin](#admin)
  - [WatchList](#watchlist)
  - [Home](#home)
- [Ajax script](#ajax-script)
- [SASS](#SASS)
- [Conclusion](#conclusion)


# Overview
This project is developed in .NET 6 and its main focus is to provider frendly user experience in storing user's watchlist, such as movies, series, and videos that they have seen and search for their series in their own list when they want to review certain titles, genre, etc. 

There are two different host that can be used in this application,

`https://localhost:44301`

`http://localhost:5001`

when using the http host, it will redirect to https host.

**Note:** If you are using this project for the first time, or locally, make sure to inject the seed that is available in the api when logged in as Administrator or disable the authorization for admin-only in case there is no admin credentials available.

Once seed has been implemented, you have two dummy access for testing,

`Username`: TestUser

`Password`: MyVeryOwnTestPassword123$

`Username`: TestAdministrator

`Password`: MyVeryOwnTestPassword123$

# Packages Used
If you need to have the packages installed to run this project, please ensure the following packages are installed when using this project:

- Microsoft.Web.LibraryManager.Cli 2.1.175
- System.IdentityModel.Tokens.Jwt 8.2.0
- Caching SqlServer 8.0.6
- Google fonts
- Bootstrap 5.1.3(through libman)
- Jquery-validate 1.19.3 (through libman)
- Jquery-validation-unobtrusive  3.2.12 (through libman)
- Jquery 3.6.0 (through libman)

All the commands that proceeds this line must be in the project directory in command line.

For Libman, ensure you have it installed in your system. If not, you can use the following command line:

`dotnet tool install --global Microsoft.Web.LibraryManager.Cli --version 2.1.175`

then,

`libman init -p cdnjs`

The command above initializes libman, which is the library manager, and sets cdnjs (content delivery network for javascript) as a default provider for client side libray.

Afterwords, install Bootstrap using the following command:

`libman install bootstrap@5.1.3 -d wwwroot/lib/bootstrap`

Install JQuery packages:

`libman install jquery@3.6.0 -d wwwroot/lib/jquery`

Install validation packages for client side
`libman install jquery-validate@1.19.3 -d wwwroot/lib/jquery-validate`

`libman install jquery-validation-unobtrusive@3.2.12 -d wwwroot/lib/jquery-validation-unobtrusive`


# Program.cs
The following describes a little bit about what has been implemented

### Distributed SQL Caching
I have set up caching to be store in SQL database. Table is name as `AppCache`. This will store results produced by the API calls to improve performance and reduce the number of redundant API calls

If you need to create the table for caching, run the following command in command line:

`dotnet sql-cache create "{connectionString}" dbo AppCache`

The {connectionString} needs to be replace with your actual connection string. If you want to use the local db in Visual studio, you can use the local connection that is used in appsettings.json (named localDb). 
Also be sure to replace any double backslash (such as in localhost\\Myproject) with a single backslash (localhost\MyProject); otherwise, the CLI command will fail. 

Make sure to replace {conncetionString} and keep the double quotation ( " " ) in the command line


### Cookie Authentication
I have implemented cookie authentication, which manages user's session. Its scheme consist of: 
A cookie that will hold the cookie produce by the API. The API will return a cookie that holds JWT. More info on my README in WatchNest - API.

A Login Path that if unaunthenticated user attempts to access an endpoint or resource, they will be redirected to the login page.

A Logout Path when a user logs out, they are redirected back to the login page.

Access Denied Path: If a user tries to access a resource they are not authorized to view, they are directed to a Access Denied Page.

```csharp
builder.Services.AddAuthentication("CookieAuth").AddCookie("CookieAuth",opts =>
{
    opts.Cookie.Name = "AuthToken";
    opts.LoginPath = "/Home/Index";
    opts.LogoutPath = "/Home/Logout";
    opts.AccessDeniedPath = "/Home/AccessDenied";
});

```

### HTTP Configuration
This application uses client named "APIClient" for making API calls. It contains the following:

`Base Address`: The base address for the HTTP client is set to `https://localhost:44350/`. This will be the default address used for all outgoing requests from this client.

`Accept`: This header is set to application/json, ensuring the client expects JSON responses from the API. 

`Lifetime handler`: The handler lifetime is set to 30 minutes. This means the HttpClient handler will be reused for 30 minutes before being disposed and replaced. This improves performance by reducing the overhead of creating new handlers for each request.

```csharp
builder.Services.AddHttpClient("APIClient",client =>
{
    client.BaseAddress = new Uri("https://localhost:44350/");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
}).SetHandlerLifetime(TimeSpan.FromMinutes(30));
```

### Middleware: JwtClaimsMiddleware 
This Middleware uses a class to extract information from JWT stored in the cookie and attach the claims to the current `HttpContext.User`. 

Here is the General Process:

1. JWT Retrieval: Retrieves the JWT token stored in a cookie named `AuthToken`
2. JWT Decoding: Uses JwtSecurityTokenHandler to decode the token. Reads the claims from the token to extract information.
3.  Claims Handling: Creates a ClaimsIdentity using the claims from the decoded JWT. This way, it assigns the identity to HttpContext.User, enabling claims-based authentication throughout the application.

Once the process is done, it moves to the next middleware in the pipeline.  
```csharp
public async Task InvokeAsync(HttpContext context)
{
    var jwt = context.Request.Cookies["AuthToken"];

    if (!string.IsNullOrEmpty(jwt))
    {
        try
        {
            //Decode
            var handler = new JwtSecurityTokenHandler();
            var token = handler.ReadJwtToken(jwt);

            // ClaimsIdentity from the token claims
            var claims = new ClaimsIdentity(token.Claims, "Bearer");

            // Attach claims to the HttpContext.User
            context.User = new ClaimsPrincipal(claims);
        }
        catch
        {
            _logger.LogError("There was an error with jwt Token. Please look into it!");
            context.User = new ClaimsPrincipal();
        }
    }
    await _next(context);
}
```
## Models
Most Model classes are pretty straight forward, so I will be explaining the implementations for Account, Admin, and WatchList Services. These classes maintain Separation of Concerns by keeping business logic encapsulated within the service while letting the controllers focus on handling HTTP request.

### Account Services
This class inherits `IAccountService` interface and handles account related operations such as login, logout, and registration. It uses `IHttpClientFactory` to send and recieve request to and from the API. 

**Methods**

`LoginAsync`: Send a POST request to Account/Login endpoint with loginDTO object which contains user's username and password. Once Successful, it recieves a cookie containing JWT and is used for authentication in this application. This method returns a tuple, a bool flag, error message, and JWT cookie. 

`LogoutAsync`: Sends a DELETE request to Account/Logout endpoint and it returns a bool whether the opperations was a success.

`RegisterAsync`: Sends a POST request to the Account/Register endpoint with object RegisterDTO that contains user details. It return a tuple that contains a bool flag and any error messages.

`ExtractErrorMessageAsync`: A Helper method that parses error response from the API, and attempts to deserialize the response into an `ErrorDetails` object.
```csharp
public async Task<(bool IsSuccess, string? ErrorMessage, string? JwtCookie)> LoginAsync(string username, string password);

public async Task<(bool IsSuccess, string? ErrorMessage)> RegisterAsync(string username, string password, string email);

public async Task<bool> LogoutAsync();
```
### Admin Services
This is reponsivle for implementing administrative actions by interacting with API endpoints, similiar to the previous method. .

**NOTE** ApiResponse<T> is described in later section.

**Methods**
`GetAllUsersAsync`: Fetches all users in the database with a GET request to the API and returns an ApiResponse<T> and its type is tied to UserModel.

`GetAllSeriesAsync`: Fetches all series from the database with a GET request to the API and returns ApiResponse<T> and its type is SeriesDTO.

`DeleteUserAsync`: Deletes user by their Id with a DELETE request to the API and return a bool determining if it was sucessful or not. 

`GetCachedUserListAsync`: Tries to retrieved cached user list, if it exist. If it is empty, expired, or null, it cahces a new user list from the API call and set the expiration to 1 hour and 30 min. 

`RefreshCacheAsync`: Invalidates cache based on cache key.
```csharp
public async Task<ApiResponse<UserModel>> GetAllUsersAsync();
public async Task<ApiResponse<SeriesDTO>> GetAllSeriesAsync();
public async Task<ApiResponse<SeriesDTO>> GetFilteredSeriesAsync(FilterDTO filterDTO);
public async Task<ApiResponse<UserModel>> GetCachedUserListAsync(string cacheKey);
public async Task RefreshCacheAsync(string cacheKey);
public async Task<bool> DeleteUserAsync(string userID);
```
### WatchList Service
This class provides management for dealing with users watchlist with CRUD operations, filtering and caching.

**Methods**

`GetSeriesAsync`: retrieves series for specific user and includes a refresh for caching. The api results gets cached for 3 min and if there is no series, then it will return empty ApiResponse<SeriesDTO>. 

`FilterSeriesAsync`: retreives filter series based on FilterDTO which contains sort order, sort column, filter query, and UserID. It returns an ApiResponse<SeriesDTO> object.

`GetSeriesByIdAsync`: Fetches details of a single series by its series ID by calling a GET request. and returns a UpdatedModel for the method.

`AddSeriesAsync`: Adds a new series using a POST request and returns a bool that represents success. 

`DeleteSeriesAsync`: Deletes a series by its ID via a DELETE request and returns a boolean indicating the operation's success.

`UpdateSeriesAsync`: Updates an existing series using a PUT request and returns a bool determining the success of the API call.

## Views
The Views folder contains four different folders, three of them are respective to controllers names and action while one of them is a shared folder that contains partial views, two different layouts, error page, and NotFound page.

I will describe one of them as most, if not all, are the same or similar structure or pattern.

### Filter.cshtml
This view is within WatchList folder that uses `_AfterLoginLayout` Layout.
```cshtml
@{
    Layout = "_AfterLoginLayout";
}

@model (IEnumerable<SeriesDTO>,FilterModel)

<div class="container-fluid p-0 pt-2 mt-5">
    <div class="row">
        <!-- Filter Column -->
        <div class="col-md-3 text-white">
            <div class="d-grid gap-1">
                <h3 class="text-center">Filter by</h3>
                <form method="get" asp-action="Filter" asp-controller="WatchList">
                    <div class="mb-2">
                        <label for="FilterQuery">Search </label>
                        <input type="text" id="filterQuery" name="FilterQuery" class="form-control" placeholder="Enter query" value="@Model.Item2.FilterQuery" />
                    </div>
                    <div class="mb-2">
                        <label for="sortColumn">By</label>
                        <select id="sortColumn" name="SortColumn" class="form-select">
                            <option value="TitleWatched" selected="@((Model.Item2.SortColumn == "TitleWatched") ? true : false)">Title</option>
                            <option value="Provider" selected="@((Model.Item2.SortColumn == "Provider") ? true : false)">Provider</option>
                            <option value="Genre" selected="@((Model.Item2.SortColumn == "Genre") ? true : false)">Genre</option>
                        </select>
                    </div>
                    <div class="mb-2">
                        <label for="sortOrder">Sort by:</label>
                        <select id="sortOrder" name="SortOrder" class="form-select">
                            <option value="ASC" selected="@((Model.Item2.SortOrder == "ASC") ? true : false)">Ascending</option>
                            <option value="DESC" selected="@((Model.Item2.SortOrder == "DESC") ? true : false)">Descending</option>
                        </select>
                    </div>
                    <div class="mb-2">
                        <button type="submit" class="btn btn-primary">Apply Filters</button>

                    </div>
                </form>
            </div>
        </div>

        <!-- Table Column -->
        <div class="col-md-9">
            <div id="table-content" data-viewtype="User" data-datatype="SeriesDTO">
                @await Html.PartialAsync("_SeriesTablePartial", Model.Item1)
            </div>
            <!--Navigation-->
            <div id="pagination-container" data-viewtype="User" data-datatype="SeriesDTO">
                @if (ViewBag.Pagination != null)
                {
                    var pagination = ViewBag.Pagination as ApiResponse<SeriesDTO>;
                    @await Html.PartialAsync("_PaginationPartial", pagination);
                }
            </div>
        </div>
    </div>
</div>
```
**Filter Section**: This section allows users to filter series by query, column, and sort order. It is also bound to Filter action in WatchList Controller via `asp-action`and `asp-controller` 

**Table Column**: This section renders series data from IEnumerable<SeriesDTO>. It is a partial view since other views might use this table and reduce redundant code.

**Pagination**: Renders and displays pagination based on the data provided from ViewBag.Pagination. This is also a partial view since other views might also use this functionality for their display.

Both, Table Column and Pagination, uses AJAX to update info without having to reload full page. 

## Controllers
There are three controller and will briefly describe them.


### Admin
This controller, as stated in its name, is for admin access only. It secures this controller by using Role-Based Access Control using `[Authorize(Roles = RoleNames.Administrator)]`. The purpose of this controller is for admin to manage users, series, and filter and sort series. 

It also uses partial views updates for UI updates without reloading the page by using AJAX, which will be described in later section.

As mentioned before, I maintain Separation of Concern by differentiate between business logic and HTTP handling. so most of the work are in the implementation within  _adminService

It also dependency injection for the following 
```csharp
 private readonly ILogger<AdminController> _logger;
 private readonly IAdminService _adminService;

 public AdminController(ILogger<AdminController> logger, IAdminService adminService)
     => ( _logger, _adminService) = (logger, adminService);
```
### WatchList
This controller is for authorized Users and Admin to use. Therefore unauthorized users need to login to access this resource. The purpose of this controller is for users to use CRUD operation for their own series watchlist. This controller also uses partial views to update its UI without reloading the page. 

its dependency injection is the following:
```csharp
private readonly ILogger<WatchListController> _logger;
private readonly IWatchListService _watchListService;

public WatchListController(ILogger<WatchListController> logger, IWatchListService watchListService)
    => (_logger, _watchListService) = (logger, watchListService);
```

### Home
The Home Controller manages Account related actions such as login, logout, and registration. This also authenticate users if they provide correct credentials and register new users if needed. Once logged in and authenticated, the action method passes to another method that navigate the user to proper page, based on users rolse, User or Admin. 

This controller also provides Access Denied and error page when the user attempts to access unauthorized, URL does not exist, and when there is a bug in the action that the user attempted to do. 

Here are the Dependency, 
```csharp
 private readonly ILogger<HomeController> _logger;
 private readonly IAccountService _accountService;

 public HomeController(ILogger<HomeController> logger, IAccountService accountService) =>
     (_logger,_accountService) = (logger, accountService);
```
## Ajax script
The AJAX script in JS folder enables dynamic pagination for tables. It updated tables and pagination based on the paginated response from the API and the type of data that is being used and displayed. As mentioned previously, this allows the table to be updated without having to do a full reload page to access paginated data.

To start off,
```Javascript
$(document).on('click', '.pagination-link', function (e) {
    e.preventDefault(); // Prevent the default link behavior

    const url = $(this).data('url'); // Get the URL from the data attribute
    const viewType = $('#table-content').data('viewtype'); // Determine if it's Admin or User
    const dataType = $('#table-content').data('datatype'); // Determine data type

    if (!url || !viewType || !dataType)
        return; // Do nothing if URL or view type is not set

    const tableContentId = '#table-content';
    const paginationContainerId = '#pagination-container';

    const urlMapping = {
        Admin: {
            UserModel: { tableUpdateUrl: '/Admin/PartialUserTable', paginationUpdateUrl: '/Admin/PaginationUpdate' },
            SeriesDTO: { tableUpdateUrl: '/Admin/PartialTableUpdate', paginationUpdateUrl: '/Admin/PaginationUpdate' },

        },
        User: {
            SeriesDTO: { tableUpdateUrl: '/WatchList/PartialTableUpdate', paginationUpdateUrl: '/WatchList/PaginationUpdate' },
        },
    };
 const updateUrls = urlMapping[viewType]?.[dataType];
 if (!updateUrls)
     return; // Do nothing for invalid viewType or dataType
    const { tableUpdateUrl, paginationUpdateUrl } = updateUrls;

```
This code attaches an event listen to elements that are using `.pagination-link` attribute while disabling link behavior.

Next it retrives url and data needed for the script to handle and direct which table and navigation needs updating. It aslo uses predefined `urlMapping` object to map`viewType` and `dataType` combinations to their respective endpoints.

```javascript
const { tableUpdateUrl, paginationUpdateUrl } = updateUrls;
$.ajax({
    url: url,
    type: 'GET',
    headers: { 'X-Requested-With': 'XMLHttpRequest' },
    xhrFields: { withCredentials: true },
    success: function (result) {
        const apiResponse = {
            data: result.data,
            pageIndex: result.pageIndex,
            recordCount: result.recordCount,
            totalPages: result.totalPages,
            links: result.links,
        };

        // Update Partial Table
        updatePartial(tableUpdateUrl, apiResponse.data, tableContentId);

        // Update Pagination
        updatePartial(paginationUpdateUrl, apiResponse, paginationContainerId);
    },
    error: function (status, error) {
        console.error("Pagination error:", status, error);
        alert("An error occurred while loading the page.");
    },
});

function updatePartial(url, data, containerId) {
    $.ajax({
        url: url,
        type: 'POST',
        contentType: 'application/json',
        data: JSON.stringify(data),
        success: function (partialHtml) {
            $(containerId).html(partialHtml);
        },
        error: function (status, error) {
            console.error(`Error updating content for ${containerId}:`, status, error);
            alert(`An error occurred while updating the content for ${containerId}.`);
        },
    });
}
```
it send a GET request to the retrived paginated data based on the URL that it recieved. After a successful retrieval, it updates the table and paginated navigation based on specific attributed, `#table-content`, `#pagination-container`. 

## SASS
Before going into depth for how I used SASS, I'll explain quickly how I created sass and how to convert sass to css to use in views files.

Start off by creating a sass by adding a file with an extension `.scss` in your wwwroot folder, or wherever you keep your design files. 

Afterwords, in the command line already in project directory, use the following command:

`sass styles.scss styles.css`

* Replace the `styles` with the name of your file.
 
This command creates a CSS file that Views files can use to render custom properties. Whenever you update the SASS file, you need to update the css file by running the command above again.

You can also use the following command to automatically recompile the changes made from SASS file,

`sass --watch styles.scss:styles.css`

* Replace the `styles` with the name of your file.

Now moving onto how and why I used SASS


For this front-end part of the project, I have used SASS for a couple of reasons.In my SASS file, I used variables such as 
```scss
$cardColor: #272524;
$gradient: linear-gradient(45deg, #ff7c7d, #ffda67);
$highlightColor: #ffda67;
```
which I can reuse them for other parts that I may use. Also if I need to change a shared color across different properties, I can do so easily by changing one variable. 

I also usedd SASS for nesting and reducing redundancy, keeping them easy to read and maintain. For example, in my file,
```scss
li {
    color: #fff;
    position: relative;

    &::after {
        content: " ";
        position: absolute;
        bottom: 0;
        left: 0;
        width: 100%;
        height: 2px;
        background: #fff;
        border-radius: 5px;
        transform: scaleX(0);
        transition: all 0.3s ease;
    }

    &:hover::after {
        transform: scale(1);
    }

    &.center-text {
        flex-grow: 1; /* Allow item to take extra space */
        text-align: center;
    }
}
```
Lastly, I enhance my website more by adding animation and special effects to it such as adding animation:

```scss
@keyframes rotate {
    0% {
        background-position: 0% 50%;
    }

    100% {
        background-position: 100% 50%;
    }
}
```

adding glass effect into login cards:

```scss
.login-card {
    /* Glass-like effect */
    position: relative;
    z-index: 3;
    width: 100%;
    margin: 0 20px;
    padding: 70px 30px 44px;
    border-radius: 1.25rem;
    background: rgba(148, 170, 51, 0.7); /* Semi-transparent*/
    box-shadow: 0 4px 30px rgba(0, 0, 0, 0.1); /* Subtle shadow */
    text-align: center;
    border: 1px solid rgba(33, 108, 231, 0.3);
}
```
lastly, adding hover states to add a small effect to navigation buttons when the user hover, or selecting items from it:
```scss
.navbar {
// omitted for clarity 

    li {
        color: #fff;
        position: relative;

        &::after {
            content: " ";
            position: absolute;
            bottom: 0;
            left: 0;
            width: 100%;
            height: 2px;
            background: #fff;
            border-radius: 5px;
            transform: scaleX(0);
            transition: all 0.3s ease;
        }

        &:hover::after {
            transform: scale(1);
        }
// omitted for clarity 
    }
}
```

There are more examples in both of my SASS files in this project, so please take a look at it and make some changes if you want to experiment with SASS. 

## Conclusion
This is the final version for the main application for the project WatchNest. As mentioned at the beginning of this README, you need to download or install the necessary package and ensure you have wwwroot as a folder in your environment. Since this project is the Main Application, you also need to download the API for this application to work as it needs to communicate to SQL server to CRUD user's series.

There are improvements that can be made, such as admin having more actions to manage users info, Users having the ability to add a description to each series, etc, however, I will be focusing more on other project in the future such as using Angular and ASP.NET Core. 
