
# Interoperate with Javascript from .NET WebAssembly

The System.Runtime.InteropServices.JavaScript namespace provides support for interop between .NET and Javascript.  This is applicable when running a .NET WebAssembly module in a JavaScript host such as a browser. These scenarios include either Blazor WebAssembly apps as detailed in [JavaScript interop with ASP.NET Core Blazor](xref:blazor/js-interop/import-export-interop), as well as non-Blazor .NET WebAssembly apps detailed in [Run .NET from JavaScript](dotnet-interop.md). 





> [!NOTE]
> The marshaling to support interop calls and reference passing can be relatively expensive in terms of performance. While this article uses HtmlElement to demonstrate many fundamentals for familiarity, avoid designs that require a high frequency of interop calls or numerous interop object references.

