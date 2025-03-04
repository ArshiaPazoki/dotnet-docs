---
title: P/Invoke source generation
description: Learn about compile-time source generation for platform invokes in .NET.
ms.date: 07/25/2022
---

# Source generation for platform invokes

.NET 7 introduces a [source generator](../../csharp/roslyn-sdk/source-generators-overview.md) for P/Invokes in C# which recognizes the <xref:System.Runtime.InteropServices.LibraryImportAttribute>.

When a P/Invoke is defined and called as shown in the following code, the built-in interop system in the .NET runtime generates an IL stub&mdash;a stream of IL instructions that is JIT-ed&mdash;at run time to facilitate the transition from managed to unmanaged.

```csharp
[DllImport(
    "nativelib",
    EntryPoint = "to_lower",
    CharSet = CharSet.Unicode)]
internal static extern string ToLower(string str);

// string lower = ToLower("StringToConvert");
```

The IL stub handles [marshalling](type-marshalling.md) of parameters and return values and calling the unmanaged code while respecting settings on <xref:System.Runtime.InteropServices.DllImportAttribute> that affect how the unmanaged code should be invoked (for example, <xref:System.Runtime.InteropServices.DllImportAttribute.SetLastError>). Since this IL stub is generated at run time, it is not available for ahead-of-time (AOT) compiler or IL trimming scenarios. Debugging the marshalling logic is also a non-trivial exercise.

The P/Invoke source generator, included with the .NET 7 SDK and enabled by default, looks for <xref:System.Runtime.InteropServices.LibraryImportAttribute> on a `static` and `partial` method to trigger compile-time source generation of marshalling code, removing the need for the generation of an IL stub at run time and allowing the P/Invoke to be inlined. Analyzers and code fixers are also included to help with migration from the built-in system to the source generator and with usage in general.

## Basic usage

The <xref:System.Runtime.InteropServices.LibraryImportAttribute> is designed to be similar to <xref:System.Runtime.InteropServices.DllImportAttribute> in usage. We can convert the example above to use P/Invoke source generation by using the <xref:System.Runtime.InteropServices.LibraryImportAttribute> and marking the method as `partial` instead of `extern`:

```csharp
[LibraryImport(
    "nativelib",
    EntryPoint = "to_lower",
    StringMarshalling = StringMarshalling.Utf16)]
internal static partial string ToLower(string str);
```

During compilation, the source generator will trigger to generate an implementation of the `ToLower` method that handles marshalling of the `string` parameter and return value as UTF-16. Since the marshalling is now generated source code, you can actually look at and step through the logic in a debugger.

### `MarshalAs`

The source generator also respects the <xref:System.Runtime.InteropServices.MarshalAsAttribute>. The preceding code could also be written as:

```csharp
[LibraryImport(
    "nativelib",
    EntryPoint = "to_lower")]
[return: MarshalAs(UnmanagedType.LPWStr)]
internal static partial string ToLower(
    [MarshalAs(UnmanagedType.LPWStr)] string str);
```

Some settings for <xref:System.Runtime.InteropServices.MarshalAsAttribute> are not supported. The source generator will emit an error if you try to use unsupported settings. For more details, see [Differences from DllImport](#differences-from-dllimport).

### Calling convention

To specify the calling convention, use <xref:System.Runtime.InteropServices.UnmanagedCallConvAttribute>, for example:

```csharp
[LibraryImport(
    "nativelib",
    EntryPoint = "to_lower",
    StringMarshalling = StringMarshalling.Utf16)]
[UnmanagedCallConv(
    CallConvs = new[] { typeof(CallConvStdcall) })]
internal static partial string ToLower(string str);
```

## Differences from `DllImport`

<xref:System.Runtime.InteropServices.LibraryImportAttribute> is intended to be a straight-forward conversion from <xref:System.Runtime.InteropServices.DllImportAttribute> in most cases, but there are some intentional changes.

- <xref:System.Runtime.InteropServices.DllImportAttribute.CallingConvention> has no equivalent on <xref:System.Runtime.InteropServices.LibraryImportAttribute>. <xref:System.Runtime.InteropServices.UnmanagedCallConvAttribute> should be used instead.
- <xref:System.Runtime.InteropServices.CharSet> (for <xref:System.Runtime.InteropServices.DllImportAttribute.CharSet>) has been replaced with <xref:System.Runtime.InteropServices.StringMarshalling> (for <xref:System.Runtime.InteropServices.LibraryImportAttribute.StringMarshalling>). ANSI has been removed and UTF-8 is now available as a first-class option.
- <xref:System.Runtime.InteropServices.DllImportAttribute.BestFitMapping> and <xref:System.Runtime.InteropServices.DllImportAttribute.ThrowOnUnmappableChar> have no equivalent. These were only relevant when marshalling an ANSI string on Windows. The generated code for marshalling an ANSI string will have the equivalent behaviour of `BestFitMapping=false` and `ThrowOnUnmappableChar=false`.
- <xref:System.Runtime.InteropServices.DllImportAttribute.ExactSpelling> has no equivalent. This was a Windows-centric setting and had no effect on non-Windows. The method name or <xref:System.Runtime.InteropServices.LibraryImportAttribute.EntryPoint> should be the exact spelling of the entry point name.
- <xref:System.Runtime.InteropServices.DllImportAttribute.PreserveSig> has no equivalent. This was a Windows-centric setting. The generated code always directly translates the signature.

There are also differences in support for some settings on <xref:System.Runtime.InteropServices.MarshalAsAttribute>, default marshalling of certain types, and other interop-related attributes. For more details, see our [documentation on compatibility differences](https://github.com/dotnet/runtime/blob/main/docs/design/libraries/LibraryImportGenerator/Compatibility.md).

## See also

- [P/Invoke](pinvoke.md)
- <xref:System.Runtime.InteropServices.LibraryImportAttribute>
