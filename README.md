# Deploy .NET web api as windows service

We start as exactly same code as in previous branch (bonus-deploy-dotnet-app-as-linux-service)

### 1. Publish app

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

### 2. Create and run service

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

### 3. Run service and change user account

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