---
title: ASP.NET Core Blazor CSS isolation
author: guardrex
description: Learn how CSS isolation scopes CSS to Razor components, which can simplify CSS and avoid collisions with other components or libraries.
monikerRange: '>= aspnetcore-5.0'
ms.author: wpickett
ms.custom: mvc
ms.date: 11/12/2024
uid: blazor/components/css-isolation
---
# ASP.NET Core Blazor CSS isolation

[!INCLUDE[](~/includes/not-latest-version.md)]

By [Dave Brock](https://twitter.com/daveabrock)

This article explains how CSS isolation scopes CSS to Razor components, which can simplify CSS and avoid collisions with other components or libraries.

Isolate CSS styles to individual pages, views, and components to reduce or avoid:

* Dependencies on global styles that can be challenging to maintain.
* Style conflicts in nested content.

## Enable CSS isolation 

To define component-specific styles, create a `.razor.css` file matching the name of the `.razor` file for the component in the same folder. The `.razor.css` file is a *scoped CSS file*. 

For an `Example` component in an `Example.razor` file, create a file alongside the component named `Example.razor.css`. The `Example.razor.css` file must reside in the same folder as the `Example` component (`Example.razor`). The "`Example`" base name of the file is **not** case-sensitive.

`Example.razor`:

```razor
@page "/example"

<h1>Scoped CSS Example</h1>
```

`Example.razor.css`:

```css
h1 { 
    color: brown;
    font-family: Tahoma, Geneva, Verdana, sans-serif;
}
```

**The styles defined in `Example.razor.css` are only applied to the rendered output of the `Example` component.** CSS isolation is applied to HTML elements in the matching Razor file. Any `h1` CSS declarations defined elsewhere in the app don't conflict with the `Example` component's styles.

> [!NOTE]
> In order to guarantee style isolation when bundling occurs, importing CSS in Razor code blocks isn't supported.

## CSS isolation bundling

CSS isolation occurs at build time. Blazor rewrites CSS selectors to match markup rendered by the component. The rewritten CSS styles are bundled and produced as a static asset. The stylesheet is referenced inside the `<head>` tag ([location of `<head>` content](xref:blazor/project-structure#location-of-head-and-body-content)). The following `<link>` element is added to an app created from the Blazor project templates:

:::moniker range=">= aspnetcore-9.0"

Blazor Web Apps:

```html
<link href="@Assets["{PACKAGE ID/ASSEMBLY NAME}.styles.css"]" rel="stylesheet">
```

Standalone Blazor WebAssembly apps:

```html
<link href="{PACKAGE ID/ASSEMBLY NAME}.styles.css" rel="stylesheet">
```

:::moniker-end

:::moniker range="< aspnetcore-9.0"

```html
<link href="{PACKAGE ID/ASSEMBLY NAME}.styles.css" rel="stylesheet">
```

:::moniker-end

The `{PACKAGE ID/ASSEMBLY NAME}` placeholder is the project's package ID (`<PackageId>` in the project file) for a library or assembly name for an app.

:::moniker range="< aspnetcore-8.0"

The following example is from a hosted Blazor WebAssembly **:::no-loc text="Client":::** app. The app's assembly name is `BlazorSample.Client`, and the `<link>` is added by the Blazor WebAssembly project template when the project is created with the Hosted option (`-ho|--hosted` option using the .NET CLI or **ASP.NET Core Hosted** checkbox using Visual Studio):

```html
<link href="BlazorSample.Client.styles.css" rel="stylesheet">
```

:::moniker-end

Within the bundled file, each component is associated with a scope identifier. For each styled component, an HTML attribute is appended with the format `b-{STRING}`, where the `{STRING}` placeholder is a ten-character string generated by the framework. The identifier is unique for each app. In the rendered `Counter` component, Blazor appends a scope identifier to the `h1` element:

```html
<h1 b-3xxtam6d07>
```

The `{PACKAGE ID/ASSEMBLY NAME}.styles.css` file uses the scope identifier to group a style declaration with its component. The following example provides the style for the preceding `<h1>` element:

```css
/* /Components/Pages/Counter.razor.rz.scp.css */
h1[b-3xxtam6d07] {
    color: brown;
}
```

At build time, a project bundle is created with the convention `obj/{CONFIGURATION}/{TARGET FRAMEWORK}/scopedcss/projectbundle/{PACKAGE ID/ASSEMBLY NAME}.bundle.scp.css`, where the placeholders are:

* `{CONFIGURATION}`: The app's build configuration (for example, `Debug`, `Release`).
* `{TARGET FRAMEWORK}`: The target framework (for example, `net6.0`).
* `{PACKAGE ID/ASSEMBLY NAME}`: The project's package ID (`<PackageId>` in the project file) for a library or assembly name for an app (for example, `BlazorSample`).

## Child component support

CSS isolation only applies to the component you associate with the format `{COMPONENT NAME}.razor.css`, where the `{COMPONENT NAME}` placeholder is usually the component name. To apply changes to a child component, use the `::deep` [pseudo-element](https://developer.mozilla.org/docs/Web/CSS/Pseudo-elements) to any descendant elements in the parent component's `.razor.css` file. The `::deep` pseudo-element selects elements that are *descendants* of an element's generated scope identifier. 

The following example shows a parent component called `Parent` with a child component called `Child`.

`Parent.razor`:

```razor
@page "/parent"

<div>
    <h1>Parent component</h1>

    <Child />
</div>
```

`Child.razor`:

```razor
<h1>Child Component</h1>
```

Update the `h1` declaration in `Parent.razor.css` with the `::deep` pseudo-element to signify the `h1` style declaration must apply to the parent component and its children.

`Parent.razor.css`:

```css
::deep h1 { 
    color: red;
}
```

The `h1` style now applies to the `Parent` and `Child` components without the need to create a separate scoped CSS file for the child component.

The `::deep` pseudo-element only works with descendant elements. The following markup applies the `h1` styles to components as expected. The parent component's scope identifier is applied to the `div` element, so the browser knows to inherit styles from the parent component.

`Parent.razor`:

```razor
<div>
    <h1>Parent</h1>

    <Child />
</div>
```

However, excluding the `div` element removes the descendant relationship. In the following example, the style is **not** applied to the child component.

`Parent.razor`:

```razor
<h1>Parent</h1>

<Child />
```

The `::deep` pseudo-element affects where the scope attribute is applied to the rule. When you define a CSS rule in a scoped CSS file, the scope is applied to the right most element. For example: `div > a` is transformed to `div > a[b-{STRING}]`, where the `{STRING}` placeholder is a ten-character string generated by the framework (for example, `b-3xxtam6d07`). If you instead want the rule to apply to a different selector, the `::deep` pseudo-element allows you do so. For example, `div ::deep > a` is transformed to `div[b-{STRING}] > a` (for example, `div[b-3xxtam6d07] > a`).

The ability to attach the `::deep` pseudo-element to any HTML element allows you to create scoped CSS styles that affect elements rendered by other components when you can determine the structure of the rendered HTML tags. For a component that renders an hyperlink tag (`<a>`) inside another component, ensure the component is wrapped in a `div` (or any other element) and use the rule `::deep > a` to create a style that's only applied to that component when the parent component renders.

> [!IMPORTANT]
> Scoped CSS only applies to ***HTML elements*** and not to Razor components or Tag Helpers, including elements with a Tag Helper applied, such as `<input asp-for="..." />`.

## CSS preprocessor support

CSS preprocessors are useful for improving CSS development by utilizing features such as variables, nesting, modules, mixins, and inheritance. While CSS isolation doesn't natively support CSS preprocessors such as Sass or Less, integrating CSS preprocessors is seamless as long as preprocessor compilation occurs before Blazor rewrites the CSS selectors during the build process. Using Visual Studio for example, configure existing preprocessor compilation as a **Before Build** task in the Visual Studio Task Runner Explorer.

Many third-party NuGet packages, such as [`AspNetCore.SassCompiler`](https://www.nuget.org/packages/AspNetCore.SassCompiler#readme-body-tab), can compile SASS/SCSS files at the beginning of the build process before CSS isolation occurs.

## CSS isolation configuration

CSS isolation is designed to work out-of-the-box but provides configuration for some advanced scenarios, such as when there are dependencies on existing tools or workflows.

### Customize scope identifier format

:::moniker range=">= aspnetcore-8.0"

Scope identifiers use the format `b-{STRING}`, where the `{STRING}` placeholder is a ten-character string generated by the framework. To customize the scope identifier format, update the project file to a desired pattern:

```xml
<ItemGroup>
  <None Update="Components/Pages/Example.razor.css" CssScope="custom-scope-identifier" />
</ItemGroup>
```

In the preceding example, the CSS generated for `Example.razor.css` changes its scope identifier from `b-{STRING}` to `custom-scope-identifier`.

Use scope identifiers to achieve inheritance with scoped CSS files. In the following project file example, a `BaseComponent.razor.css` file contains common styles across components. A `DerivedComponent.razor.css` file inherits these styles.

```xml
<ItemGroup>
  <None Update="Components/Pages/BaseComponent.razor.css" CssScope="custom-scope-identifier" />
  <None Update="Components/Pages/DerivedComponent.razor.css" CssScope="custom-scope-identifier" />
</ItemGroup>
```

Use the wildcard (`*`) operator to share scope identifiers across multiple files:

```xml
<ItemGroup>
  <None Update="Components/Pages/*.razor.css" CssScope="custom-scope-identifier" />
</ItemGroup>
```

:::moniker-end

:::moniker range="< aspnetcore-8.0"

Scope identifiers use the format `b-{STRING}`, where the `{STRING}` placeholder is a ten-character string generated by the framework. To customize the scope identifier format, update the project file to a desired pattern:

```xml
<ItemGroup>
  <None Update="Pages/Example.razor.css" CssScope="custom-scope-identifier" />
</ItemGroup>
```

In the preceding example, the CSS generated for `Example.razor.css` changes its scope identifier from `b-{STRING}` to `custom-scope-identifier`.

Use scope identifiers to achieve inheritance with scoped CSS files. In the following project file example, a `BaseComponent.razor.css` file contains common styles across components. A `DerivedComponent.razor.css` file inherits these styles.

```xml
<ItemGroup>
  <None Update="Pages/BaseComponent.razor.css" CssScope="custom-scope-identifier" />
  <None Update="Pages/DerivedComponent.razor.css" CssScope="custom-scope-identifier" />
</ItemGroup>
```

Use the wildcard (`*`) operator to share scope identifiers across multiple files:

```xml
<ItemGroup>
  <None Update="Pages/*.razor.css" CssScope="custom-scope-identifier" />
</ItemGroup>
```

:::moniker-end

### Change base path for static web assets

The `scoped.styles.css` file is generated at the root of the app. In the project file, use the [`<StaticWebAssetBasePath>` property](xref:blazor/fundamentals/static-files#static-web-asset-base-path) to change the default path. The following example places the `scoped.styles.css` file, and the rest of the app's assets, at the `_content` path:

```xml
<PropertyGroup>
  <StaticWebAssetBasePath>_content/$(PackageId)</StaticWebAssetBasePath>
</PropertyGroup>
```

### Disable automatic bundling

To opt out of how Blazor publishes and loads scoped files at runtime, use the `DisableScopedCssBundling` property. When using this property, it means other tools or processes are responsible for taking the isolated CSS files from the `obj` directory and publishing and loading them at runtime:

```xml
<PropertyGroup>
  <DisableScopedCssBundling>true</DisableScopedCssBundling>
</PropertyGroup>
```

### Disable CSS isolation

Disable CSS isolation for a project by setting the `<ScopedCssEnabled>` property to `false` in the app's project file:

```xml
<ScopedCssEnabled>false</ScopedCssEnabled>
```

## Razor class library (RCL) support

Isolated styles for components in a NuGet package or [Razor class library (RCL)](xref:razor-pages/ui-class) are automatically bundled:

* The app uses CSS imports to reference the RCL's bundled styles. For a class library named `ClassLib` and a Blazor app with a `BlazorSample.styles.css` stylesheet, the RCL's stylesheet is imported at the top of the app's stylesheet:
  
  ```css
  @import '_content/ClassLib/ClassLib.bundle.scp.css';
  ```

* The RCL's bundled styles aren't published as a static web asset of the app that consumes the styles.

For more information on RCLs, see the following articles:

* <xref:blazor/components/class-libraries>
* <xref:razor-pages/ui-class>

:::moniker range=">= aspnetcore-6.0"

## Additional resources

* [Razor Pages CSS isolation](xref:razor-pages/index#css-isolation)
* [MVC CSS isolation](xref:mvc/views/overview#css-isolation)

:::moniker-end
