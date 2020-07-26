# ASP.NET Core MVC 3.1 + Vue (TypeScript)構成のテンプレート

デバック実行時、ビルド時にサーバーサイドのリソースだけでなく、フロントエンドのリソースに対しても操作するように環境構築。
テンプレートの動作確認のためにMVCで構築しているが、WebApiで構築する場合は、`dotnet new mvc`の箇所を、`dotnet new webapi`にする。
TypeScriptはvue-cliからインストールしているだけなので、wwwroot配下にもTypescriptを設置したい場合は、別途dotnet側でTypescriptへのコンパイル設定が必要になる。

# Projectの作成

```
dotnet new mvc
dotnet add package Microsoft.AspNetCore.SpaServices.Extensions
dotnet add package Microsoft.VisualStudio.Web.BrowserLink
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
dotnet tool install --global dotnet-aspnet-codegenerator
```

# vue-cliのインストール

vue-cliはプロジェクトに対してではなく、端末に対してインストールするため、
すでにインストールしている場合は実行する必要はない。
```
npm i -g @vue-cli
```

# Vueセットアップ

```
vue ui
```
ブラウザからGUI操作でvueプロジェクトの作成/管理を行う事ができる。    
すでにプロジェクトをいくつか作成している場合は、作成済みのプロジェクトの管理ページが開くことがあるが、
その場合は、画面左上に表示されているプロジェクト名をクリックしてvueプロジェクトマネージャーを開くこと。

プリセットを手動にし、TypeScript、Router、Vuexを選択する。 
Linter/Formatterはコーディング規約を厳密にするためのモジュールなので、プロジェクトに合わせて選択するかを選ぶ。
コンポーネントをクラス記法で記述できるように、`Use Class-Syntax...`はチェックを入れる。  
IEで動かすにはJavaScriptをES5にコンパイルする必要があるため、`Use Babel alongside...`はチェックを外しておく。（ここにチェックを入れると、TargetがES2015になる。）  
`Use History mode...`はチェックを入れる。
Vue Routerは、画面内の移動としてブラウザに認識させるために、コンパイル後にSPA内のRoutingパスのRootに#をつける**ハッシュモード**で動作する。    
ただし、サーバ側でfall-backルートをSPAのビルド後のパスに設定すれば、**History mode**で動作する。  
History modeの仕組み上、サーバ側でrootingマッチしなかった場合は常にSPAを返すようになるため、ルートマッチングしなかった場合の404ページはSPA側で用意する必要がある。

ajaxでフロントエンド側とサーバのやり取りをするため、axiosをインストールしておく。
```
cd frontend
npm i axios
```
その他フロントエンドアプリにモジュールを追加する必要がある場合は、frontendフォルダ内で`npm i`を実行すること。

# Project直下の設定ファイルの変更

## 本番用にcsprojを編集する

core_pj.csproj
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
	<PropertyGroup>
		<TargetFramework>netcoreapp3.1</TargetFramework>
		<SpaRoot>frontend\</SpaRoot>
    	<DefaultItemExcludes>$(DefaultItemExcludes);$(SpaRoot)node_modules\**</DefaultItemExcludes>
    	<TypeScriptCompileBlocked>true</TypeScriptCompileBlocked>
    	<TypeScriptToolsVersion>Latest</TypeScriptToolsVersion>
    	<IsPackable>false</IsPackable>
	</PropertyGroup>
	<Target Name="PublishRunWebpack" AfterTargets="ComputeFilesToPublish">

    <Exec WorkingDirectory="$(SpaRoot)" Command="npm install" />
    <Exec WorkingDirectory="$(SpaRoot)" Command="npm run build" />


    <ItemGroup>
      <DistFiles Include="$(SpaRoot)dist\**" />
      <ResolvedFileToPublish Include="@(DistFiles->'%(FullPath)')" Exclude="@(ResolvedFileToPublish)">
        <RelativePath>%(DistFiles.Identity)</RelativePath>
        <CopyToPublishDirectory>PreserveNewest</CopyToPublishDirectory>
      </ResolvedFileToPublish>
    </ItemGroup>
  </Target>
	<ItemGroup>
		<PackageReference Include="Microsoft.AspNetCore.SpaServices.Extensions" Version="3.1.6" />
		<PackageReference Include="Microsoft.VisualStudio.Web.BrowserLink" Version="2.2.0" />
		<PackageReference Include="Microsoft.VisualStudio.Web.CodeGeneration.Design" Version="3.1.3" />
	</ItemGroup>
	<ItemGroup>

    <Content Remove="$(SpaRoot)**" />
    <None Remove="$(SpaRoot)**" />
    <None Include="$(SpaRoot)**" Exclude="$(SpaRoot)node_modules\**" />
  </ItemGroup>

  <Target Name="DebugEnsureNodeEnv" BeforeTargets="Build" Condition=" '$(Configuration)' == 'Debug' And !Exists('$(SpaRoot)node_modules') ">

    <Exec Command="node --version" ContinueOnError="true">
      <Output TaskParameter="ExitCode" PropertyName="ErrorCode" />
    </Exec>
    <Error Condition="'$(ErrorCode)' != '0'" Text="Node.js is required to build and run this project. To continue, please install Node.js from https://nodejs.org/, and then restart your command prompt or IDE." />
    <Message Importance="high" Text="Restoring dependencies using 'npm'. This may take several minutes..." />
    <Exec WorkingDirectory="$(SpaRoot)" Command="npm install" />
  </Target>
</Project>

```

## SPA起動設定
開発環境でサーバ起動時にWebpackのビルドが走るように細工する。

ASP.NET CoreサーバからNodeをキックするためのMiddleware  

言うまでもなく、namespaceはプロジェクトに合わせてかえること。

VueConnection.cs
```cs
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.SpaServices;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using System;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Net.NetworkInformation;
using System.Runtime.InteropServices;
using System.Threading.Tasks;

namespace core_pj.Middleware.VueConnection
{
    public static class VueConnection
    {
        private static int Port { get; } = 8080;
        private static Uri DevelopmentServerEndpoint { get; } = new Uri($"http://localhost:{Port}");
        private static TimeSpan Timeout { get; } = TimeSpan.FromSeconds(60);

        private static string DoneMessage { get; } = "DONE  Compiled successfully in";

        public static void UseVueDevelopmentServer(this ISpaBuilder spa)
        {
            spa.UseProxyToSpaDevelopmentServer(async () =>
            {
                var loggerFactory = spa.ApplicationBuilder.ApplicationServices.GetService<ILoggerFactory>();
                var logger = loggerFactory.CreateLogger("Vue");

                if (IsRunning())
                {
                    return DevelopmentServerEndpoint;
                }


                var isWindows = RuntimeInformation.IsOSPlatform(OSPlatform.Windows);
                var processInfo = new ProcessStartInfo
                {
                    FileName = isWindows ? "cmd" : "npm",
                    Arguments = $"{(isWindows ? "/c npm " : "")}run serve",
                    WorkingDirectory = "frontend",
                    RedirectStandardError = true,
                    RedirectStandardInput = true,
                    RedirectStandardOutput = true,
                    UseShellExecute = false,
                };
                var process = Process.Start(processInfo);
                var tcs = new TaskCompletionSource<int>();
                _ = Task.Run(() =>
                {
                    try
                    {
                        string line;
                        while ((line = process.StandardOutput.ReadLine()) != null)
                        {
                            logger.LogInformation(line);
                            if (!tcs.Task.IsCompleted && line.Contains(DoneMessage))
                            {
                                tcs.SetResult(1);
                            }
                        }
                    }
                    catch (EndOfStreamException ex)
                    {
                        logger.LogError(ex.ToString());
                        tcs.SetException(new InvalidOperationException("'npm run serve' failed.", ex));
                    }
                });
                _ = Task.Run(() =>
                {
                    try
                    {
                        string line;
                        while ((line = process.StandardError.ReadLine()) != null)
                        {
                            logger.LogError(line);
                        }
                    }
                    catch (EndOfStreamException ex)
                    {
                        logger.LogError(ex.ToString());
                        tcs.SetException(new InvalidOperationException("'npm run serve' failed.", ex));
                    }
                });

                var timeout = Task.Delay(Timeout);
                if (await Task.WhenAny(timeout, tcs.Task) == timeout)
                {
                    throw new TimeoutException();
                }

                return DevelopmentServerEndpoint;
            });

        }

        private static bool IsRunning() => IPGlobalProperties.GetIPGlobalProperties()
                .GetActiveTcpListeners()
                .Select(x => x.Port)
                .Contains(Port);
    }
}
```

Startup.cs
```diff
using Microsoft.Extensions.Hosting;
+ using Microsoft.AspNetCore.SpaServices.Webpack;

//(中略)
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddControllersWithViews();
+            services.AddSpaStaticFiles(configuration =>
+            {
+                configuration.RootPath = "frontend/dist";
+            });
        }


        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
+               app.UseBrowserLink();
            }
            else
            {
                app.UseExceptionHandler("/Error");
                app.UseHsts();
            }

            app.UseHttpsRedirection();
            app.UseStaticFiles();
+            app.UseSpaStaticFiles();
            app.UseRouting();
            app.UseAuthorization();

            app.UseEndpoints(endpoints =>
            {
-                endpoints.MapControllerRoute(
-                    name: "default",
-                    pattern: "{controller=Home}/{action=Index}/{id?}");
+                endpoints.MapControllers();
            });
+            app.UseSpa(spa =>
+            {
+                spa.Options.SourcePath = "frontend";
+                if (env.IsDevelopment())
+                {
+                    spa.UseVueDevelopmentServer();
+                }
+            });
        }
```

# 開発実行
F5: デバッグあり
Ctrl + F5 デバッグ無し

# ビルド
```
dotnet Publish
```
bin\Debug\netcoreapp3.1\publish\
に出力される。

パス指定する場合は -o オプションをつけること。
