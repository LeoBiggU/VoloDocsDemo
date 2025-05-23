# Blazor UI: Basic Theme

````json
//[doc-params]
{
    "UI": ["Blazor", "BlazorServer"]
}
````

The Basic Theme is a theme implementation for the Blazor UI. It is a minimalist theme that doesn't add any styling on top of the plain [Bootstrap](https://getbootstrap.com/). You can take the Basic Theme as the **base theme** and build your own theme or styling on top of it. See the *Customization* section.

> If you are looking for a professional, enterprise ready theme, you can check the [Lepton Theme](https://abp.io/themes), which is a part of the [ABP](https://abp.io/).

> See the [Theming document](theming.md) to learn about themes.

## Installation

If you need to manually this theme, follow the steps below:

{{if UI == "Blazor"}}

* Install the [Volo.Abp.AspNetCore.Components.WebAssembly.BasicTheme.Bundling](https://www.nuget.org/packages/Volo.Abp.AspNetCore.Components.WebAssembly.BasicTheme.Bundling) NuGet package to your `Blazor` project.
* Add `AbpAspNetCoreComponentsWebAssemblyBasicThemeBundlingModule` into the `[DependsOn(...)]` attribute for your [module class](../../architecture/modularity/basics.md) in the your `Blazor` project.
* Install the [Volo.Abp.AspNetCore.Components.WebAssembly.BasicTheme](https://www.nuget.org/packages/Volo.Abp.AspNetCore.Components.WebAssembly.BasicTheme) NuGet package to your `Blazor.Client` project.
* Add `AbpAspNetCoreComponentsWebAssemblyBasicThemeModule` into the `[DependsOn(...)]` attribute for your [module class](../../architecture/modularity/basics.md) in the your `Blazor.Client` project.

Update `Routes.razor` file in `Blazor.Client` project as below:

````csharp
@using Volo.Abp.AspNetCore.Components.Web.BasicTheme.Themes.Basic
@using Volo.Abp.AspNetCore.Components.WebAssembly.WebApp
<Router AppAssembly="typeof(Program).Assembly" AdditionalAssemblies="WebAppAdditionalAssembliesHelper.GetAssemblies<YourBlazorClientModule>()">
    <Found Context="routeData">
        <AuthorizeRouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)">
            <NotAuthorized>
                <RedirectToLogin />
            </NotAuthorized>
        </AuthorizeRouteView>
    </Found>
</Router>
````

{{end}}

{{if UI == "BlazorServer"}}

* Make sure [AspNetCore Basic Theme](../mvc-razor-pages/basic-theme.md) installation steps are completed. 

* Install the [Volo.Abp.AspNetCore.Components.Server.BasicTheme](https://www.nuget.org/packages/Volo.Abp.AspNetCore.Components.Server.BasicTheme) NuGet package to your web project.

* Add `AbpAspNetCoreComponentsServerBasicThemeModule` into the `[DependsOn(...)]` attribute for your [module class](../../architecture/modularity/basics.md) in the your Blazor UI project.

* Perform following changes in `App.razor` file
  * Add usings at the top of the page.
    ```html
    @using Volo.Abp.AspNetCore.Components.Server.BasicTheme.Bundling
    ```
  * Then replace script & style bunles as following
    ```
    <AbpStyles BundleName="@BlazorBasicThemeBundles.Styles.Global" />
    ```

    ```
    <AbpScripts BundleName="@BlazorBasicThemeBundles.Scripts.Global" />
    ```

Update `Routes.razor` file as below:

````csharp
@using Volo.Abp.AspNetCore.Components.Web.BasicTheme.Themes.Basic
@using Volo.Abp.AspNetCore.Components.Web.Theming.Routing
@using Microsoft.Extensions.Options
@inject IOptions<AbpRouterOptions> RouterOptions
<Router AppAssembly="typeof(Program).Assembly" AdditionalAssemblies="RouterOptions.Value.AdditionalAssemblies">
    <Found Context="routeData">
        <AuthorizeRouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)">
            <NotAuthorized>
                <RedirectToLogin />
            </NotAuthorized>
        </AuthorizeRouteView>
    </Found>
</Router>
````

{{end}}

## The Layout

![basic-theme-application-layout](../../../images/basic-theme-application-layout.png)

Application Layout implements the following parts, in addition to the common parts mentioned above;

* [Branding](branding.md) Area
* Main [Menu](navigation-menu.md)
* Main [Toolbar](toolbars.md) with Language Selection & User Menu
* [Page Alerts](page-alerts.md)

## Customization

You have two options two customize this theme:

### Overriding Styles / Components

In this approach, you continue to use the theme as NuGet and NPM packages and customize the parts you need to. There are several ways to customize it;

#### Override the Styles

You can simply override the styles in the Global Styles file of your application.

#### Override the Components

See the [Customization / Overriding Components](customization-overriding-components.md) to learn how you can replace components, customize and extend the user interface.

### Overriding the Menu Item
Basic theme supports overriding a single menu item with a custom component. You can create a custom component and call `UseComponent` extension method in the **MenuContributor**.

```csharp
using Volo.Abp.UI.Navigation;

//...

context.Menu.Items.Add(
  new ApplicationMenuItem("Custom.1", "My Custom Menu", "#")
    .UseComponent(typeof(MyMenuItemComponent)));
```

```html
<li class="nav-item">
    <a href="#" class="nav-link">
        My Custom Menu
    </a>
</li>
```

### Copy & Customize

You can run the following [ABP CLI](../../../cli) command in **Blazor{{if UI == "Blazor"}}WebAssembly{{else}} Server{{end}}** project directory to copy the source code to your solution:

{{if UI == "Blazor"}}

`abp add-package Volo.Abp.AspNetCore.Components.WebAssembly.BasicTheme --with-source-code --add-to-solution-file`

Then, navigate to downloaded `Volo.Abp.AspNetCore.Components.WebAssembly.BasicTheme` project directory and run:

`abp add-package Volo.Abp.AspNetCore.Components.Web.BasicTheme --with-source-code --add-to-solution-file`

{{end}}

{{if UI == "BlazorServer"}}

`abp add-package Volo.Abp.AspNetCore.Components.Server.BasicTheme --with-source-code --add-to-solution-file`

Then, navigate to downloaded `Volo.Abp.AspNetCore.Components.Server.BasicTheme` project directory and run:

`abp add-package Volo.Abp.AspNetCore.Components.Web.BasicTheme --with-source-code --add-to-solution-file`

{{end}}

----

Or, you can download the source code of the Basic Theme, manually copy the project content into your solution, re-arrange the package/module dependencies (see the Installation section above to understand how it was installed to the project) and freely customize the theme based on your application requirements.

{{if UI == "Blazor"}}
- [Basic Theme Source Code](https://github.com/abpframework/abp/blob/dev/modules/basic-theme/src/Volo.Abp.AspNetCore.Components.WebAssembly.BasicTheme)
{{end}}

{{if UI == "BlazorServer"}}
- [Basic Theme Source Code](https://github.com/abpframework/abp/blob/dev/modules/basic-theme/src/Volo.Abp.AspNetCore.Components.Server.BasicTheme)
{{end}}

## See Also

* [Theming](theming.md)
