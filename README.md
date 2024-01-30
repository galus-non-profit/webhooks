# Deploy .NET web api as systemd linux service

## On your local machine

### 1. Modify code to allow run it as windows service and linux local service

- in Program.cs just after using statements modify building web application to look like this:

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

- to allow compiler differentiate it is linux or windows we need to modify our web api csproj by adding in main PropertyGroup tag:

```xml
<IsWindows Condition="'$([System.Runtime.InteropServices.RuntimeInformation]::IsOSPlatform($([System.Runtime.InteropServices.OSPlatform]::Windows)))' == 'true'">true</IsWindows>
<IsOSX Condition="'$([System.Runtime.InteropServices.RuntimeInformation]::IsOSPlatform($([System.Runtime.InteropServices.OSPlatform]::OSX)))' == 'true'">true</IsOSX>
<IsLinux Condition="'$([System.Runtime.InteropServices.RuntimeInformation]::IsOSPlatform($([System.Runtime.InteropServices.OSPlatform]::Linux)))' == 'true'">true</IsLinux>
```

- and add three new property groups:

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

- download few missed packages:
```Microsoft.Extensions.Hosting.WindowsServices```

- save changes, push code to repository and log in to remote machine. 

## On your remote machine

### 1. Download source code from git

- make folder to store your source code ex: /home/monday, then clone repository, switch to branch where branch (for this repo branch's name is: bonus-deploy-dotnet-app-as-linux-service)

```bash
mkdir /home/monday
cd /home/monday
git clone https://github.com/galus-non-profit/webhooks.git # or use SSH 
git checkout "bonus-deploy-dotnet-app-as-linux-service"
```

### 2. Publish code as service 

- go to src folder and publish it next to webhook service in /opt directory:

```bash
mkdir /opt/mondayapi
dotnet publish -o /opt/mondayapi/
```

### 3. Configure service 

- in same location with published files create service file named mondayapi.service with following content:

```ini
[Unit]
Description=Monday Service # name it as you want
After=network.target

[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/bin/dotnet /opt/mondayapi/Monday.WebApi.dll # path to main dll
WorkingDirectory=/opt/mondayapi # dir with published files
KillMode=process
Restart=on-failure
Environment=ASPNETCORE_URLS=http://127.0.0.1:5082 # specify port - remember that 5080 and 5090 are occupied by application deployed as docker container and webhook service
# you can specify other env variables that you don't want to pass in settings json file

[Install]
WantedBy=multi-user.target
```
- as it was before link service file:

```bash
ln -s /opt/mondayapi/mondayapi.service  /etc/systemd/system/mondayapi.service
```

- if everything is correct you should can run service

```bash
systemctl start mondayapi.service
```

 and can access weather api endpoint on local port 5082, to check it run:

```bash
curl http://localhost:5082/weatherforecast
```
in response you should see json with weather

### 3. Expose api by reverse proxy

- edit /etc/nginx/sites-available/default file by adding new location:

```nginx
location /mondayapi {
        proxy_pass http://127.0.0.1:5082;
        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr; # ip over header  must
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name;
    }
```

- restart nginx service:

```bash
systemctl restart nginx.service
```
- check if everything works by past api url to browser:

```
https://{server_address}/mondayapi/weatherforecast
```