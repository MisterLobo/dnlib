dot10 is a library that can read, write and create .NET assemblies and modules.

It was written for de4dot which must have a rock solid assembly reader and writer library since it has to deal with heavily obfuscated assemblies with invalid metadata. If the CLR can load the assembly, dot10 must be able to read it and save it.

Features
* Supports reading, writing and creating .NET assemblies/modules targeting any .NET framework (eg. desktop, Silverlight, Windows Phone, etc).
* Supports reading and writing mixed mode assemblies (eg. C++/CLI)
* Can read and write non-ECMA compatible .NET assemblies that MS' CLR can load and execute
* Very stable and can handle obfuscated assemblies that crash other similar libraries.
* High and low level access to the metadata
* Output size of non-obfuscated assemblies is usually smaller than the original assembly
* Metadata tokens and heaps can be preserved when saving an assembly
* Assembly reader has hooks for decrypting methods and strings
* Assembly writer has hooks for various writer events
* Easy to port code from Mono.Cecil to dot10
* Add/delete Win32 resource blobs
* Saved assemblies can be strong name signed and enhanced strong name signed

Compiling
---------

You must have Visual Studio 2008 or later. The solution file was created by Visual Studio 2010, so if you use VS2008, open the solution file and change the version number so VS2008 can read it.

Examples
--------

All examples use C#, but since it's a .NET library, you can use any .NET language (eg. VB.NET).

See the Examples project for several examples.

Opening a .NET assembly/module
------------------------------

First of all, the important namespaces are `dot10.DotNet` and `dot10.DotNet.Emit`. `dot10.DotNet.Emit` is only needed if you intend to read/write method bodies. All the examples below assume you have the appropriate using statements at the top of each source file:

```C#
using dot10.DotNet;
using dot10.DotNet.Emit;
```

ModuleDefMD is the class that is created when you open a .NET module. It has several `Load()` methods that will create a ModuleDefMD instance. If it's not a .NET module/assembly, a `BadImageFormatException` will be thrown.

Read a .NET module from a file:

```C#
ModuleDefMD module = ModuleDefMD.Load(@"C:\path\to\file.exe");
```

Read a .NET module from a byte array:

```C#
byte[] data = System.IO.File.ReadAllBytes(@"C:\path\of\file.dll");
ModuleDefMD module = ModuleDefMD.Load(data);
```

You can also pass in a Stream instance, an address in memory (HINSTANCE) or even a System.Reflection.Module instance:

```C#
System.Reflection.Module reflectionModule = typeof(void).Module;	// Get mscorlib.dll's module
ModuleDefMD module = ModuleDefMD.Load(reflectionModule);
```

To get the assembly, use its Assembly property:

```C#
AssemblyDef asm = module.Assembly;
Console.WriteLine("Assembly: {0}", asm);
```

Saving a .NET assembly/module
-----------------------------

Use `module.Write()`. It can save the assembly to a file or a Stream.

```C#
module.Write(@"C:\saved-assembly.dll");
```

If it's a C++/CLI assembly, you should use `NativeWrite()`

```C#
module.NativeWrite(@"C:\saved-assembly.dll");
```

To detect it at runtime, use this code:

```C#
if (module.IsILOnly) {
	// This assembly has only IL code, and no native code (eg. it's a C# or VB assembly)
	module.Write(@"C:\saved-assembly.dll");
}
else {
	// This assembly has native code (eg. C++/CLI)
	module.NativeWrite(@"C:\saved-assembly.dll");
}
```

Strong name sign an assembly
----------------------------

When saving the assembly, set the ModuleWriterOptions.StrongNameKey property:

```C#
using dot10.DotNet.Writer;
...
ModuleDefMD mod = ModuleDefMD.Load(.....);
ModuleWriterOptions opts = new ModuleWriterOptions(mod);
opts.StrongNameKey = new StrongNameKey(@"c:\my\file.snk");
mod.Write(@"C:\out\file.dll", opts);
```

Enhanced strong name signing an assembly
----------------------------------------

See this [MSDN article](http://msdn.microsoft.com/en-us/library/hh415055.aspx) for info on enhanced strong naming.

Enhanced strong name signing without key migration:

```C#
// Open or create an assembly
ModuleDefMD mod = ModuleDefMD.Load(....);

// Open or create the signature keys
var signatureKey = new StrongNameKey(....);
var signaturePubKey = new StrongNamePublicKey(....);

// Create module writer options
var opts = new ModuleWriterOptions(mod);

// This method will initialize the required properties
opts.InitializeEnhancedStrongNameSigning(signatureKey, signaturePubKey);

// Write and strong name sign the assembly
mod.Write(@"C:\out\file.dll", opts);
```

Enhanced strong name signing with key migration:

```C#
// Open or create an assembly
ModuleDefMD mod = ModuleDefMD.Load(....);

// Open or create the identity and signature keys
var signatureKey = new StrongNameKey(....);
var signaturePubKey = new StrongNamePublicKey(....);
var identityKey = new StrongNameKey(....);
var identityPubKey = new StrongNamePublicKey(....);

// Create module writer options
var opts = new ModuleWriterOptions(mod);

// This method will initialize the required properties and add
// the required attribute to the assembly.
opts.InitializeEnhancedStrongNameSigning(mod, signatureKey, signaturePubKey, identityKey, identityPubKey);

// Write and strong name sign the assembly
mod.Write(@"C:\out\file.dll", opts);
```

Type classes
------------

The metadata has three type tables: `TypeRef`, `TypeDef`, and `TypeSpec`. The classes dot10 use are called the same. These three classes all implement `ITypeDefOrRef`.

There's also type signature classes. The base class is `TypeSig`. You'll find `TypeSig`s in method signatures (return type and parameter types) and locals. The `TypeSpec` class also has a `TypeSig` property.

All of these types implement `IType`.

`TypeRef` is a reference to a type in (usually) another assembly.

`TypeDef` is a type definition and it's a type defined in some module. This class does *not* derive from `TypeRef`. :)

`TypeSpec` can be a generic type, an array type, etc.

`TypeSig` is the base class of all type signatures (found in method sigs and locals). It has a `Next` property that points to the next `TypeSig`. Eg. a Byte[] would first contain a `SZArraySig`, and its `Next` property would point to Byte signature.

`CorLibTypeSig` is a simple corlib type. You don't create these directly. Use eg. `module.CorLibTypes.Int32` to get a System.Int32 type signature.

`ValueTypeSig` is used when the next class is a value type.

`ClassSig` is used when the next class is a reference type.

`GenericInstSig` is a generic instance type. It has a reference to the generic type (a `TypeDef` or a `TypeRef`) and the generic arguments.

`PtrSig` is a pointer sig.

`ByRefSig` is a by reference type.

`ArraySig` is a multi-dimensional array type. Most likely when you create an array, you should use `SZArraySig`, and *not* `ArraySig`.

`SZArraySig` is a single dimension, zero lower bound array. In C#, a `byte[]` is a `SZArraySig`, and *not* an `ArraySig`.

`GenericVar` is a generic type variable.

`GenericMVar` is a generic method variable.

Some examples if you're not used to the way type signatures are represented in metadata:

```C#
ModuleDef mod = ....;

// Create a byte[]
SZArraySig array1 = new SZArraySig(mod.CorLibTypes.Byte);

// Create an int[][]
SZArraySig array2 = new SZArraySig(new SZArraySig(mod.CorLibTypes.Int32));

// Create an int[,]
ArraySig array3 = new ArraySig(mod.CorLibTypes.Int32, 2);

// Create an int[*] (one-dimensional array)
ArraySig array4 = new ArraySig(mod.CorLibTypes.Int32, 1);

// Create a Stream[]. Stream is a reference class so it must be enclosed in a ClassSig.
// If it were a value type, you would use ValueTypeSig instead.
TypeRef stream = new TypeRefUser(mod, "System.IO", "Stream", mod.CorLibTypes.AssemblyRef);
SZArraySig array5 = new SZArraySig(new ClassSig(stream));
```

Sometimes you must convert an `ITypeDefOrRef` (`TypeRef`, `TypeDef`, or `TypeSpec`) to/from a `TypeSig`. There's extension methods you can use:

```C#
// array5 is defined above
ITypeDefOrRef type1 = array5.ToTypeDefOrRef();
TypeSig type2 = type1.ToTypeSig();
```

Naming conventions of metadata table classes
--------------------------------------------

For most tables in the metadata, there's a corresponding dot10 class with the exact same name. Eg. the metadata has a `TypeDef` table, and dot10 has a `TypeDef` class. Some tables don't have a class because they're referenced by other classes, and that information is part of some other class. Eg. the `TypeDef` class contains all its properties and events, even though the `TypeDef` table has no property or event column.

For each of these table classes, there's an abstract base class, and two sub classes. These sub classes are named the same as the base class but ends in either `MD` (for classes created from the metadata) or `User` (for classes created by the user). Eg. `TypeDef` is the base class, and it has two sub classes `TypeDefMD` which is auto-created from metadata, and `TypeRefUser` which is created by the user when adding user types. Most of the XyzMD classes are internal and can't be referenced directly by the user. They're created by `ModuleDefMD` (which is the only public `MD` class). All XyzUser classes are public.

Metadata table classes
----------------------

Here's a list of the most common metadata table classes

`AssemblyDef` is the assembly class.

`AssemblyRef` is an assembly reference.

`EventDef` is an event definition. Owned by a `TypeDef`.

`FieldDef` is a field definition. Owned by a `TypeDef`.

`GenericParam` is a generic parameter (owned by a `MethodDef` or a `TypeDef`)

`MemberRef` is what you create if you need a field reference or a method reference.

`MethodDef` is a method definition. It usually has a `CilBody` with CIL instructions. Owned by a `TypeDef`.

`MethodSpec` is a instantiated generic method.

`ModuleDef` is the base module class. When you read an existing module, a `ModuleDefMD` is created.

`ModuleRef` is a module reference.

`PropertyDef` is a property definition. Owned by a `TypeDef`.

`TypeDef` is a type definition. It contains a lot of interesting stuff, including methods, fields, properties, etc.

`TypeRef` is a type reference. Usually to a type in another assembly.

`TypeSpec` is a type specification, eg. an array, generic type, etc.

Method classes
--------------

The following are the method classes: `MethodDef`, `MemberRef` (method ref) and `MethodSpec`. They all implement `IMethod`.

Field classes
-------------

The following are the field classes: `FieldDef` and `MemberRef` (field ref). They both implement `IField`.

Comparing types, methods, fields, etc
-------------------------------------

dot10 has a `SigComparer` class that can compare any type with any other type. Any method with any other method, etc. It also has several pre-created `IEqualityComparer<T>` classes (eg. `TypeEqualityComparer`, `FieldEqualityComparer`, etc) which you can use if you intend to eg. use a type as a key in a `Dictionary<TKey, TValue>`.

The `SigComparer` class can also compare types with `System.Type`, methods with `System.Reflection.MethodBase`, etc.

It has many options you can set, see `SigComparerOptions`. The default options is usually good enough, though.

```C#
// Compare two types
TypeRef type1 = ...;
TypeDef type2 = ...;
if (new SigComparer().Equals(type1, type2))
	Console.WriteLine("They're equal");
```

```C#
// Use the type equality comparer
Dictionary<IType, int> dict = new Dictionary<IType, int>(TypeEqualityComparer.Instance);
TypeDef type1 = ...;
dict.Add(type1, 10);
```

```C#
// Compare a `TypeRef` with a `System.Type`
TypeRef type1 = ...;
if (new SigComparer().Equals(type1, typeof(int)))
	Console.WriteLine("They're equal");
```

It has many `Equals()` and `GetHashCode()` overloads.

.NET Resources
--------------

There's three types of .NET resource, and they all derive from the common base class `Resource`. `ModuleDef.Resources` is a list of all resources the module owns.

`EmbeddedResource` is a resource that has data embedded in the owner module. This is the most common type of resource and it's probably what you want.

`AssemblyLinkedResource` is a reference to a resource in another assembly.

`LinkedResource` is a reference to a resource on disk.

Win32 resources
---------------

`ModuleDef.Win32Resources` can be null or a `Win32Resources` instance. You can add/remove any Win32 resource blob. dot10 doesn't try to parse these blobs.

Parsing method bodies
---------------------

This is usually only needed if you have decrypted a method body. If it's a standard method body, you can use `MethodBodyReader.Create()`. If it's similar to a standard method body, you can derive a class from `MethodBodyReaderBase` and override the necessary methods.

Resolving references
--------------------

`TypeRef.Resolve()` and `MemberRef.Resolve()` both use `module.Context.Resolver` to resolve the type, field or method. The custom attribute parser code may also resolve type references.

If you call Resolve() or read custom attributes, you should initialize module.Context to a `ModuleContext`. It should normally be shared between all modules you open.

```C#
AssemblyResolver asmResolver = new AssemblyResolver();
ModuleContext modCtx = new ModuleContext(asmResolver);

// All resolved assemblies will also get this same modCtx
asmResolver.DefaultModuleContext = modCtx;

// Enable the TypeDef cache for all assemblies that are loaded
// by the assembly resolver. Only enable it if all auto-loaded
// assemblies are read-only.
asmResolver.EnableTypeDefCache = true;
```

All assemblies that you yourself open should be added to the assembly resolver cache.

```C#
MOduleDefMD mod = MOduleDefMD.Load(...);
mod.Context = modCtx;	// Use the previously created (and shared) context
mod.Context.AssemblyResolver.AddToCache(mod);
```

Creating mscorlib type references
---------------------------------

Every module has a `CorLibTypes` property. It has references to a few of the simplest types such as all integer types, floating point types, Object, String, etc. If you need a type that's not there, you must create it yourself, eg.:

```C#
TypeRef consoleRef = new TypeRefUser(mod, "System", "Console", mod.CorLibTypes.AssemblyRef);
```

Importing runtime types, methods, fields
----------------------------------------

To import a `System.Type`, `System.Reflection.MethodInfo`, `System.Reflection.FieldInfo`, etc into a module, use the `Importer` class.

```C#
Importer importer = new Importer(mod);
ITypeDefOrRef consoleRef = importer.Import(typeof(System.Console));
IMethod writeLine = importer.Import(typeof(System.Console).GetMethod("WriteLine"));
```

You can also use it to import types, methods etc from another `ModuleDef`.

Resolving types, methods, and more
----------------------------------

`ModuleDefMD` has several `ResolveXXX()` methods, eg. `ResolveTypeDef()`, `ResolveMethod()`, etc.

Using decrypted methods
-----------------------

If `ModuleDefMD.MethodDecrypter` is initialized, `ModuleDefMD` will call it and check whether the method has been decrypted. If it has, it calls `IMethodDecrypter.GetMethodBody()` which you should implement. Return the new `MethodBody`. `GetMethodBody()` should usually call `MethodBodyReader.Create()` which does the actual parsing of the CIL code.

It's also possible to override `ModuleDefMD.ReadUserString()`. This method is called by the CIL parser when it finds a `Ldstr` instruction. If `ModuleDefMD.StringDecrypter` is not null, its `ReadUserString()` method is called with the string token. Return the decrypted string or null if it should be read from the `#US` heap.

Low level access to the metadata
--------------------------------

The low level classes are in the `dot10.DotNet.MD` namespace.

Open an existing .NET module/assembly and you get a ModuleDefMD. It has several properties, eg. `StringsStream` is the #Strings stream.

The `MetaData` property gives you full access to the metadata.

To get a list of all valid TypeDef rids (row IDs), use this code:

```C#
using dot10.DotNet.MD;
// ...
ModuleDefMD mod = ModuleDefMD.Load(...);
RidList typeDefRids = mod.MetaData.GetTypeDefRidList();
for (int i = 0; i < typeDefRids.Count; i++)
	Console.WriteLine("rid: {0}", typeDefRids[i]);
```
