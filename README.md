# 1. Webhook deploy

## Required resources and apps

- basic linux machine (used Linux ubuntu 20.04, 1 vcpu, 1 GiB)
- dockerhub account or equivalent in other registry
- docker desktop or equivalent
- windows terminal
- ssh client app (optional), used Bitwise Client 9.33

## On your local machine

### 1.1 Create simple Web Api project

- create project directory and src directory inside. 
- open terminal and navigate to src directory

```sh
dotnet new sln --name Monday
dotnet new webapi --name Monday.WebApi --output WebApi --no-https
dotnet sln add WebApi
```

- adjust weather controller if you want

### 1.2 Prepare containerization

- in src directory add Dokcerfile and docker-compose.yaml files with following content:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY Monday.sln Monday.sln
COPY WebApi/Monday.WebApi.csproj WebApi/Monday.WebApi.csproj
RUN dotnet restore

COPY . .

RUN dotnet build WebApi/Monday.WebApi.csproj --configuration Release --output /app/build

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app

RUN apt-get update && apt-get install -y curl=7.88.1-10+deb12u5 --no-install-recommends && rm -rf /var/lib/apt/lists/*

RUN groupadd -g 10000 dotnet && useradd -u 10000 -g dotnet dotnet && chown -R dotnet:dotnet /app
USER dotnet:dotnet

ENV ASPNETCORE_URLS http://*:5080
EXPOSE 5080

COPY --chown=dotnet:dotnet --from=build /app/build .

HEALTHCHECK --interval=5s --timeout=10s --retries=3 CMD curl --fail http://localhost:5080/health || exit 1

ENTRYPOINT ["dotnet", "Monday.WebApi.dll"]
```
Docker compose file:

```yaml
services:
  monday-webapi:
    build:
      context: .
      dockerfile: Dockerfile
    hostname: webapi
    image: <<your_docker_hub_user_name>>/monday-webapi:latest
    environment:
      ASPNETCORE_ENVIRONMENT: "Development"
    restart: unless-stopped
```

### 1.3 Get registry access token and login to registry (skip if you're logged already) 

- get access token:

In docker.io: 
login to your docker hub and go to settings -> security page, and create access token by click "new access token" button (pass its name and scope: Read-only), then copy token from modal view

```sh
docker login -u <<your_docker_user_name_>> docker.io --password <<token_>> # default registry: docker.io
docker compose build
```
if you want to use other registries you can use:

```bash
docker login registry.example.com -u -u <<your_docker_user_name_>> docker.io --password <<token_>>
```
[more info](https://docs.gitlab.com/ee/user/packages/container_registry/authenticate_with_container_registry.html)

### 1.4 Build image and push it to registry

in src directory enter commands:

```sh
docker compose config
docker compose build
docker compose push
```
## On remote linux machine

### 1.5 Install Nginx
- start as root:

```bash
sudo -i
```
then:
```sh
apt update
apt install nginx
```

- enable nginx (start always)
```sh
systemctl enable nginx
```

### 1.6 Install Docker

- install docker with this [tutorial](https://docs.docker.com/engine/install/ubuntu/) for ubuntu
```sh
apt-get update
apt-get install ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
```

```sh
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```sh
apt-get update
```
- install docker packages

```sh
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

- start and enable docker

```sh
systemctl start docker
systemctl enable docker
```
### 1.7 Create docker compose file and redeploy script and run application

- create your working directory where you will store files (in this tutorial was: /home/webhooks)
- create local docker-compose.yaml file:

```yaml
services:
  app:
    image: <<your_docker_hub_user_name>>/monday-webapi:latest # see docker compose from local machine
    ports:
      - 127.0.0.1:5080:5080/tcp
    restart: unless-stopped
    # docker login, nazwa registry, + hasło z pliku 
```
- in same directory create file redeploy.sh with following content:

```bash
#!/bin/sh
docker login -u <<your_docker_user_name_>> docker.io --password <<token_>>
docker compose down
docker compose pull
docker compose up -d
```

change redeploy's owner

```bash
chmod +x redeploy.sh
```

- login to docker, pull image and up:

```bash
docker login -u <<your_docker_user_name_>> docker.io --password <<token_>>
docker compose up -d
```
check if app is running and is available on port 5080 (specified in Dockerfile)

```bash
curl http://localhost:5080/weatherforecast
```
should return json with weather.

### 1.8 Download and unzip webhook app

- download from https://github.com/adnanh/webhook/releases (for ubuntu 20.04 it will be webhook-linux-amd64.tar.gz) and copy to your machine by ftp or:

use curl:
```bash
curl -L -o webhook-linux-amd64.tar.gz https://github.com/adnanh/webhook/releases/download/2.8.1/webhook-linux-amd64.tar.gz
```

or use wget:
```bash
wget -c  https://github.com/adnanh/webhook/releases/download/2.8.1/webhook-linux-amd64.tar.gz
```

unpack zipped file, change its owner, and move to service directory (/opt/webhook)

```bash
mkdir /opt/webhook
tar --strip-components=1 -xvzf webhook-linux-amd64.tar.gz -C /opt/webhook
chown root:root /opt/webhook/webhook
# suid, sticky bit
```
[read more about chmod and chown] (https://askubuntu.com/questions/443789/what-does-chmod-x-filename-do-and-how-do-i-use-it
https://www.redhat.com/sysadmin/suid-sgid-sticky-bit)

### 1.9 Configure webhook

Follow webhook [documentation](https://github.com/adnanh/webhook) and create hooks.json file in /opt/webhook directory 

```json
[
  {
    "id": "monday",
    "execute-command": "/home/webhook/redeploy.sh",
    "command-working-directory": "/home/webhook",
    "response-message": "OK"
  }
]
```
check [json example](https://github.com/adnanh/webhook/blob/master/hooks.json.example)
- in same directory (opt/webhook) create service unit file:

```ini
[Unit]
Description=Webhook Service
After=network.target

[Service]
Type=simple
User=root
Group=root
ExecStart=/opt/webhook/webhook -hooks=/opt/webhook/hooks.json -hotreload=false -ip=127.0.0.1 -port=5090 -secure=false -verbose=true -debug=false
# exec start: binary located in /opt/webhook/webhook file will actions from hooks.json file: execute command from /home/webhook/redeploy when receive request on 5090 port
WorkingDirectory=/opt/webhook
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

create link between created webhook.service and systemd file:

```bash
ln -s /opt/webhook/webhook.service  /etc/systemd/system/webhook.service
```

### 1.10 Run webhook service

```bash
systemctl start webhook.service
# then you can check it status by:
systemctl status webhook.service
```
To read recent logs execute command:

```bash
journalctl -u whitebear -r
```

At this point you can execute webhook locally on port 5090.
Before execute, go back to your local machine, change weather response, build image and push it to registry.

Wait a while, a moment and call hooks local endpoint

```bash
curl http://127.0.0.1:5090/hooks
```
After hit /hooks endpoint weather should change. 

At this point both web api and webhook are available locally, we need to expose it by reverse proxy


### 1.11 Configure nginx
9. Configure nginx

go to /etc/nginx/sites-available and edit file named "default" its content should looks like below (expose local container with web api on port 5080 and installed webhook on port 5090):

```nginx
server {
    server_name default_server;
    listen 80;

    location /hooks {
        proxy_pass http://127.0.0.1:5090/hooks;
    }

    location / {
        proxy_pass http://127.0.0.1:5080;
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
}
```

then link default file in sites available to enabled:

```bash
ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
```

restart nginx 
```bash
systemctl restart nginx
```

Then check if webhook works. In your local machine paste address:

```bash
remote_machine_ip:5090/hooks/
```
Response should be string "OK" and redeploy script should run on remote machine. 


### 1.12 Add webhook to dockerhub

Go to dockerhub, click on monday image and select webhooks tab:

![Alt text](/assets/webhooks.png)

then create webhook by adding its url and name

After pushing new image version, dockerhub will send request and run redeploy script.


# 2. Deploy .NET web api as systemd linux service

## On your local machine

### 2.1 Modify code to allow run it as windows service and linux local service

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

### 2.2 Install .NET sdk and runtime
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


### 2.3 Download source code from git

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

### 2.3 Publish code as service 

- go to src folder and publish it next to webhook service in /opt directory:

```bash
mkdir /opt/mondayapi
dotnet publish -o /opt/mondayapi/
```

### 2.5 Configure service 

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

### 2.6 Expose api by reverse proxy

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

#  3. Deploy .NET web api as windows service

We start as exactly same code as in previous branch (bonus-deploy-dotnet-app-as-linux-service)

### 3.1 Publish app

- in your project folder clone monday api project

```bash
git clone https://github.com/galus-non-profit/webhooks.git # or use SSH 
git checkout "bonus-deploy-dotnet-app-as-linux-service"
```

create folder for published code for ex in powershell::

```bash
mkdir C:\Program Files\monday\mondayapi
```
then publish code from src folder (or specify path to monday project):

```bash
dotnet publish -o 'C:\Program Files\monday\mondayapi\'
```
Adjust appsettings.json by adding default port for app:

```json
"Kestrel": {
    "Endpoints": {
      "MyHttpEndpoint": {
        "Url": "http://localhost:5081"
      }
    }
  }
```

### 3.2 Create and run service

go to published code folder 
```bash
cd C:\Program Files\monday\mondayapi
```

run command to create windows service

``` bash
New-Service -Name "MondayApi" -BinaryPathName .\Monday.WebApi.exe # check if WebApi.exe is in specified location
```

sometimes it returns error, so try to pass absolute path:

```bash
New-Service -Name "MondayApi" -BinaryPathName C:\Program Files\monday\mondayapi\Monday.WebApi.exe
```

### 3.3 Run service and change user account

launch service by service window:


![Alt text](/assets/windows_services.png)

or use command:

```bash
sc.exe start MondayApi  # MondayApi is service name specified in creating service name
```

to stop, or delete use sc.exe stop
If you have not installed sc.exe tool. 

sc.exe delete equivalent of command:

```bash
Remove-Service -Name "MondayApi"
```

To change user account go, to windows service panel, right click on MondayApi service and choose "properties tab", then select  "Login" tab to specify user credential. Use local user (not system). This is important for security reasons so that the api cannot use elevated privileges.


![Alt text](/assets/user.png)

After hit https://localhost:5081/ you should see swagger page:

![Alt text](/assets/swagger.png)