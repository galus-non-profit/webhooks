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
    options.KnownProxies.Add(IPAddress.Parse("your_ip_machine"));
    // TODO move it to some settings
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
```
Microsoft.Extensions.Hosting.WindowsServices
Microsoft.Extensions.Hosting.Systemd
```


- save changes, push code to repository and log in to remote machine. 

## On your remote machine

### 1. Install .NET sdk and runtime
Installing .NET may differ, in ubuntu 20.04 install .net with this [tutorial](https://learn.microsoft.com/en-us/dotnet/core/install/linux-ubuntu-2004)

use following commands:

```bash
wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb

sudo dpkg -i packages-microsoft-prod.deb

rm packages-microsoft-prod.deb

sudo apt-get update && \
  sudo apt-get install -y dotnet-sdk-8.0

sudo apt-get update && \
  sudo apt-get install -y aspnetcore-runtime-8.0
```

- after installation use "dotnet -h" to check if dotnet cli works


### 2. Download source code from git

- make folder to store your source code ex: /home/monday, then clone repository, switch to branch where branch (for this repo branch's name is: bonus-deploy-dotnet-app-as-linux-service)

```bash
mkdir /home/monday
cd /home/monday
git clone https://github.com/galus-non-profit/webhooks.git # or use SSH 
git checkout "bonus-deploy-dotnet-app-as-linux-service"
```

- to check if your app can work in actual machine setup and your code works on linux try tu run app

```bash
# in src project file
dotnet run --project WebApi/Monday.WebApi.csproj --urls "http://localhost:5082"
```
if everything works fine, you should see standard dotnet logs:

```log
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5082
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /home/mondayapi/webhooks/src/WebApi
```

### 3. Publish code as service 

- go to src folder and publish it next to webhook service in /opt directory:

```bash
mkdir /opt/mondayapi
dotnet publish -o /opt/mondayapi/
```

### 4. Configure service 

- in same location with published files create service file named mondayapi.service with following content:

```ini
[Unit]
Description=Monday Service # name it as you want
After=network.target

[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/bin/dotnet /opt/mondayapi/Monday.WebApi.dll # path to dotnet binary and  main project dll
WorkingDirectory=/opt/mondayapi # dir with published files
KillMode=process
Restart=on-failure
Environment=ASPNETCORE_URLS=http://127.0.0.1:5082 # specify port - remember that 5080 and 5090 are occupied by application deployed as docker container and webhook service
# you can specify other env variables that you don't want to pass in settings json file

[Install]
WantedBy=multi-user.target
```
> [!CAUTION]
> remove all comments in file above, comments will corrupt service configuration

- as it was before link service file:

```bash
ln -s /opt/mondayapi/mondayapi.service  /etc/systemd/system/mondayapi.service
```

- if everything is correct you should can run service

```bash
systemctl enable mondayapi.service
systemctl start mondayapi.service
```
to verify service status use command:

```bash
systemctl status mondayapi.service
```
Status of working service looks like this:

![Alt text](/assets/service-status.png)

 and access weather api endpoint on local port 5082, to check it run:

```bash
curl http://127.0.0.1:5082/weatherforecast
```
in response you should see json with weather

- to read service logs use:
```bash
journalctl -u mondayapi -r
```

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