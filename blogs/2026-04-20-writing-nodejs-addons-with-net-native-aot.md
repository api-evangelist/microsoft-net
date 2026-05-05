---
title: "Writing Node.js addons with .NET Native AOT"
url: "https://devblogs.microsoft.com/dotnet/writing-nodejs-addons-with-dotnet-native-aot/"
date: "Mon, 20 Apr 2026 20:00:00 +0000"
author: "Drew Noakes"
feed_url: "https://devblogs.microsoft.com/dotnet/feed/"
---
<p><a href="https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit">C# Dev Kit</a> is a VS Code extension. Like all VS Code extensions, its front end is TypeScript running in Node.js. For certain platform-specific tasks, such as reading the Windows Registry, we&#8217;ve historically used native Node.js addons written in C++, which are compiled via <a href="https://github.com/nodejs/node-gyp">node-gyp</a> during installation to the developer&#8217;s workspace.</p>
<p>This works, but it comes with overhead. Using node-gyp to build these particular packages requires an old version of Python to be installed on every developer&#8217;s machine. For a team that works on .NET tooling, this requirement added complexity and friction. New contributors had to set up tools they&#8217;d never touch directly, and CI pipelines needed to provision and maintain them, which slowed down builds and added yet another set of dependencies to keep up to date over time.</p>
<p>The C# Dev Kit team already has the .NET SDK installed, so why not use C# and Native AOT to streamline our engineering systems?</p>
<h2>How Node.js addons work</h2>
<p>A Node.js native addon is a shared library (<code>.dll</code> on Windows, <code>.so</code> on Linux, <code>.dylib</code> on macOS) that exports a specific entry point. When Node.js loads such a library, it calls the function <code>napi_register_module_v1</code>. The addon registers any functions it provides, and from that point on, JavaScript treats it like any other module.</p>
<p>The interface that makes this possible is <a href="https://nodejs.org/api/n-api.html">N-API</a> (also called Node-API) &#8211; a stable, ABI-compatible C API for building addons. N-API doesn&#8217;t care what language produced the shared library, only that it exports the right symbols and calls the right functions. This makes Native AOT a viable option because it can produce shared libraries with arbitrary native entry points, which is all N-API needs.</p>
<p>Throughout the rest of this post, let&#8217;s look at the key parts of a small Native AOT Node.js addon that can read a string value from the registry. To keep things simple, we&#8217;ll put all the code in one class, though you could easily factor things out to be reusable.</p>
<h2>The project file</h2>
<p>The project file is minimal:</p>
<pre><code class="language-xml">&lt;Project Sdk="Microsoft.NET.Sdk"&gt;
  &lt;PropertyGroup&gt;
    &lt;TargetFramework&gt;net10.0&lt;/TargetFramework&gt;
    &lt;PublishAot&gt;true&lt;/PublishAot&gt;
    &lt;AllowUnsafeBlocks&gt;true&lt;/AllowUnsafeBlocks&gt;
  &lt;/PropertyGroup&gt;
&lt;/Project&gt;</code></pre>
<p><a href="https://learn.microsoft.com/dotnet/core/deploying/native-aot/"><code>PublishAot</code></a> tells the SDK to produce a shared library when the project is published. <code>AllowUnsafeBlocks</code> is needed because the N-API interop involves function pointers and fixed buffers.</p>
<h2>The module entry point</h2>
<p>Node.js expects the shared library to export <code>napi_register_module_v1</code>. In C#, we can do this with <a href="https://learn.microsoft.com/dotnet/api/system.runtime.interopservices.unmanagedcallersonlyattribute"><code>[UnmanagedCallersOnly]</code></a>:</p>
<pre><code class="language-csharp">public static unsafe partial class RegistryAddon
{
    [UnmanagedCallersOnly(
        EntryPoint = "napi_register_module_v1",
        CallConvs = [typeof(CallConvCdecl)])]
    public static nint Init(nint env, nint exports)
    {
        Initialize();

        RegisterFunction(
            env,
            exports,
            "readStringValue"u8,
            &amp;ReadStringValue);

        // Register additional functions...

        return exports;
    }
}</code></pre>
<p>A few C# features are doing work here. <a href="https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/nint-nuint"><code>nint</code></a> is a native-sized integer — the managed equivalent of <code>intptr_t</code> &#8211; used to pass around N-API handles. The <a href="https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/reference-types#utf-8-string-literals"><code>u8</code> suffix</a> produces a <code>ReadOnlySpan&lt;byte&gt;</code> containing a UTF-8 string literal, which we pass directly to N-API without any encoding or allocation. And <code>[UnmanagedCallersOnly]</code> tells the AOT compiler to export the method with the specified entry point name and calling convention, making it callable from native code.</p>
<p>Each call to <code>RegisterFunction</code> attaches a C# function pointer to a named property on the JavaScript <code>exports</code> object, so that calling <code>addon.readStringValue(...)</code> in JavaScript invokes the corresponding C# method directly, in-process.</p>
<h2>Calling N-API from .NET</h2>
<p>N-API functions are exported by <code>node.exe</code> itself, so rather than linking against a separate library, we need to resolve them against the host process. We declare our P/Invoke methods using <a href="https://learn.microsoft.com/dotnet/standard/native-interop/pinvoke-source-generation"><code>[LibraryImport]</code></a> with <code>"node"</code> as the library name, and then register a custom resolver via <a href="https://learn.microsoft.com/dotnet/api/system.runtime.interopservices.nativelibrary.setdllimportresolver"><code>NativeLibrary.SetDllImportResolver</code></a> that redirects to the host process at runtime:</p>
<pre><code class="language-csharp">private static void Initialize()
{
    NativeLibrary.SetDllImportResolver(
        System.Reflection.Assembly.GetExecutingAssembly(),
        ResolveDllImport);

    static nint ResolveDllImport(
        string libraryName,
        Assembly assembly,
        DllImportSearchPath? searchPath)
    {
        if (libraryName is not "node")
            return 0;

        return NativeLibrary.GetMainProgramHandle();
    }
}</code></pre>
<p>With this resolver in place, the runtime knows to look up all <code>"node"</code> imports from the host process, and the N-API P/Invoke declarations work without any additional configuration:</p>
<pre><code class="language-csharp">private static partial class NativeMethods
{
    [LibraryImport("node", EntryPoint = "napi_create_string_utf8")]
    internal static partial Status CreateStringUtf8(
        nint env, ReadOnlySpan&lt;byte&gt; str, nuint length, out nint result);

    [LibraryImport("node", EntryPoint = "napi_create_function")]
    internal static unsafe partial Status CreateFunction(
        nint env, ReadOnlySpan&lt;byte&gt; utf8name, nuint length,
        delegate* unmanaged[Cdecl]&lt;nint, nint, nint&gt; cb,
        nint data, out nint result);

    [LibraryImport("node", EntryPoint = "napi_get_cb_info")]
    internal static unsafe partial Status GetCallbackInfo(
        nint env, nint cbinfo, ref nuint argc,
        Span&lt;nint&gt; argv, nint* thisArg, nint* data);

    // ... other N-API functions as needed
}</code></pre>
<p>For each registered function we must register a native function as a named property on the exports object:</p>
<pre><code class="language-csharp">private static unsafe void RegisterFunction(
    nint env, nint exports, ReadOnlySpan&lt;byte&gt; name,
    delegate* unmanaged[Cdecl]&lt;nint, nint, nint&gt; callback)
{
    NativeMethods.CreateFunction(env, name, (nuint)name.Length, callback, 0, out nint fn);
    NativeMethods.SetNamedProperty(env, exports, name, fn);
}</code></pre>
<p>The source-generated <code>[LibraryImport]</code> handles the marshalling. <code>ReadOnlySpan&lt;byte&gt;</code> maps cleanly to <code>const char*</code>, function pointers are passed through directly, and the generated code is trimming-compatible out of the box.</p>
<h2>Marshalling strings</h2>
<p>Most of the interop work comes down to moving strings between JavaScript and .NET. N-API uses UTF-8, so the conversion is straightforward, though it does require a buffer. Here&#8217;s a helper that reads a string argument passed from JavaScript:</p>
<pre><code class="language-csharp">private static unsafe string? GetStringArg(nint env, nint cbinfo, int index)
{
    nuint argc = (nuint)(index + 1);
    Span&lt;nint&gt; argv = stackalloc nint[index + 1];
    NativeMethods.GetCallbackInfo(env, cbinfo, ref argc, argv, null, null);

    if ((int)argc &lt;= index)
        return null;

    // Ask N-API for the UTF-8 byte length
    NativeMethods.GetValueStringUtf8(env, argv[index], null, 0, out nuint len);

    // Allocate a buffer
    int bufLen = (int)len + 1;
    byte[]? rented = null;
    Span&lt;byte&gt; buf = bufLen &lt;= 512
        ? stackalloc byte[bufLen]
        : (rented = ArrayPool&lt;byte&gt;.Shared.Rent(bufLen));

    try
    {
        fixed (byte* pBuf = buf)
            NativeMethods.GetValueStringUtf8(env, argv[index], pBuf, len + 1, out _);

        return Encoding.UTF8.GetString(buf[..(int)len]);
    }
    finally
    {
        if (rented is not null)
            ArrayPool&lt;byte&gt;.Shared.Return(rented);
    }
}</code></pre>
<p>This code asks N-API for the byte length, allocates a buffer (on the stack for small strings, from the pool for larger ones), reads the bytes, then decodes to a .NET <code>string</code>.</p>
<p>Returning a string to JavaScript is the same process in reverse. We encode a .NET string into a UTF-8 buffer and pass it to <code>napi_create_string_utf8</code>:</p>
<pre><code class="language-csharp">private static nint CreateString(nint env, string value)
{
    int byteCount = Encoding.UTF8.GetByteCount(value);

    byte[]? rented = null;
    Span&lt;byte&gt; buf = byteCount &lt;= 512
        ? stackalloc byte[byteCount]
        : (rented = ArrayPool&lt;byte&gt;.Shared.Rent(byteCount));

    try
    {
        Encoding.UTF8.GetBytes(value, buf);
        NativeMethods.CreateStringUtf8(
            env, buf[..byteCount], (nuint)byteCount, out nint result);
        return result;
    }
    finally
    {
        if (rented is not null)
            ArrayPool&lt;byte&gt;.Shared.Return(rented);
    }
}</code></pre>
<p>Both directions use <code>Span&lt;T&gt;</code>, <code>stackalloc</code>, and <code>ArrayPool</code> to avoid heap allocations for typical string sizes. Once you have these helpers in place, you can write exported functions without thinking much about marshalling values.</p>
<h2>Implementing an exported function</h2>
<p>With the N-API plumbing in place, implementing an actual exported function is straightforward. Here&#8217;s one that reads a value from the Windows Registry and returns it to JavaScript as a string:</p>
<pre><code class="language-csharp">[UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
private static nint ReadStringValue(nint env, nint info)
{
    try
    {
        var keyPath = GetStringArg(env, info, 0);
        var valueName = GetStringArg(env, info, 1);

        if (keyPath is null || valueName is null)
        {
            ThrowError(env, "Expected two string arguments: keyPath, valueName");
            return 0;
        }

        using var key = Registry.CurrentUser.OpenSubKey(
            keyPath,
            writable: false);

        return key?.GetValue(valueName) is string value
            ? CreateString(env, value)
            : GetUndefined(env);
    }
    catch (Exception ex)
    {
        ThrowError(env, $"Registry read failed: {ex.Message}");
        return 0;
    }
}</code></pre>
<p>The structure is the same for every exported function. Read any arguments to the function first. Here we read string arguments with <code>GetStringArg</code>. Then, do the work using normal .NET APIs, and finally return a result via <code>CreateString</code> or similar. One thing to be careful about is exception handling &#8211; an unhandled exception in an <code>[UnmanagedCallersOnly]</code> method will crash the host process. We catch exceptions and forward them to JavaScript via <code>ThrowError</code>, which causes a standard JavaScript <code>Error</code> to be thrown on the calling side.</p>
<p>This example also shows why native addons are useful in the first place. Node.js doesn&#8217;t have built-in access to the Windows Registry, so a native addon lets us use <code>Microsoft.Win32.Registry</code> from .NET and expose the result to JavaScript with minimal ceremony.</p>
<h2>Calling our function from TypeScript</h2>
<p>First, we must produce a platform-specific shared library. Running <code>dotnet publish</code> produces a native library appropriate for your operating system (for example, <code>RegistryAddon.dll</code> on Windows, <code>libRegistryAddon.so</code> on Linux, or <code>libRegistryAddon.dylib</code> on macOS). By convention, Node.js treats paths ending with <code>.node</code> as native addons, so we rename this output file to <code>MyNativeAddon.node</code>.</p>
<p>We declare a TypeScript interface for our module, through which we expose type-safe access to our module&#8217;s functions:</p>
<pre><code class="language-typescript">interface RegistryAddon {
    readStringValue(keyPath: string, valueName: string): string | undefined;

    // Declare additional functions...
}</code></pre>
<p>From there, loading it in TypeScript is a standard <code>require()</code> call:</p>
<pre><code class="language-typescript">// Load our native module
const registry = require('./native/win32-x64/RegistryAddon.node') as RegistryAddon

// Call our native function
const sdkPath = registry.readStringValue(
    'SOFTWARE\\dotnet\\Setup\\InstalledVersions\\x64\\sdk', 'InstallLocation')</code></pre>
<p>And with that, we&#8217;re done! We can call from TypeScript into native code that was written in C#. While this particular registry addon is Windows-only, the same Native AOT and N-API approach works equally well on Windows, Linux, and macOS.</p>
<h2>What about existing libraries?</h2>
<p>There is an existing project, <a href="https://github.com/microsoft/node-api-dotnet">node-api-dotnet</a>, that provides a higher-level framework for .NET/JavaScript interop. It handles a lot of the boilerplate and supports richer scenarios. For our use case, we only needed a handful of functions, and the thin N-API wrapper gave us full control over the interop layer without bringing in additional dependencies. If you need to expose entire .NET classes or handle callbacks from JavaScript into .NET, a library like that is worth considering.</p>
<h2>What we gained</h2>
<p>The immediate, practical benefit was simplifying our contributor experience. Anyone who wants to develop in our repo no longer needs a specific Python version. <code>yarn install</code> works with just Node.js, C++ tooling and the .NET SDK, which are tools we already require for development. Our CI pipelines are simpler as well.</p>
<p>Performance has been comparable to the C++ implementation. Native AOT produces optimized native code, and for the kind of work these functions do &#8211; string marshalling, registry access &#8211; there&#8217;s no meaningful difference in practice. The .NET runtime does bring a garbage collector and a slightly larger memory footprint, but in a long-running VS Code extension process this is negligible.</p>
<p>Looking ahead, this opens up some interesting possibilities. We currently run substantial .NET workloads in a separate process, communicating over a pipe. With Native AOT producing shared libraries that load directly into the Node.js process, we could potentially host some of that logic in-process, avoiding the serialization and process-management overhead. That&#8217;s a longer-term exploration, but the foundation is now in place.</p>
<h2>A footnote</h2>
<p>When the idea of using Native AOT first arose, no one on the team had direct experience of integrating native code with Node.js. Even though we have experience with Native AOT, the prospect of learning N-API&#8217;s C calling conventions and wiring up the interop might have seemed daunting enough to put the whole idea on the back burner. GitHub Copilot allowed us to get a working proof-of-concept running very quickly, at which point the idea seemed promising enough to pursue. It&#8217;s been a fantastic tool for exploring ideas that we wouldn&#8217;t previously have had the time for. It&#8217;s improving our products, and the team&#8217;s quality of life.</p>
<h2>Summary</h2>
<p>Native AOT increases the number of places you can run your .NET code. In this case, it allowed us to consolidate our tooling around fewer technologies and streamline our developer experience, particularly for onboarding new developers to the codebase.</p>
<p>If you&#8217;re running in Node.js, or in any other environment that can load native code, consider using Native AOT to produce that code. It allows you to write your native code in a language with memory safety, a rich standard library, and modern tooling. And if you&#8217;re not very familiar with native coding, you might be surprised to learn just how simple it can be to wire this all up (especially if you have a Copilot to help).</p>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/writing-nodejs-addons-with-dotnet-native-aot/">Writing Node.js addons with .NET Native AOT</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
