# Demo ASP.NET Core app binding to MySql

## Pre Demo Setup

Before the demo make sure all VMs are provisioned and run all of the services at least once and create the database. This will get most of the timeconsuming network/build stuff done up front and the actual demo will be faster.

Run each service in a separate powershell console to make it easy to project all supervisors simultaneously.

During the demo we will upload an update and change config in a separate local console.

### Provision VMs

```
vagrant up # respond to all prompts
```

### Start mysql
```
$c = Get-Credential vagrant # vagrant is the password when prompted
Enter-PSSession -VMName hab1 -Credential $c
hab start core/mysql --group dev --password hab
```

### Create Database

```
$c = Get-Credential vagrant # vagrant is the password when prompted
Enter-PSSession -VMName hab1 -Credential $c
cd c:\habitat-aspnet-sample
hab pkg install core/dotnet-core-sdk
hab pkg exec core/dotnet-core-sdk dotnet restore
hab pkg exec core/dotnet-core-sdk dotnet ef database update
```

### Start first aspnet service

```
hab sup start core/habitat-aspnet-sample --group dev --bind database:mysql.dev --strategy rolling --topology leader --password hab
```

### Start other aspnet services

Hab2:
```
$c = Get-Credential vagrant # vagrant is the password when prompted
Enter-PSSession -VMName hab2 -Credential $c
hab start core/habitat-aspnet-sample --group dev --bind database:mysql.dev --peer 192.168.137.6:9638 --strategy rolling --topology leader --password hab
```

Hab3:
```
$c = Get-Credential vagrant # vagrant is the password when prompted
Enter-PSSession -VMName hab3 -Credential $c
hab start core/habitat-aspnet-sample --group dev --bind database:mysql.dev --peer 192.168.137.6:9638 --strategy rolling --topology leader --password hab
```

### start haproxy

```
vagrant ssh haproxy
sudo HAB_HAPROXY="$(cat /vagrant/habitat/ha.toml)" hab start core/haproxy  --group dev --bind backend:habitat-aspnet-sample.dev --peer 192.168.137.81:9638
```

### Shutdown all supervisors

```
$c = Get-Credential vagrant # vagrant is the password when prompted
Invoke-Command -VMName @('hab1','hab2','hab3','haproxy') -Credential $c -Command { Taskkill /IM hab-sup.exe /T /F }
```

Finally run `cls` in all windows to clear the display.

## Demo

### Start mysql

```
hab sup start core/mysql --group dev --password hab
```

### Start `hab1`'s aspnet service

```
hab sup start core/habitat-aspnet-sample --group dev --bind database:mysql.dev --strategy rolling --topology leader --password hab
```

### Start the other aspnet services

```
hab sup start core/habitat-aspnet-sample --group dev --bind database:mysql.dev --peer 192.168.137.6:9638 --strategy rolling --topology leader --password hab
```

### Start haproxy

```
sudo HAB_HAPROXY="$(cat /vagrant/habitat/ha.toml)" hab start core/haproxy  --group dev --bind backend:habitat-aspnet-sample.dev --peer 192.168.137.6:9638
```

### Update important message config

```
"important_message = 'all is fine'" | hab config apply --peer 192.168.137.6 habitat-aspnet-sample.dev 1
```

### Update sample app

```
hab studio build . -w
hab pkg upload -z <key> C:\dev\habitat-aspnet-sample\habitat\results\core-habitat-aspnet-sample-0.2.0-20170219124600-x86_64-windows.hart
```
