# Deploy .NET weba api as systemd linux service

## On your local machine

### 1. Modify code to allow run it as windows service and linux local service

In Program.cs just after using statements modify building web application to look like this:

```csharp
// using statements

var builder = WebApplication.CreateBuilder(args);
builder.Configuration.AddEnvironmentVariables();

#if Linux
builder.Host.UseSystemd();

builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.KnownProxies.Add(IPAddress.Parse("20.234.69.131"));
});

#endif
#if Windows
builder.Host.UseWindowsService(options =>
{
    options.ServiceName = ServiceConstants.ServiceName;
});
#endif

builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.KnownProxies.Add(IPAddress.Parse("your_machine_ip"));
});

// rest of registration services, build app and add middlewares
```

to allow compiler differentiate it is linux or windows we need to modify our web api csproj by adding in main PropertyGroup tag:

```xml
<IsWindows Condition="'$([System.Runtime.InteropServices.RuntimeInformation]::IsOSPlatform($([System.Runtime.InteropServices.OSPlatform]::Windows)))' == 'true'">true</IsWindows>
<IsOSX Condition="'$([System.Runtime.InteropServices.RuntimeInformation]::IsOSPlatform($([System.Runtime.InteropServices.OSPlatform]::OSX)))' == 'true'">true</IsOSX>
<IsLinux Condition="'$([System.Runtime.InteropServices.RuntimeInformation]::IsOSPlatform($([System.Runtime.InteropServices.OSPlatform]::Linux)))' == 'true'">true</IsLinux>
```

and add three new property groups:

```xml
<PropertyGroup Condition="'$(IsWindows)'=='true'">
    <DefineConstants>Windows</DefineConstants>
</PropertyGroup>
<PropertyGroup Condition="'$(IsOSX)'=='true'">
    <DefineConstants>OSX</DefineConstants>
</PropertyGroup>
<PropertyGroup Condition="'$(IsLinux)'=='true'">
    <DefineConstants>Linux</DefineConstants>
</PropertyGroup>
```

Download few missed packages:
```Microsoft.Extensions.Hosting.WindowsServices```

Save changes, push code to repository and log in to remote machine. 

## On your remote machine
