---
title: ASP.NET Core Razor component lifecycle
author: guardrex
description: Learn about the ASP.NET Core Razor component lifecycle and how to use lifecycle events.
monikerRange: '>= aspnetcore-3.1'
ms.author: wpickett
ms.custom: mvc
ms.date: 11/12/2024
uid: blazor/components/lifecycle
---
# ASP.NET Core Razor component lifecycle

[!INCLUDE[](~/includes/not-latest-version.md)]

This article explains the ASP.NET Core Razor component lifecycle and how to use lifecycle events.

## Lifecycle events

The Razor component processes Razor component lifecycle events in a set of synchronous and asynchronous lifecycle methods. The lifecycle methods can be overridden to perform additional operations in components during component initialization and rendering.

This article simplifies component lifecycle event processing in order to clarify complex framework logic and doesn't cover every change that was made over the years. You may need to access the [`ComponentBase` reference source](https://github.com/dotnet/aspnetcore/blob/main/src/Components/Components/src/ComponentBase.cs) to integrate custom event processing with Blazor's lifecycle event processing. Code comments in the reference source include additional remarks on lifecycle event processing that don't appear in this article or in the [API documentation](/dotnet/api/).

[!INCLUDE[](~/includes/aspnetcore-repo-ref-source-links.md)]

The following simplified diagrams illustrate Razor component lifecycle event processing. The C# methods associated with the lifecycle events are defined with examples in the following sections of this article.

Component lifecycle events:

1. If the component is rendering for the first time on a request:
   * Create the component's instance.
   * Perform property injection.
   * Call [`OnInitialized{Async}`](#component-initialization-oninitializedasync). If an incomplete <xref:System.Threading.Tasks.Task> is returned, the <xref:System.Threading.Tasks.Task> is awaited and then the component is rerendered. The synchronous method is called prior to the asynchronous method.
1. Call [`OnParametersSet{Async}`](#after-parameters-are-set-onparameterssetasync). If an incomplete <xref:System.Threading.Tasks.Task> is returned, the <xref:System.Threading.Tasks.Task> is awaited and then the component is rerendered. The synchronous method is called prior to the asynchronous method.
1. Render for all synchronous work and complete <xref:System.Threading.Tasks.Task>s.

> [!NOTE]
> Asynchronous actions performed in lifecycle events might delay component rendering or displaying data. For more information, see the [Handle incomplete asynchronous actions at render](#handle-incomplete-asynchronous-actions-at-render) section later in this article.

A parent component renders before its children components because rendering is what determines which children are present. If synchronous parent component initialization is used, the parent initialization is guaranteed to complete first. If asynchronous parent component initialization is used, the completion order of parent and child component initialization can't be determined because it depends on the initialization code running.

![Component lifecycle events of a Razor component in Blazor](~/blazor/components/lifecycle/_static/lifecycle1.png)

DOM event processing:

1. The event handler is run.
1. If an incomplete <xref:System.Threading.Tasks.Task> is returned, the <xref:System.Threading.Tasks.Task> is awaited and then the component is rerendered.
1. Render for all synchronous work and complete <xref:System.Threading.Tasks.Task>s.

![DOM event processing](~/blazor/components/lifecycle/_static/lifecycle2.png)

The `Render` lifecycle:

1. Avoid further rendering operations on the component when both of the following conditions are met:
   * It is not the first render.
   * [`ShouldRender`](xref:blazor/components/rendering#suppress-ui-refreshing-shouldrender) returns `false`.
1. Build the render tree diff (difference) and render the component.
1. Await the DOM to update.
1. Call [`OnAfterRender{Async}`](#after-component-render-onafterrenderasync). The synchronous method is called prior to the asynchronous method.

![Render lifecycle](~/blazor/components/lifecycle/_static/lifecycle3.png)

Developer calls to [`StateHasChanged`](#state-changes-statehaschanged) result in a rerender. For more information, see <xref:blazor/components/rendering#statehaschanged>.

## Quiescence during prerendering

In server-side Blazor apps, prerendering waits for *quiescence*, which means that a component doesn't render until all of the components in the render tree have finished rendering. Quiescence can lead to noticeable delays in rendering when a component performs long-running tasks during initialization and other lifecycle methods, leading to a poor user experience. For more information, see the [Handle incomplete asynchronous actions at render](#handle-incomplete-asynchronous-actions-at-render) section later in this article.

## When parameters are set (`SetParametersAsync`)

<xref:Microsoft.AspNetCore.Components.ComponentBase.SetParametersAsync%2A> sets parameters supplied by the component's parent in the render tree or from route parameters.

The method's <xref:Microsoft.AspNetCore.Components.ParameterView> parameter contains the set of [component parameter](xref:blazor/components/index#component-parameters) values for the component each time <xref:Microsoft.AspNetCore.Components.ComponentBase.SetParametersAsync%2A> is called. By overriding the <xref:Microsoft.AspNetCore.Components.ComponentBase.SetParametersAsync%2A> method, developer code can interact directly with <xref:Microsoft.AspNetCore.Components.ParameterView>'s parameters.

The default implementation of <xref:Microsoft.AspNetCore.Components.ComponentBase.SetParametersAsync%2A> sets the value of each property with the [`[Parameter]`](xref:Microsoft.AspNetCore.Components.ParameterAttribute) or [`[CascadingParameter]` attribute](xref:Microsoft.AspNetCore.Components.CascadingParameterAttribute) that has a corresponding value in the <xref:Microsoft.AspNetCore.Components.ParameterView>. Parameters that don't have a corresponding value in <xref:Microsoft.AspNetCore.Components.ParameterView> are left unchanged.

Generally, your code should call the base class method (`await base.SetParametersAsync(parameters);`) when overriding <xref:Microsoft.AspNetCore.Components.ComponentBase.SetParametersAsync%2A>. In advanced scenarios, developer code can interpret the incoming parameters' values in any way required by not invoking the base class method. For example, there's no requirement to assign the incoming parameters to the properties of the class. However, you must refer to the [`ComponentBase` reference source](https://github.com/dotnet/aspnetcore/blob/main/src/Components/Components/src/ComponentBase.cs) when structuring your code without calling the base class method because it calls other lifecycle methods and triggers rendering in a complex fashion.

[!INCLUDE[](~/includes/aspnetcore-repo-ref-source-links.md)]

If you want to rely on the initialization and rendering logic of <xref:Microsoft.AspNetCore.Components.ComponentBase.SetParametersAsync%2A?displayProperty=nameWithType> but not process incoming parameters, you have the option of passing an empty <xref:Microsoft.AspNetCore.Components.ParameterView> to the base class method:

```csharp
await base.SetParametersAsync(ParameterView.Empty);
```

If event handlers are provided in developer code, unhook them on disposal. For more information, see <xref:blazor/components/component-disposal>.

In the following example, <xref:Microsoft.AspNetCore.Components.ParameterView.TryGetValue%2A?displayProperty=nameWithType> assigns the `Param` parameter's value to `value` if parsing a route parameter for `Param` is successful. When `value` isn't `null`, the value is displayed by the component.

Although [route parameter matching is case insensitive](xref:blazor/fundamentals/routing#route-parameters), <xref:Microsoft.AspNetCore.Components.ParameterView.TryGetValue%2A> only matches case-sensitive parameter names in the route template. The following example requires the use of `/{Param?}` in the route template in order to get the value with <xref:Microsoft.AspNetCore.Components.ParameterView.TryGetValue%2A>, not `/{param?}`. If `/{param?}` is used in this scenario, <xref:Microsoft.AspNetCore.Components.ParameterView.TryGetValue%2A> returns `false` and `message` isn't set to either `message` string.

`SetParamsAsync.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/SetParamsAsync.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/SetParamsAsync.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/lifecycle/SetParamsAsync.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="razor" source="~/../blazor-samples/6.0/BlazorSample_WebAssembly/Pages/lifecycle/SetParamsAsync.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="razor" source="~/../blazor-samples/5.0/BlazorSample_WebAssembly/Pages/lifecycle/SetParamsAsync.razor":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="razor" source="~/../blazor-samples/3.1/BlazorSample_WebAssembly/Pages/lifecycle/SetParamsAsync.razor":::

:::moniker-end

## Component initialization (`OnInitialized{Async}`)

<xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitialized%2A> and <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> are used exclusively to initialize a component for the entire lifetime of the component instance. Parameter values and parameter value changes shouldn't affect the initialization performed in these methods. For example, loading static options into a dropdown list that doesn't change for the lifetime of the component and that isn't dependent on parameter values is performed in one of these lifecycle methods. If parameter values or changes in parameter values affect component state, use [`OnParametersSet{Async}`](#when-parameters-are-set-setparametersasync) instead.

These methods are invoked when the component is initialized after having received its initial parameters in <xref:Microsoft.AspNetCore.Components.ComponentBase.SetParametersAsync%2A>. The synchronous method is called prior to the asynchronous method.

If synchronous parent component initialization is used, the parent initialization is guaranteed to complete before child component initialization. If asynchronous parent component initialization is used, the completion order of parent and child component initialization can't be determined because it depends on the initialization code running.

For a synchronous operation, override <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitialized%2A>:

`OnInit.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/OnInit.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/OnInit.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/lifecycle/OnInit.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="razor" source="~/../blazor-samples/6.0/BlazorSample_WebAssembly/Pages/lifecycle/OnInit.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="razor" source="~/../blazor-samples/5.0/BlazorSample_WebAssembly/Pages/lifecycle/OnInit.razor":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="razor" source="~/../blazor-samples/3.1/BlazorSample_WebAssembly/Pages/lifecycle/OnInit.razor":::

:::moniker-end

To perform an asynchronous operation, override <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> and use the [`await`](/dotnet/csharp/language-reference/operators/await) operator:

```csharp
protected override async Task OnInitializedAsync()
{
    await ...
}
```

If a custom [base class](xref:blazor/components/index#specify-a-base-class) is used with custom initialization logic, call <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> on the base class:

```csharp
protected override async Task OnInitializedAsync()
{
    await ...

    await base.OnInitializedAsync();
}
```

It isn't necessary to call <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A?displayProperty=nameWithType> unless a custom base class is used with custom logic. For more information, see the [Base class lifecycle methods](#base-class-lifecycle-methods) section.

A component must ensure that it's in a valid state for rendering when <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> awaits a potentially incomplete <xref:System.Threading.Tasks.Task>. If the method returns an incomplete <xref:System.Threading.Tasks.Task>, the part of the method that completes synchronously must leave the component in a valid state for rendering. For more information, see the introductory remarks of <xref:blazor/components/sync-context> and <xref:blazor/components/component-disposal>.

Blazor apps that prerender their content on the server call <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> *twice*:

* Once when the component is initially rendered statically as part of the page.
* A second time when the browser renders the component.

:::moniker range=">= aspnetcore-8.0"

To prevent developer code in <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> from running twice when prerendering, see the [Stateful reconnection after prerendering](#stateful-reconnection-after-prerendering) section. The content in the section focuses on Blazor Web Apps and stateful SignalR *reconnection*. To preserve state during the execution of initialization code while prerendering, see <xref:blazor/components/prerender#persist-prerendered-state>.

:::moniker-end

:::moniker range="< aspnetcore-8.0"

To prevent developer code in <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> from running twice when prerendering, see the [Stateful reconnection after prerendering](#stateful-reconnection-after-prerendering) section. Although the content in the section focuses on Blazor Server and stateful SignalR *reconnection*, the scenario for prerendering in hosted Blazor WebAssembly solutions (<xref:Microsoft.AspNetCore.Mvc.Rendering.RenderMode.WebAssemblyPrerendered>) involves similar conditions and approaches to prevent executing developer code twice. To preserve state during the execution of initialization code while prerendering, see <xref:blazor/components/integration#persist-prerendered-state>.

:::moniker-end

While a Blazor app is prerendering, certain actions, such as calling into JavaScript (JS interop), aren't possible. Components may need to render differently when prerendered. For more information, see the [Prerendering with JavaScript interop](#prerendering-with-javascript-interop) section.

If event handlers are provided in developer code, unhook them on disposal. For more information, see <xref:blazor/components/component-disposal>.

::: moniker range=">= aspnetcore-8.0"

Use *streaming rendering* with static server-side rendering (static SSR) or prerendering to improve the user experience for components that perform long-running asynchronous tasks in <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> to fully render. For more information, see the following resources:

* [Handle incomplete asynchronous actions at render](#handle-incomplete-asynchronous-actions-at-render) (this article)
* <xref:blazor/components/rendering#streaming-rendering>

:::moniker-end

## After parameters are set (`OnParametersSet{Async}`)

<xref:Microsoft.AspNetCore.Components.ComponentBase.OnParametersSet%2A> or <xref:Microsoft.AspNetCore.Components.ComponentBase.OnParametersSetAsync%2A> are called:

* After the component is initialized in <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitialized%2A> or <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A>.

* When the parent component rerenders and supplies:

  * Known or primitive immutable types when at least one parameter has changed.
  * Complex-typed parameters. The framework can't know whether the values of a complex-typed parameter have mutated internally, so the framework always treats the parameter set as changed when one or more complex-typed parameters are present.
  
  For more information on rendering conventions, see <xref:blazor/components/rendering#rendering-conventions-for-componentbase>.

The synchronous method is called prior to the asynchronous method.

The methods can be invoked even if the parameter values haven't changed. This behavior underscores the need for developers to implement additional logic within the methods to check whether parameter values have indeed changed before re-initializing data or state dependent on those parameters.

For the following example component, navigate to the component's page at a URL:

* With a start date that's received by `StartDate`: `/on-parameters-set/2021-03-19`
* Without a start date, where `StartDate` is assigned a value of the current local time: `/on-parameters-set`

:::moniker range=">= aspnetcore-5.0"

> [!NOTE]
> In a component route, it isn't possible to both constrain a <xref:System.DateTime> parameter with the [route constraint `datetime`](xref:blazor/fundamentals/routing#route-constraints) and [make the parameter optional](xref:blazor/fundamentals/routing#route-parameters). Therefore, the following `OnParamsSet` component uses two [`@page`](xref:mvc/views/razor#page) directives to handle routing with and without a supplied date segment in the URL.

:::moniker-end

`OnParamsSet.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/OnParamsSet.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/OnParamsSet.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/lifecycle/OnParamsSet.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="razor" source="~/../blazor-samples/6.0/BlazorSample_WebAssembly/Pages/lifecycle/OnParamsSet.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="razor" source="~/../blazor-samples/5.0/BlazorSample_WebAssembly/Pages/lifecycle/OnParamsSet.razor":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="razor" source="~/../blazor-samples/3.1/BlazorSample_WebAssembly/Pages/lifecycle/OnParamsSet.razor":::

:::moniker-end

Asynchronous work when applying parameters and property values must occur during the <xref:Microsoft.AspNetCore.Components.ComponentBase.OnParametersSetAsync%2A> lifecycle event:

```csharp
protected override async Task OnParametersSetAsync()
{
    await ...
}
```

If a custom [base class](xref:blazor/components/index#specify-a-base-class) is used with custom initialization logic, call <xref:Microsoft.AspNetCore.Components.ComponentBase.OnParametersSetAsync%2A> on the base class:

```csharp
protected override async Task OnParametersSetAsync()
{
    await ...

    await base.OnParametersSetAsync();
}
```

It isn't necessary to call <xref:Microsoft.AspNetCore.Components.ComponentBase.OnParametersSetAsync%2A?displayProperty=nameWithType> unless a custom base class is used with custom logic. For more information, see the [Base class lifecycle methods](#base-class-lifecycle-methods) section.

A component must ensure that it's in a valid state for rendering when <xref:Microsoft.AspNetCore.Components.ComponentBase.OnParametersSetAsync%2A> awaits a potentially incomplete <xref:System.Threading.Tasks.Task>. If the method returns an incomplete <xref:System.Threading.Tasks.Task>, the part of the method that completes synchronously must leave the component in a valid state for rendering. For more information, see the introductory remarks of <xref:blazor/components/sync-context> and <xref:blazor/components/component-disposal>.

If event handlers are provided in developer code, unhook them on disposal. For more information, see <xref:blazor/components/component-disposal>.

If a disposable component doesn't use a <xref:System.Threading.CancellationToken>, <xref:Microsoft.AspNetCore.Components.ComponentBase.OnParametersSet%2A> and <xref:Microsoft.AspNetCore.Components.ComponentBase.OnParametersSetAsync%2A> should check if the component is disposed. If <xref:Microsoft.AspNetCore.Components.ComponentBase.OnParametersSetAsync%2A> returns an incomplete <xref:System.Threading.Tasks.Task>, the component must ensure that the part of the method that completes synchronously leaves the component in a valid state for rendering. For more information, see the introductory remarks of <xref:blazor/components/sync-context> and <xref:blazor/components/component-disposal>.

For more information on route parameters and constraints, see <xref:blazor/fundamentals/routing>.

For an example of implementing `SetParametersAsync` manually to improve performance in some scenarios, see <xref:blazor/performance/rendering#implement-setparametersasync-manually>.

## After component render (`OnAfterRender{Async}`)

:::moniker range=">= aspnetcore-8.0"

<xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRender%2A> and <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRenderAsync%2A> are invoked after a component has rendered interactively and the UI has finished updating (for example, after elements are added to the browser DOM). Element and component references are populated at this point. Use this stage to perform additional initialization steps with the rendered content, such as JS interop calls that interact with the rendered DOM elements. The synchronous method is called prior to the asynchronous method.

These methods aren't invoked during prerendering or static server-side rendering (static SSR) on the server because those processes aren't attached to a live browser DOM and are already complete before the DOM is updated.

For <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRenderAsync%2A>, the component doesn't automatically rerender after the completion of any returned `Task` to avoid an infinite render loop.

:::moniker-end

:::moniker range="< aspnetcore-8.0"

<xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRender%2A> and <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRenderAsync%2A> are called after a component has finished rendering. Element and component references are populated at this point. Use this stage to perform additional initialization steps with the rendered content, such as JS interop calls that interact with the rendered DOM elements. The synchronous method is called prior to the asynchronous method.

These methods aren't invoked during prerendering because prerendering isn't attached to a live browser DOM and is already complete before the DOM is updated.

For <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRenderAsync%2A>, the component doesn't automatically rerender after the completion of any returned `Task` to avoid an infinite render loop.

:::moniker-end

The `firstRender` parameter for <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRender%2A> and <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRenderAsync%2A>:

* Is set to `true` the first time that the component instance is rendered.
* Can be used to ensure that initialization work is only performed once.

`AfterRender.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/AfterRender.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/AfterRender.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/lifecycle/AfterRender.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="razor" source="~/../blazor-samples/6.0/BlazorSample_WebAssembly/Pages/lifecycle/AfterRender.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="razor" source="~/../blazor-samples/5.0/BlazorSample_WebAssembly/Pages/lifecycle/AfterRender.razor":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="razor" source="~/../blazor-samples/3.1/BlazorSample_WebAssembly/Pages/lifecycle/AfterRender.razor":::

:::moniker-end

The `AfterRender.razor` sample produces following output to console when the page is loaded and the button is selected:

> :::no-loc text="OnAfterRender: firstRender = True":::  
> :::no-loc text="HandleClick called":::  
> :::no-loc text="OnAfterRender: firstRender = False":::

Asynchronous work immediately after rendering must occur during the <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRenderAsync%2A> lifecycle event:

```csharp
protected override async Task OnAfterRenderAsync(bool firstRender)
{
    ...
}
```

If a custom [base class](xref:blazor/components/index#specify-a-base-class) is used with custom initialization logic, call <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRenderAsync%2A> on the base class:

```csharp
protected override async Task OnAfterRenderAsync(bool firstRender)
{
    ...

    await base.OnAfterRenderAsync(firstRender);
}
```

It isn't necessary to call <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRenderAsync%2A?displayProperty=nameWithType> unless a custom base class is used with custom logic. For more information, see the [Base class lifecycle methods](#base-class-lifecycle-methods) section.

Even if you return a <xref:System.Threading.Tasks.Task> from <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRenderAsync%2A>, the framework doesn't schedule a further render cycle for your component once that task completes. This is to avoid an infinite render loop. This is different from the other lifecycle methods, which schedule a further render cycle once a returned <xref:System.Threading.Tasks.Task> completes.

<xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRender%2A> and <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRenderAsync%2A> *aren't called during the prerendering process on the server*. The methods are called when the component is rendered interactively after prerendering. When the app prerenders:

1. The component executes on the server to produce some static HTML markup in the HTTP response. During this phase, <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRender%2A> and <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRenderAsync%2A> aren't called.
1. When the Blazor script (`blazor.{server|webassembly|web}.js`) starts in the browser, the component is restarted in an interactive rendering mode. After a component is restarted, <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRender%2A> and <xref:Microsoft.AspNetCore.Components.ComponentBase.OnAfterRenderAsync%2A> **are** called because the app isn't in the prerendering phase any longer.

If event handlers are provided in developer code, unhook them on disposal. For more information, see <xref:blazor/components/component-disposal>.

## Base class lifecycle methods

When overriding Blazor's lifecycle methods, it isn't necessary to call base class lifecycle methods for <xref:Microsoft.AspNetCore.Components.ComponentBase>. However, a component should call an overridden base class lifecycle method in the following situations:

* When overriding <xref:Microsoft.AspNetCore.Components.ComponentBase.SetParametersAsync%2A?displayProperty=nameWithType>, `await base.SetParametersAsync(parameters);` is usually invoked because the base class method calls other lifecycle methods and triggers rendering in a complex fashion. For more information, see the [When parameters are set (`SetParametersAsync`)](#when-parameters-are-set-setparametersasync) section.
* If the base class method contains logic that must be executed. Library consumers usually call base class lifecycle methods when inheriting a base class because library base classes often have custom lifecycle logic to execute. If the app uses a base class from a library, consult the library's documentation for guidance.

In the following example, `base.OnInitialized();` is called to ensure that the base class's `OnInitialized` method is executed. Without the call, `BlazorRocksBase2.OnInitialized` doesn't execute.

`BlazorRocks2.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/BlazorRocks2.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/BlazorRocks2.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/index/BlazorRocks2.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="razor" source="~/../blazor-samples/6.0/BlazorSample_WebAssembly/Pages/index/BlazorRocks2.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="razor" source="~/../blazor-samples/5.0/BlazorSample_WebAssembly/Pages/index/BlazorRocks2.razor":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="razor" source="~/../blazor-samples/3.1/BlazorSample_WebAssembly/Pages/index/BlazorRocks2.razor":::

:::moniker-end

`BlazorRocksBase2.cs`:

:::moniker range=">= aspnetcore-9.0"

:::code language="csharp" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/BlazorRocksBase2.cs":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="csharp" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/BlazorRocksBase2.cs":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="csharp" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/BlazorRocksBase2.cs":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="csharp" source="~/../blazor-samples/6.0/BlazorSample_WebAssembly/BlazorRocksBase2.cs":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="csharp" source="~/../blazor-samples/5.0/BlazorSample_WebAssembly/BlazorRocksBase2.cs":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="csharp" source="~/../blazor-samples/3.1/BlazorSample_WebAssembly/BlazorRocksBase2.cs":::

:::moniker-end

## State changes (`StateHasChanged`)

<xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> notifies the component that its state has changed. When applicable, calling <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> enqueues a rerender that occurs when the app's main thread is free.

<xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A> is called automatically for <xref:Microsoft.AspNetCore.Components.EventCallback> methods. For more information on event callbacks, see <xref:blazor/components/event-handling#eventcallback>.

For more information on component rendering and when to call <xref:Microsoft.AspNetCore.Components.ComponentBase.StateHasChanged%2A>, including when to invoke it with <xref:Microsoft.AspNetCore.Components.ComponentBase.InvokeAsync%2A?displayProperty=nameWithType>, see <xref:blazor/components/rendering>.

## Handle incomplete asynchronous actions at render

Asynchronous actions performed in lifecycle events might not have completed before the component is rendered. Objects might be `null` or incompletely populated with data while the lifecycle method is executing. Provide rendering logic to confirm that objects are initialized. Render placeholder UI elements (for example, a loading message) while objects are `null`.

In the following `Slow` component, <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> is overridden to asynchronously execute a long-running task. While `isLoading` is `true`, a loading message is displayed to the user. After the `Task` returned by <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> completes, the component is rerendered with the updated state, showing the "`Finished!`" message.

`Slow.razor`:

```razor
@page "/slow"

<h2>Slow Component</h2>

@if (isLoading)
{
    <div><em>Loading...</em></div>
}
else
{
    <div>Finished!</div>
}

@code {
    private bool isLoading = true;

    protected override async Task OnInitializedAsync()
    {
        await LoadDataAsync();
        isLoading = false;
    }

    private Task LoadDataAsync()
    {
        return Task.Delay(10000);
    }
}
```

The preceding component uses an `isLoading` variable to display the loading message. A similar approach is used for a component that loads data into a collection and checks if the collection is `null` to present the loading message. The following example checks the `movies` collection for `null` to either display the loading message or display the collection of movies:

```razor
@if (movies == null)
{
    <p><em>Loading...</em></p>
}
else
{
    @* display movies *@
}

@code {
    private Movies[]? movies;

    protected override async Task OnInitializedAsync()
    {
        movies = await GetMovies();
    }
}
```

Prerendering waits for *quiescence*, which means that a component doesn't render until all of the components in the render tree have finished rendering. This means that a loading message doesn't display while a child component's <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> method is executing a long-running task during prerendering. To demonstrate this behavior, place the preceding `Slow` component into a test app's `Home` component:

```razor
@page "/"

<PageTitle>Home</PageTitle>

<h1>Hello, world!</h1>

Welcome to your new app.

<Slow />
```

> [!NOTE]
> Although the examples in this section discuss the <xref:Microsoft.AspNetCore.Components.ComponentBase.OnInitializedAsync%2A> lifecycle method, other lifecycle methods that execute during prerendering may delay final rendering of a component. Only [`OnAfterRender{Async}`](#after-component-render-onafterrenderasync) isn't executed during prerendering and is immune to delays due to quiescence.

During prerendering, the `Home` component doesn't render until the `Slow` component is rendered, which takes ten seconds. The UI is blank during this ten-second period, and there's no loading message. After prerendering, the `Home` component renders, and the `Slow` component's loading message is displayed. After ten more seconds, the `Slow` component finally displays the finished message.

:::moniker range=">= aspnetcore-8.0"

As the preceding demonstration illustrates, quiescence during prerendering results in a poor user experience. To improve the user experience, begin by implementing [streaming rendering](xref:blazor/components/rendering#streaming-rendering) to avoid waiting for the asynchronous task to complete while prerendering.

Add the [`[StreamRendering]` attribute](xref:Microsoft.AspNetCore.Components.StreamRenderingAttribute) to the `Slow` component (use `[StreamRendering(true)]` in .NET 8):

```razor
@attribute [StreamRendering]
```

When the `Home` component is prerendering, the `Slow` component is quickly rendered with its loading message. The `Home` component doesn't wait for ten seconds for the `Slow` component to finish rendering. However, the finished message displayed at the end of prerendering is replaced by the loading message while the component finally renders, which is another ten-second delay. This occurs because the `Slow` component is rendering twice, along with `LoadDataAsync` executing twice. When a component is accessing resources, such as services and databases, double execution of service calls and database queries creates undesirable load on the app's resources.

To address the double rendering of the loading message and the re-execution of service and database calls, persist prerendered state with <xref:Microsoft.AspNetCore.Components.PersistentComponentState> for final rendering of the component, as seen in the following updates to the `Slow` component:

:::moniker-end

:::moniker range=">= aspnetcore-10.0"

```razor
@page "/slow"
@attribute [StreamRendering]

<h2>Slow Component</h2>

@if (Data is null)
{
    <div><em>Loading...</em></div>
}
else
{
    <div>@Data</div>
}

@code {
    [SupplyParameterFromPersistentComponentState]
    public string? Data { get; set; }

    protected override async Task OnInitializedAsync()
    {
        Data ??= await LoadDataAsync();
    }

    private async Task<string> LoadDataAsync()
    {
        await Task.Delay(5000);
        return "Finished!";
    }
}
```

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-10.0"

```razor
@page "/slow"
@attribute [StreamRendering]
@implements IDisposable
@inject PersistentComponentState ApplicationState

<h2>Slow Component</h2>

@if (data is null)
{
    <div><em>Loading...</em></div>
}
else
{
    <div>@data</div>
}

@code {
    private string? data;
    private PersistingComponentStateSubscription persistingSubscription;

    protected override async Task OnInitializedAsync()
    {
        if (!ApplicationState.TryTakeFromJson<string>(nameof(data), out var restored))
        {
            data = await LoadDataAsync();
        }
        else
        {
            data = restored!;
        }

        // Call at the end to avoid a potential race condition at app shutdown
        persistingSubscription = ApplicationState.RegisterOnPersisting(PersistData);
    }

    private Task PersistData()
    {
        ApplicationState.PersistAsJson(nameof(data), data);

        return Task.CompletedTask;
    }

    private async Task<string> LoadDataAsync()
    {
        await Task.Delay(5000);
        return "Finished!";
    }

    void IDisposable.Dispose()
    {
        persistingSubscription.Dispose();
    }
}
```

:::moniker-end

By combining streaming rendering with persistent component state:

* Services and databases only require a single call for component initialization.
* Components render their UIs quickly with loading messages during long-running tasks for the best user experience.

For more information, see the following resources:

* <xref:blazor/components/rendering#streaming-rendering>
* <xref:blazor/components/prerender#persist-prerendered-state>.

:::moniker-end

:::moniker range="< aspnetcore-8.0"

Quiescence during prerendering results in a poor user experience. The delay can be addressed in apps that target .NET 8 or later with a feature called *streaming rendering*, usually combined with [persisting component state during prerendering](xref:blazor/components/integration#persist-prerendered-state) to avoid waiting for the asynchronous task to complete. In versions of .NET earlier than .NET 8, executing a [long-running background task](#cancelable-background-work) that loads the data after final rendering can address a long rendering delay due to quiescence.

:::moniker-end

## Handle errors

For information on handling errors during lifecycle method execution, see <xref:blazor/fundamentals/handle-errors>.

## Stateful reconnection after prerendering

When prerendering on the server, a component is initially rendered statically as part of the page. Once the browser establishes a SignalR connection back to the server, the component is rendered *again* and interactive. If the [`OnInitialized{Async}`](#component-initialization-oninitializedasync) lifecycle method for initializing the component is present, the method is executed *twice*:

* When the component is prerendered statically.
* After the server connection has been established.

This can result in a noticeable change in the data displayed in the UI when the component is finally rendered. To avoid this behavior, pass in an identifier to cache the state during prerendering and to retrieve the state after prerendering.

The following code demonstrates a `WeatherForecastService` that avoids the change in data display due to prerendering. The awaited <xref:System.Threading.Tasks.Task.Delay%2A> (`await Task.Delay(...)`) simulates a short delay before returning data from the `GetForecastAsync` method.

Add <xref:Microsoft.Extensions.Caching.Memory.IMemoryCache> services with <xref:Microsoft.Extensions.DependencyInjection.MemoryCacheServiceCollectionExtensions.AddMemoryCache%2A> on the service collection in the app's `Program` file:

```csharp
builder.Services.AddMemoryCache();
```

`WeatherForecastService.cs`:

:::moniker range=">= aspnetcore-9.0"

:::code language="csharp" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/WeatherForecastService.cs":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="csharp" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/WeatherForecastService.cs":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="csharp" source="~/../blazor-samples/7.0/BlazorSample_Server/lifecycle/WeatherForecastService.cs":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="csharp" source="~/../blazor-samples/6.0/BlazorSample_Server/lifecycle/WeatherForecastService.cs":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="csharp" source="~/../blazor-samples/5.0/BlazorSample_Server/lifecycle/WeatherForecastService.cs":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="csharp" source="~/../blazor-samples/3.1/BlazorSample_Server/lifecycle/WeatherForecastService.cs":::

:::moniker-end

For more information on the <xref:Microsoft.AspNetCore.Mvc.TagHelpers.ComponentTagHelper.RenderMode>, see <xref:blazor/fundamentals/signalr#server-side-render-mode>.

:::moniker range=">= aspnetcore-8.0"

The content in this section focuses on Blazor Web Apps and stateful SignalR *reconnection*. To preserve state during the execution of initialization code while prerendering, see <xref:blazor/components/prerender#persist-prerendered-state>.

:::moniker-end

:::moniker range="< aspnetcore-8.0"

Although the content in this section focuses on Blazor Server and stateful SignalR *reconnection*, the scenario for prerendering in hosted Blazor WebAssembly solutions (<xref:Microsoft.AspNetCore.Mvc.Rendering.RenderMode.WebAssemblyPrerendered>) involves similar conditions and approaches to prevent executing developer code twice. To preserve state during the execution of initialization code while prerendering, see <xref:blazor/components/integration#persist-prerendered-state>.

:::moniker-end

## Prerendering with JavaScript interop

[!INCLUDE[](~/blazor/includes/prerendering.md)]

## Cancelable background work

Components often perform long-running background work, such as making network calls (<xref:System.Net.Http.HttpClient>) and interacting with databases. It's desirable to stop the background work to conserve system resources in several situations. For example, background asynchronous operations don't automatically stop when a user navigates away from a component.

Other reasons why background work items might require cancellation include:

* An executing background task was started with faulty input data or processing parameters.
* The current set of executing background work items must be replaced with a new set of work items.
* The priority of currently executing tasks must be changed.
* The app must be shut down for server redeployment.
* Server resources become limited, necessitating the rescheduling of background work items.

To implement a cancelable background work pattern in a component:

* Use a <xref:System.Threading.CancellationTokenSource> and <xref:System.Threading.CancellationToken>.
* Upon [disposal of the component](xref:blazor/components/component-disposal) and at any point cancellation is desired by manually canceling the token, call [`CancellationTokenSource.Cancel`](xref:System.Threading.CancellationTokenSource.Cancel%2A) to signal that the background work should be cancelled.
* After the asynchronous call returns, call <xref:System.Threading.CancellationToken.ThrowIfCancellationRequested%2A> on the token.

In the following example:

* `await Task.Delay(10000, cts.Token);` represents long-running asynchronous background work.
* `BackgroundResourceMethod` represents a long-running background method that shouldn't start if the `Resource` is disposed before the method is called.

`BackgroundWork.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/BackgroundWork.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/BackgroundWork.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/lifecycle/BackgroundWork.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="razor" source="~/../blazor-samples/6.0/BlazorSample_WebAssembly/Pages/lifecycle/BackgroundWork.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="razor" source="~/../blazor-samples/5.0/BlazorSample_WebAssembly/Pages/lifecycle/BackgroundWork.razor":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="razor" source="~/../blazor-samples/3.1/BlazorSample_WebAssembly/Pages/lifecycle/BackgroundWork.razor":::

:::moniker-end

To display a loading indicator while the background work is taking place, use the following approach.

Create a loading indicator component with a `Loading` parameter that can display child content in a <xref:Microsoft.AspNetCore.Components.RenderFragment>. For the `Loading` parameter:

* When `true`, display a loading indicator.
* When `false`, render the component's content (`ChildContent`). For more information, see [Child content render fragments](xref:blazor/components/index#child-content-render-fragments).

`ContentLoading.razor`:

```razor
@if (Loading)
{
    <progress id="loadingIndicator" aria-label="Content loading…"></progress>
}
else
{
    @ChildContent
}

@code {
    [Parameter]
    public RenderFragment? ChildContent { get; set; }

    [Parameter]
    public bool Loading { get; set; }
}
```

:::moniker range=">= aspnetcore-6.0"

To load CSS styles for the indicator, add the styles to `<head>` content with the <xref:Microsoft.AspNetCore.Components.Web.HeadContent> component. For more information, see <xref:blazor/components/control-head-content>.

```razor
@if (Loading)
{
    <!-- OPTIONAL ...
    <HeadContent>
        <style>
            ...
        </style>
    </HeadContent>
    -->
    <progress id="loadingIndicator" aria-label="Content loading…"></progress>
}
else
{
    @ChildContent
}

...
```

:::moniker-end

Wrap the component's Razor markup with the `ContentLoading` component and pass a value in a C# field to the `Loading` parameter when initialization work is performed by the component:

```razor
<ContentLoading Loading="@loading">
    ...
</ContentLoading>

@code {
    private bool loading = true;
    ...

    protected override async Task OnInitializedAsync()
    {
        await LongRunningWork().ContinueWith(_ => loading = false);
    }

    ...
}
```

## Blazor Server reconnection events

The component lifecycle events covered in this article operate separately from [server-side reconnection event handlers](xref:blazor/fundamentals/signalr#reflect-the-server-side-connection-state-in-the-ui). When the SignalR connection to the client is lost, only UI updates are interrupted. UI updates are resumed when the connection is re-established. For more information on circuit handler events and configuration, see <xref:blazor/fundamentals/signalr>.

## Additional resources

[Handle caught exceptions outside of a Razor component's lifecycle](xref:blazor/components/sync-context#handle-caught-exceptions-outside-of-a-razor-components-lifecycle)
