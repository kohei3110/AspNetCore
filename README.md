# .NET Core での Secret 運用

## 方式
1. UserSecret を使用
2. Azure Key Vault を使用

### 1. UserSecret を使った秘密文字の管理

開発時、アプリケーションで使用する機密データを管理する方式として、UserSecret を活用する。

#### サンプルプログラムの準備

```powershell
git clone https://github.com/dotnet/AspNetCore.Docs.git
cd .\AspNetCore.Docs\aspnetcore\security\app-secrets\samples\2.x\UserSecrets
```

#### シークレットストレージの有効化

```powershell
dotnet user-secrets init
```
`vaulsample.csproj` に、`<UserSecretsId>` が追加される。

#### シークレットを設定する
Key-Value で定義されるアプリケーションシークレットを定義する。

```powershell
dotnet user-secrets set "Movies:ServiceApiKey" "12345"
```

#### シークレットにアクセスする
セットしたシークレットには、下記のようにコンストラクタを通じてアクセス可能。

```C#:Startup.cs
    public class Startup
    {
        private string _moviesApiKey = null;

        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        public void ConfigureServices(IServiceCollection services)
        {
            // Read the secret via the Configuration API
            _moviesApiKey = Configuration["Movies:ServiceApiKey"];
        }
    
    ・・・
```

#### シークレットを POCO にマップする
.NET Configuration API の object graph binding を使用して、シークレットを POCO にマッピング可能。

```C#:Startup3.cs
            var moviesConfig = Configuration.GetSection("Movies")
                                            .Get<MovieSettings>();
            _moviesApiKey = moviesConfig.ServiceApiKey;
```

上記のプロパティは、 POCO にて定義されている。

```MovieSettings.cs
    public class MovieSettings
    {
        public string ConnectionString { get; set; }

        public string ServiceApiKey { get; set; }
    }
```

#### シークレットを使用した文字列の置換
以下のように平文でパスワードを保存するのは危険。

```appsettings.json
{
  "ConnectionStrings": {
    "Movies": "Server=(localdb)\\mssqllocaldb;Database=Movie-1;User Id=johndoe;Password=pass123;MultipleActiveResultSets=true"
  }
}
```

パスワードをシークレットとして保存し、プログラムからアクセスする。

```powershell
dotnet user-secrets set "DbPassword" "pass123"
```

`appsettings.json` からパスワードを削除。

```appsettings.json
{
  "ConnectionStrings": {
    "Movies": "Server=(localdb)\\mssqllocaldb;Database=Movie-1;User Id=johndoe;MultipleActiveResultSets=true"
  }
}
```

シークレットをオブジェクトのプロパティに設定することで、安全にセキュアな情報を参照可能。

```C#:Startup2.cs
    ・・・
        public void ConfigureServices(IServiceCollection services)
        {
            var builder = new SqlConnectionStringBuilder(
                Configuration.GetConnectionString("Movies"));
            builder.Password = Configuration["DbPassword"];
            _connection = builder.ConnectionString;
        }
```

#### シークレットを一覧表示する

```powershell
> dotnet user-secrets list
Movies:ServiceApiKey = 12345
DbPassword = pass123
```

### 2. Azure Key Vault を使用
本番環境でシークレットを参照するには、Azure Key Vault を使用する。<br>
Azure Key Vault を使用する一般的なシナリオは下記の通り。

 - 機密性の高い構成データへのアクセスを制御する。
 - 構成データを保存するときに、FIPS 140-2 Level 2 で検証されたハードウェアセキュリティモジュール (Hsm) の要件を満たしている。

Azure CLI から、Key Vault シークレットを作成する。

```powershell
> az login
> az group create --name "secret-demo" --location westus
> az keyvault create --name secret-demo-vault --resource-group "secret-demo" --location westus
> az keyvault secret set --vault-name secret-demo-vault --name "SecretName" --value "secret_value_1_prod"
> az keyvault secret set --vault-name secret-demo-vault --name "Section--SecretName" --value "secret_value_2_prod"
cd ../../../../key-vault-configuration/samples/3.x/SampleApp/
```

`SecretClient` インスタンスは、 `KeyVaultSecretManager` インスタンスとともに使用され、クレデンシャルの値を読み込む。

```C#:Program.cs
#if Managed
#region snippet2
        // using Azure.Security.KeyVault.Secrets;
        // using Azure.Identity;
        // using Azure.Extensions.AspNetCore.Configuration.Secrets;

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureAppConfiguration((context, config) =>
                {
                    if (context.HostingEnvironment.IsProduction())
                    {
                        var builtConfig = config.Build();
                        var secretClient = new SecretClient(
                            new Uri($"https://{builtConfig["KeyVaultName"]}.vault.azure.net/"),
                            new DefaultAzureCredential());
                        config.AddAzureKeyVault(secretClient, new KeyVaultSecretManager());
                    }
                })
                .ConfigureWebHostDefaults(webBuilder => webBuilder.UseStartup<Startup>());
```

■参考

 - [ASP.NET Core での開発におけるアプリ シークレットの安全な保存](https://docs.microsoft.com/ja-jp/aspnet/core/security/app-secrets?view=aspnetcore-5.0&tabs=windows)

 - [ASP.NET Core の Azure Key Vault 構成プロバイダー](https://docs.microsoft.com/ja-jp/aspnet/core/security/key-vault-configuration?view=aspnetcore-5.0)

 - [Azure.Security.KeyVault.Secrets](https://docs.microsoft.com/en-us/dotnet/api/azure.security.keyvault.secrets.secretclient?view=azure-dotnet)

■WIP

 - ビルドに失敗する。

```powershell
> dotnet build
.NET 向け Microsoft (R) Build Engine バージョン 16.9.0+57a23d249
Copyright (C) Microsoft Corporation.All rights reserved.

  復元対象のプロジェクトを決定しています...
C:\Users\koheisaito\OneDrive - Microsoft\Desktop\work\demos\AspNetCore.Docs\aspnetcore\security\key-vault-configuration\samples\3.x\SampleApp\SampleApp.csproj : error NU1101: パッケージ Azure.Extensions.AspNetCore.Configuration.Secrets が見つかりません。ソース Microsoft Visual Studio Offline Packages には、この ID のパッケージが存在しません。
C:\Users\koheisaito\OneDrive - Microsoft\Desktop\work\demos\AspNetCore.Docs\aspnetcore\security\key-vault-configuration\samples\3.x\SampleApp\SampleApp.csproj : error NU1101: パッケージ Azure.Identity が見つかりません。ソ
ース Microsoft Visual Studio Offline Packages には、この ID のパッケージが存在しません。
  C:\Users\koheisaito\OneDrive - Microsoft\Desktop\work\demos\AspNetCore.Docs\aspnetcore\security\key-vault-configuration\samples\3.x\SampleApp\SampleApp.csproj を復元できませんでした (144 ms)。

ビルドに失敗しました。

C:\Users\koheisaito\OneDrive - Microsoft\Desktop\work\demos\AspNetCore.Docs\aspnetcore\security\key-vault-configuration\samples\3.x\SampleApp\SampleApp.csproj : error NU1101: パッケージ Azure.Extensions.AspNetCore.Configuration.Secrets が見つかりません。ソース Microsoft Visual Studio Offline Packages には、この ID のパッケージが存在しません。
C:\Users\koheisaito\OneDrive - Microsoft\Desktop\work\demos\AspNetCore.Docs\aspnetcore\security\key-vault-configuration\samples\3.x\SampleApp\SampleApp.csproj : error NU1101: パッケージ Azure.Identity が見つかりません。ソ
ース Microsoft Visual Studio Offline Packages には、この ID のパッケージが存在しません。
    0 個の警告
    2 エラー

経過時間 00:00:01.31
```