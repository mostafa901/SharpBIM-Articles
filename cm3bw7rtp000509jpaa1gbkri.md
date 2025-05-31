---
title: "Using AppDomains to Resolve DLL Conflicts in Revit Plugins"
datePublished: Sun Nov 10 2024 17:53:15 GMT+0000 (Coordinated Universal Time)
cuid: cm3bw7rtp000509jpaa1gbkri
slug: using-appdomains-to-resolve-dll-conflicts-in-revit-plugins
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1731258606409/05533a65-08f2-409d-8469-07f5dcbde0c3.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1731258680539/e37b618a-47cb-4048-a38c-612804492788.png
tags: dependency-management, revit-architecture-plugins, revit-development, appdomain-isolation, dll-conflict-resolution, appdomain-in-net, ui-library-conflicts, dependency-isolation

---

DLL Hell—a persistent challenge for .NET Framework developers—becomes particularly thorny when creating add-ins for complex applications like Revit. Juggling dependency versions often leads to cryptic conflicts, as each DLL expects a specific version to be present. I recently faced this issue while integrating `Auth0.Client` into a custom login process for a Revit 2019 add-in. Revit's preloaded dependencies clashed with those required by `Auth0.Client`, presenting an intriguing challenge: how to make these libraries cooperate without compromising Revit's core functionality.

While not identical to other DLL conflicts discussed on the Autodesk forums [here](https://forums.autodesk.com/t5/revit-api-forum/proper-way-to-handle-app-config-bindingredirects-in-revit-add-in/m-p/5692149) and [here](https://forums.autodesk.com/t5/revit-api-forum/old-restsharp-version-in-revit-2025/m-p/12974216), this issue shares similar principles. Here's my journey from initial stumbling blocks to an elegant AppDomain-based solution.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1731257322918/1c0274cf-cb3d-4a07-8f9d-cc4991d0ad9c.gif align="center")

### The Problem: Conflicting Versions of `System.Runtime.CompilerServices.Unsafe.dll`

In this particular case, the `Auth0.Client` library required `System.Runtime.CompilerServices.Unsafe.dll` version 6, but Revit 2019 was locked into version 4. Revit preloads its version at runtime, which led to a missing method exception when `Auth0.Client` tried to call a method only available in the newer version. Classic DLL Hell.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1731259992384/a54b768f-48eb-438b-84f6-8db93d26136b.png align="center")

### Standard Solutions (That Didn’t Work)

Initially, I tried several conventional workarounds:

1. `AppDomain.AssemblyResolve` Event: This is often the first recourse in resolving assembly issues, but it fell short. Revit’s own assembly binding order took precedence, loading version 4 of the DLL before any alternative could be set.
    
2. **Forced Load Order**: I tried forcing the correct DLL version to load before Revit’s, by simply adding `Assembly.Load` at the `OnStart` overload function, but this proved futile since Revit wouldn’t relinquish its hold on its preferred version.
    
3. **Config File Binding Redirects**: Editing `Revit.config` and using my Dll application config file `SharpBIMAddin.dll.config` with binding redirects was another attempt, but Revit’s built-in DLL handling prevented this from working as intended.
    

It quickly became clear that a new approach was needed—one that would allow the `Auth0.Client` library to operate within its own ecosystem, without impacting Revit’s core environment.

### The Solution: Isolated Execution with AppDomains

Metaphorically, using AppDomain in Revit is like placing a fish (the conflicting DLL) in a water-filled, transparent sack (the isolated AppDomain) inside a larger aquarium (the Default AppDomain). The fish can swim in its own water without disturbing the rest of the tank, just as the isolated AppDomain lets the DLL operate with its specific version requirements without impacting Revit’s internal dependencies. This "sack in an aquarium" approach preserves harmony in the larger environment while meeting the unique needs of each element.

![AI Generated](https://cdn.hashnode.com/res/hashnode/image/upload/v1731253238598/f48bf90b-5ab1-4622-a2ff-2114c52946aa.jpeg align="center")

`AI Generated Image`

After trying conventional methods without success, I found that crteating an isolated `AppDomain` was the most effective solution. This separate domain loads and manages `Auth0.Client` and its dependencies independently, bypassing Revit’s locked-in version requirements. Below is a breakdown of the implementation, which turned a major compatibility roadblock into an elegant, sustainable solution.

```csharp
var authService = UtilityHelper.LoadAssemblyViaAppDomain<AuthSandbox>(dllPath);
authService.LoadAssembly(dllPath);
string result = "";
try
{
    result = authService.ExecuteLogin();
    authService.Dispose();
}
catch (Exception)
{
// something went wrong
}
```

### Step 1: Encapsulating Login Logic in a Separate DLL

I placed all login-related code in a standalone library that would handle only the authentication process, outputting a JSON-formatted result. This approach ensured a clear separation between Revit’s operations and the login functionality, making it easier to manage dependencies.

### Step 2: Creating and Configuring a New AppDomain

By configuring a new `AppDomain`, I could load only the specific DLL versions needed for `Auth0.Client`, without interference from Revit’s preloaded dependencies. The following snippet shows how the new `AppDomain` is set up:

```csharp
public T LoadAssemblyViaAppDomain<T>(string targetDll) where T : AssemblySandbox
{
    var domSetup = new AppDomainSetup
    {
        ShadowCopyFiles = "ShadowCopied",
        ShadowCopyDirectories = "ShadowDires",
        CachePath = Path.GetTempPath(),
        ApplicationBase = Path.GetDirectoryName(targetDll)
    };

    AppDomain domain = AppDomain.CreateDomain("IsolatedDomain", null, domSetup);
    Type assemblySandboxType = typeof(T);
    var assemblySandbox = (T)domain.CreateInstanceAndUnwrap(
        assemblySandboxType.Assembly.FullName, assemblySandboxType.ToString());
    domain.DomainUnload += assemblySandbox.Domain_DomainUnload;

    assemblySandbox.Domain = domain;
    return assemblySandbox;
}
```

This setup utilizes `AppDomainSetup` with shadow copying enabled, making it possible to isolate DLLs effectively. The `AppDomain.CreateDomain` method dynamically creates a unique domain, allowing Revit to maintain its ecosystem while `Auth0.Client` manages its own.

### Step 3: Implementing a Sandbox Class

The next step involved creating a sandbox class inheriting from `MarshalByRefObject`. This class enables safe, cross-domain communication. Here’s how I structured it:

```csharp
public abstract class AssemblySandbox : MarshalByRefObject
{
    protected AppDomain Domain { get; private set; }

    public AssemblySandbox() { }

    public void SetDomain(AppDomain domain)
    {
        if (Domain == null && domain != null)
        {
            Domain = domain;
            Resolver = new AssemblyResolver(Domain);
            Domain.DomainUnload += Domain_DomainUnload;
        }
    }
    public abstract void LoadAssembly(string assemblyPath);

    ....
}
```

The `AssemblySandbox` class encapsulates methods for assembly loading and cleanup, keeping the isolated domain’s lifecycle under tight control.

### Step 4: Loading and Using the Login Library in the New Domain

With the isolated domain and sandbox class in place, the final step was to load the login library, call the authentication method, and retrieve the result. Here’s how it all comes together:

```csharp
public class AuthSandbox : AssemblySandbox
{
    public string ExecuteLogin()
    {
        CancelToken = new CancellationTokenSource();
        var result = _authService.Login(CancelToken.Token);
        result.Wait(60000, CancelToken.Token);
        CancelToken.Cancel();
        var userResult = result.Result;
        result.Dispose();
        return userResult;
    }
    
    public override void LoadAssembly(string assemblyPath)
    {
        var assembly = Domain.Load(File.ReadAllBytes(assemblyPath));
        var type = assembly.GetType("SharpBIM.AuthLogin.GoogleAuth");
        _authService = (IAuthService)Activator.CreateInstance(type);
    }
}
```

This `AuthSandbox` class loads the specific login assembly in isolation and invokes the login method. Note the use of `Activator.CreateInstance` to create an instance of the `LoginClass` within the isolated domain, allowing for seamless operation within a self-contained environment.

### Bridging the AppDomain Communication

One challenge with `AppDomains` is that objects crossing domain boundaries must either be serializable or inherit from `MarshalByRefObject`. To keep data exchange simple, I formatted outputs like login results in JSON, allowing Revit to receive results without needing direct access to the Auth0 library.

### Code Snippets from SharpBIM AppDomain Sample

The [SharpBIM.AppDomainSample repository](https://github.com/mostafa901/SharpBIM.AppDomainSample) demonstrates how to use **AppDomain** to resolve DLL conflicts in Revit plugins. By creating an isolated AppDomain, the sample shows how developers can load specific DLL versions independently of Revit’s locked dependencies, such as with `System.Runtime.CompilerServices.Unsafe.dll`. Key features include setting custom assembly paths, dynamic loading of assemblies within the isolated domain, and using `MarshalByRefObject` for cross-domain data communication. This approach helps ensure compatibility and stability, making it a practical solution for Revit plugin development.

Explaining each part of the code would make this article much longer than it already is. However, I'm happy to help explain anything if you need it. Just let me know.

### Lessons and Takeaways

This approach, while requiring a few extra steps, effectively sidestepped Revit’s restrictive DLL handling. The use of AppDomains enabled me to work around dependency conflicts and allowed the `Auth0.Client` library to operate as intended without clashing with Revit’s preloaded assemblies. While AppDomains are unique to the .NET Framework and may not apply to .NET Core or .NET 5+, they remain a powerful tool for handling dependency isolation in legacy environments.

One important consideration is handling conflicts with UI libraries. To achieve this, we load all UI components in a separate project within the isolated AppDomain. Once the UI operations are complete, we seamlessly switch back to the Revit context to carry out necessary updates or actions. This approach allows us to alternate between Revit’s domain and the custom AppDomain as needed, enabling us to execute additional methods in isolation without disrupting Revit’s environment.

In summary, isolating dependencies within a separate AppDomain allows developers to create flexible, dependency-resilient add-ins for applications like Revit, which don’t natively support modern dependency management. For additional insights into similar dependency issues, check out discussions on the Autodesk forum, like [this one](https://forums.autodesk.com/t5/revit-api-forum/proper-way-to-handle-app-config-bindingredirects-in-revit-add-in/m-p/5692149) and [this one](https://forums.autodesk.com/t5/revit-api-forum/old-restsharp-version-in-revit-2025/m-p/12974216).

As Revit development continues to evolve, approaches like this can provide robust solutions for dealing with DLL conflicts, giving developers the freedom to integrate modern libraries with minimal interference. Keep an eye on this space for more in-depth explorations and code samples soon!