# Plugin Development Guide
This document is a guide for developing a plugin that projects native APIs across multiple platforms
(Android, iOS, Windows) and multiple application frameworks (Cordova, React Native, Xamarin).

## Plugin layout
Layout your plugin code as follows, with native-code projects for each platform. Not all platforms are
required, but there must be at least one.

        MyPlugin/
            plugin.xml (See below)
            android/
                (Android Studio project files and Java code)
            ios
                (XCode project files and Obj-C code)
            windows/
                (VS project files and C# or C++/CX code)

## General guidelines
 * *Public* types and members are projected as part of the plugin API. *Private* or *internal* types and members are
   ignored.
 * There are no special attributes that need to be applied on any types or members to enable the projection.
 * Native APIs are subject to some transformations to make the projected API surface more consistent across platforms
   and more natural for the consuming environment (C# or JavaScript). Read further for more details.

Each of the platform-specific plugin projects may itself be a native SDK (if the SDKs were designed with
cross-platform projections in mind), or they may be wrappers/adapters that enable native SDKs or OS functionality
for each of the platforms to be projected as a cross-platform API.

## Supported API Constructs
 * **Classes** - Ordinary classes are projected in the plugin APIs, and instances of those classes may be passed back
   and forth as property/method parameters and return values. But be aware of marshalling considerations discussed below.
 * **Constructors** - Constructors that take parameters are supported. Overloads in the number of constructor parameters
   are supported.
 * **Properties** - Static and instance properties are supported. (See projection notes for Android and iOS.)
 * **Methods** - Static and instance methods are supported.
 * **Async** - Asynchronous methods are supported using some transformations; see projection notes for each platform.
 * **Exceptions** - Exceptions thrown by an API implementation are propagated out to the caller. In most cases the
   exception message is included, but no additional exception data. (See the note about error projections for iOS.)
 * **Events** - Static and instance .NET style multicast events are supported on all platforms. (See projection notes
   for Android and iOS.)
 * **Collections** - Basic collections such as arrays, lists, and dictionaries are supported, with automatic
   conversions between collection types for different environments.

## Limitations
 * **Interfaces** - Interface support is not fully implemented. Projected classes may implement interfaces,
   but those interfaces are omitted from the projected API. Use of interfaces as property types or parameters
   to constructors or methods may cause build-time or run-time errors.
 * **Inheritance** - Inheritance is not fully implemented. Projected classes may inherit from base classes other than
   Object, but the inheritance relationship and inherited members are omitted from the projection. Use of base
   classes as property types or parameters to constructors or methods may cause build-time or run-time errors.
 * **Nested types** - WinRT (and therefore Windows projections to JavaScript) does not support nested types. It is
   possible to project plugin APIs with nested types only if not targeting the Windows platform.
 * **Generics** - Generics support is partially implemented. Specifically, generic collections such as IList<T> and
   IDictionary<K, V> and their Android & iOS equivalents are known to work, but complex nested generic types may not.
 * **Overloads** - Overloaded methods (that is multiple methods on the same type with the same name but
   different parameter types or counts) are not fully implemented.

Since most of these limitations are not design limitations, it's possible some of them could be removed after
further development.

Besides these limitations, there are some special considerations related to marshalling that may affect the API design.

## Marshalling
When instances of plugin classes are passed between native code and the JavaScript runtime for Cordova or
React Native, they must be *marshalled*. (This does not apply to Xamarin apps.) Classes can be marshalled in one
of two ways: by reference or by value. Generally, long-lived or complex types should be marshalled by reference,
while short-lived or simple types (property bags) and events should be marshalled by value.

### Marshal by reference
By default, an instance of a native class is marshalled to JavaScript by reference. That means the main instance
remains in existence on the native side, and the JavaScript side just gets a reference, which it may use to
make further calls to members of that object. This has some important implications:

 * All calls to members on an object that is marshalled by reference are async, because the JavaScript bridge is
   async. The plugin build process akes care of converting all synchronous properties and methods to async
   JavaScript projections.
 * Because the native instance is held in case of further calls via the reference, the JavaScript side is
   responsible for the lifetime of the instance. The JavaScript owner of the object must call a `release()`
   method to allow the native instance to be disposed. This is a bit unusual for JavaScript programming, but
   necessary because JavaScript doesn't have any support for finalizers or weak references.

An instance of a marshal-by-ref class can be created from the JavaScript side by simply invoking the constructor.
Doing so actually makes an asynchronous call over the bridge to invoke the native constructor, though that
asynchronicity is hidden from the caller: the next call to an async method on the object will automatically
wait for that construction to finish if necessary.

### Marshal by value
Classes may be designated as marshal-by-value in plugin.xml, to override the default marshal-by-ref behavior.
When an instance of a marshal-by-value class is marshalled, all of its property values are copied over the bridge.
(But if a marshal-by-value class includes a property that is a marshal-by-ref class, then that property is
still copied by reference.) Marshal-by-value has the following implications:

 * Only property values are accessible to JavaScript; no method calls are available.
 * Access to properties is synchronous.
 * After a native instance is marshalled by value to JavaScript, no reference is maintained to the native instance,
   so the instance may get deleted unless other native objects still reference it.

Marshal-by-value classes may be either *one-way* or *two-way* marshallable. A one-way marshal-by-value class has a
non-public constructor and only property getters, so it may be constructed only on the native-code side and is
then read-only from the JavaScript side. A two-way marshal-by-value class has a public parameter-less constructor
and both getters and setters for all properties. So it may be constructed on either side of the bridge and passed
in either direction.

Note event objects are automatically marshalled by value (one-way), so they don't need to be explicitly designated
as such in plugin.xml.

## Target Platforms
For more details and specific requirements concerning each target platform, refer to the following documents:
 * [Android](Android.md)
 * [iOS](iOS.md)
 * [Windows](Windows.md)

## Plugin XML
See the plugin.xml file in the test plugin for example syntax.

### Plugin identity properties
The **plugin.id** attribute becomes the npm package name prefix for Cordova and React Native plugins. The
**description**, **author**, **keywords**, and **license** elements go to the corresponding properties
in the generated package.json.

### Assembly elements
The **assembly** element is required for all plugins. The **assembly.name** attribute becomes the the Xamarin
NuGet package ID and assembly name, and is also used in a few other places.

The **assembly.js-target** attribute indicates the name of the global object that is injected into the
Cordova scripting environment by the generated Cordova plugin. Classes projected by the plugin are properties
on that object.

Optional **class** elements under the assembly element can be used to specify parameters that control how
classes and members are projected as plugin APIs. In some cases C3P may not have enough information to
automatically infer the right kind of projection, so these hints can help achieve a desired result that
is consistent across platforms.

### Namespace mappings
A small amount of metadata is required to map namespaces from the native APIs to plugin APIs. The mapping
mechanism depends on the platform.

* **Android** - Use a `<namespace-mapping>` element to map a Java namespace to a plugin namespace. Any classes
in packages not mapped to a plugin namespace will not be projected as plugin APIs.
* **iOS** - Use a `<namespace-mapping>` element to map an [Obj-C prefix 3- or 4-letter prefix](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Conventions/Conventions.html)
to a plugin namespace. Any classes with prefixes not mapped to a plugin namespace will not be projected as
plugin APIs.
* **Windows** - Windows code is already organized into namespaces. Windows namespaces are projected
directly to plugin APIs if there is an `<include>` element for the namespace.

Following are namespace mapping examples for all 3 platforms, from the test plugin's plugin.xml:

```
    <platform name="android">
        <namespace-mapping
            package="com.microsoft.c3p.test"
            namespace="Microsoft.C3P.Test" />
    </platform>

    <platform name="ios">
        <namespace-mapping
            prefix="C3PT"
            namespace="Microsoft.C3P.Test" />
    </platform>

    <platform name="windows">
        <include namespace="Microsoft.C3P.Test" />
    </platform>
```

Like all APIs, namespace mappings should converge to a common cross-platform API. Inconsistencies or
conflicts are detected at link time and reported as warnings or errors.

### Cordova config file transformations
The plugin.xml may include [Cordova config file transformations](http://cordova.apache.org/docs/en/latest/plugin_ref/spec.html#config-file). These are ignored when
linking React Native or Xamarin plugins.

## The runtime library plugin
Cordova and React Native plugins built with C3P depend on a separate C3P runtime library plugin. (Xamarin plugins
do not have a similar dependency.) These C3P plugins should already be published to NPM as `c3p-cordova` and
`c3p-reactnative`.

If you want to build those plugins yourself, use the CLI 'pack' command:

    cd c3p/src/lib
    c3p pack cordova
    c3p pack reactnative

This will output plugins at c3p/src/lib/build/cordova and c3p/src/lib/build/reactnative.
