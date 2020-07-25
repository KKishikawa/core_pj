# Projectの作成

```
dotnet new mvc
dotnet add package Microsoft.AspNetCore.SpaServices.Extensions
dotnet add package Microsoft.VisualStudio.Web.BrowserLink
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
dotnet tool install --global dotnet-aspnet-codegenerator
```

# Nodeパッケージのインストール

```
npm i -g @vue-cli
```

# Vueセットアップ

```
vue ui
```
ブラウザからvueプロジェクトの作成を行う事ができる。

ajaxでフロントエンド側とサーバのやり取りをするため、axiosをインストールしておく。
```
cd frontend
npm i axios
```

# Project直下の設定ファイルの変更

## Webpackの設定

開発環境と本番環境で設定を分ける。  
（開発環境ではNodeでホストして実行/デバッグを行うが、本番環境ではビルド後の静的ファイルを使用するため。）   
HTMLLoaderなどは特に不要。

webpack.common.js
```js
const path = require('path');
const VueLoaderPlugin = require('vue-loader/lib/plugin');

module.exports = {
    entry: { main: './Frontend/index.ts' },
    output: {
        path: path.resolve(__dirname, 'wwwroot'),
        filename: 'js/[name].js',
        publicPath: '/'
    },
    resolve: {
        extensions: ['.ts', '.js', '.vue', '.json'],
        alias: {
            'vue$': 'vue/dist/vue.esm.js'
        }
    },
    module: {
        rules: [
            {
                test: /\.vue$/,
                loader: 'vue-loader',
                options: {}
            },
            {
                test: /\.tsx?$/,
                loader: 'ts-loader',
                exclude: /node_modules/,
                options: {
                    appendTsSuffixTo: [/\.vue$/]
                }
            }
        ]
    },
    plugins: [
        new VueLoaderPlugin(),
    ]
};
```

開発環境ではデバッグのためにソースマップを有効にする。

webpack.dev.js
```js
const merge = require('webpack-merge');
const common = require('./webpack.common.js');

module.exports = merge(common, {
    mode: 'development',
    devtool: 'inline-source-map',
    devServer: {
        contentBase: './wwwroot',
    }
});
```

本番環境ではminifyする。

webpack.prod.js
```js
const merge = require('webpack-merge');
const common = require('./webpack.common.js');
const TerserPlugin = require('terser-webpack-plugin');

module.exports = merge(common, {
    mode: 'production',
    optimization: {
        minimizer: [
            new TerserPlugin({
                terserOptions: { ecma: 5, compress: true,
                    output: { comments: false, beautify: false }
                }
            })
        ]
    }
});
```

## package.jsonを編集して、Webpackのコマンドを呼び分ける

package.json
```diff
"scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
+   "build": "webpack --config webpack.dev.js"
+   "release": "webpack --config webpack.prod.js"

  },
```

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
