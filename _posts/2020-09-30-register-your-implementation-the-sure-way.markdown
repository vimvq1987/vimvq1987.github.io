---
layout: post
title: "Episerver Custom Implementation: The Sure Way to Register Your Overrides 🛠️"
date: 2020-09-30 10:00:00 +0200
categories: [CMS, Commerce, Episerver]
tags: [episerver, dependency-injection, di, override, configurablemodule, programming]
author: vimvq1987
---

The point of **Episerver's Dependency Injection (DI)** is that you can plug in your custom implementation for almost everything. But it can be tricky at times to properly register your custom implementation to ensure it's the one that gets used.

The default DI framework (and most popular ones) works by letting the implementation registered **later** win, which means it overrides any other implementation registered before it. To make Episerver use your custom implementation, you have to make sure yours is **registered last**.

## 🛑 Never Use `[ServiceConfiguration]` for Overrides

Do not register your custom implementation using the `[ServiceConfiguration]` attribute. Implementations with that attribute will be registered **first** in the initialization pipeline, leading to two problems:

1.  **Default Overrides You:** The default implementation may be registered in an `IConfigurableModule.ConfigureContainer` method, which runs **later** than `[ServiceConfiguration]`. Yours will be overridden by the default.
2.  **Indeterministic Order:** If the default implementation is **also** registered using `[ServiceConfiguration]`, you run into an indeterministic situation. The order is randomized every time your website starts. Sometimes yours wins, sometimes the default does, which can cause a nasty bug (**Heisenbug**, if you know the reference 😉).

## 🧐 The Limitations of `IConfigurableModule.ConfigureContainer`

That leaves you with registering your implementation by **`IConfigurableModule.ConfigureContainer`**.

In many cases, registering your implementations here will just work, because the default implementations are registered by the `[ServiceConfiguration]` attribute. However, that is **not always the case**. There is a possibility that the default one was also registered using `IConfigurableModule.ConfigureContainer`, and things get tricky:

* The order in which `IConfigurableModule.ConfigureContainer` is executed is **not determined**.
* You cannot easily make your module depend on a specific module to control the order (unlike with `IInitializationModule`).
* Even if you could, it's not always clear which module you should depend on, and in many cases, that module is `internal`, making it inaccessible.

## ✅ The Guaranteed Way: `ConfigurationComplete` Event

This is the point of this post: To make sure your implementation is registered regardless of how the default one is registered, you can always fall back to using the **`ConfigurationComplete` event** of `ServiceConfigurationContext`.

This event is called **once all `ConfigureContainer` methods have been called**. You can be sure that the default implementation is already registered—now it's time to override it!

```csharp
public void ConfigureContainer(ServiceConfigurationContext context)
{
    // Register to the event that fires after all ConfigureContainer methods
    context.ConfigurationComplete += Context_ConfigurationComplete;
}

private void Context_ConfigurationComplete(object sender, ServiceConfigurationEventArgs e)
{
    // This registration is guaranteed to be the last one, overriding all others.
    e.Services.AddSingleton<IOrderRepository, CustomOrderRepository>();
}
```

Simple as that!

Note: This technique only applies to cases when you want to override the default implementation. If you register an implementation of your own interfaces/abstract classes, or you are merely adding your implementation (not overriding the default one, e.g., an implementation of IShippingPlugin), you can register it in any way you prefer.