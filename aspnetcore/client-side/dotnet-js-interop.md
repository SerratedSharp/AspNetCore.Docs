
# Interoperate with Javascript from .NET WebAssembly

The System.Runtime.InteropServices.JavaScript namespace provides support for interop between a .NET WebAssembly and JavaScript and is colloquially referred to as JSImport/JSExport interop.  This is applicable when running a .NET WebAssembly module in a JavaScript host such as a browser. These scenarios include either Blazor WebAssembly apps as detailed in [JavaScript interop with ASP.NET Core Blazor](../blazor/js-interop/import-export-interop), non-Blazor .NET WebAssembly apps detailed in [Run .NET from JavaScript](dotnet-interop.md), and other .NET WebAssembly platforms which support JSImport/JSExport.

## Prerequisites 

[.NET 7.0 SDK](https://dotnet.microsoft.com/download/dotnet/7.0): Install the latest version of the [.NET SDK](https://dotnet.microsoft.com/download/dotnet/).

A project using either one of the following project types:

- A WebAssembly Browser App project setup according to [Run .NET from JavaScript](https://learn.microsoft.com/en-us/aspnet/core/client-side/dotnet-interop?view=aspnetcore-8.0) by completing [Prerequisites](https://learn.microsoft.com/en-us/aspnet/core/client-side/dotnet-interop?view=aspnetcore-8.0#prerequisites) and [Project Configuration](https://learn.microsoft.com/en-us/aspnet/core/client-side/dotnet-interop?view=aspnetcore-8.0#project-configuration).
    - Once you have installed the wasm-tools and wasm-expiremental workloads, a new project template will be available in Visual Studio's new project dialog called "WebAssembly Browser App".
- A Blazor client-side project setup according to [JavaScript JSImport/JSExport interop with ASP.NET Core Blazor](https://learn.microsoft.com/en-us/aspnet/core/blazor/javascript-interoperability/import-export-interop?view=aspnetcore-8.0).

This isn't an exhaustive list, as other commercial and/or open-source platforms exist which enable compiling .NET code to WebAssemby and support code using `System.Runtime.InteropServices.JavaScript`.

## Importing Static JS Methods

This example imports an existing static JS method into C#.  Strictly speaking, `.log()` is an instance method of the `console` object.  However, as demonstrated here JSImport can be accessed using static semantics for instances available on global properties.

```C#
public partial class GlobalProxy
{
    // mapping existing console.log to a C# method
    [JSImport("globalThis.console.log")]
    public static partial void ConsoleLog(string text);
}

//... called from Program.cs Main() of a WASM project:
GlobalProxy.ConsoleLog("Hello World");//We'd expect output to appear in the browser console
```

The following demonstrates declaring and importing a static JS method. This JavaScript would informally be called a JS shim.  In this case it "shims" or fills the gap between the .NET implementation and existing JS capabilities/libraries.  As demonstrated later, a JS shim can also serve to encapsulate additional logic, and reduce the number objects or calls crossing the interop boundary.  JS declarations such as this would typically be loaded from a *.js module.  Some developers opt to implement shims in typescript, but it is important to be knowledgeable of the full JS typename to ensure the correct name is referenced by JSImport.

Declaring a custom JS static method:
```JS
globalThis.callAlert = function (text) {
	globalThis.window.alert(text);
}
```

Mapping to a C# method proxy:
```C# 
using System.Runtime.InteropServices.JavaScript;

public partial class GlobalProxy
{
	[JSImport("globalThis.callAlert", "GlobalShim")]
	public static partial void CallAlert(string text);
}

//... called from WASM Program.cs Main():
GlobalProxy.CallAlert("Hello World");
```

This example assumes the Javascript file containing the Javascript callAlert declaration was loaded from an ES6 module, and thus the module name is passed to the JSImport attribute.  By contrast, the module name is ommitted in the prior example which imported `globalThis.console.log` directly.

# JS Interop using JSImport
Approaches of exposing types or methods from JS to C#.  Allows C# code to call into JS, or hold and pass references to JS objects.

## JS Primitives

Demonstrates JSImport leveraging type mappings of several primitive JS types, and use of JSMarshalAs where explicit mappings are required at compile time.

```JS
// PrimitivesShim.js
let PrimitivesShim = {};
(function (PrimitivesShim) {

    globalThis.counter = 0;

    // Takes no parameters and returns nothing
    PrimitivesShim.IncrementCounter = function () {
        globalThis.counter += 1;
    };

    // Returns an int
    PrimitivesShim.GetCounter = () => globalThis.counter;
    // Identical with more verbose syntax:
    //Primitives.GetCounter = function () { return counter; };

    // Takes a parameter and returns nothing.  JS doesn't restrict the parameter type, but we can restrict it in the .NET proxy if desired.
    PrimitivesShim.LogValue = (value) => { console.log(value); };

    // Called for various .NET types to demonstrate mapping to JS primitive types
    PrimitivesShim.LogValueAndType = (value) => { console.log(typeof value, value); };
    
})(PrimitivesShim);

export { PrimitivesShim };
```

```C#
using System.Runtime.InteropServices.JavaScript;
using System.Threading.Tasks;

public partial class PrimitivesProxy
{    
    // Importing an existing JS method.
    [JSImport("globalThis.console.log")]
    public static partial void ConsoleLog([JSMarshalAs<JSType.Any>] object value);

    // Importing static methods from a JS module.
    [JSImport("PrimitivesShim.IncrementCounter", "PrimitivesShim")]
    public static partial void IncrementCounter();

    [JSImport("PrimitivesShim.GetCounter", "PrimitivesShim")]
    public static partial int GetCounter();

    // The JS shim function name doesn't necessarily have to match the C# method name
    [JSImport("PrimitivesShim.LogValue", "PrimitivesShim")]
    public static partial void LogInt(int value);

    // A second mapping to the same JS function with compatible type
    [JSImport("PrimitivesShim.LogValue", "PrimitivesShim")]
    public static partial void LogString(string value);

    // Accept any type as parameter. .NET types will be mapped to JS types where possible,
    // or otherwise be marshalled as an untyped object reference to the .NET object proxy.
    // The JS implementation logs to browser console the JS type and value to demonstrate results of marshalling.
    [JSImport("PrimitivesShim.LogValueAndType", "PrimitivesShim")]
    public static partial void LogValueAndType([JSMarshalAs<JSType.Any>] object value);

    // Some types have multiple mappings, and need explicit marshalling to the desired JS type.
    // A long/Int64 can be mapped as either a Number or BigInt.
    // Passing a long value to the above method will generate an error "ToJS for System.Int64 is not implemented." at runtime.
    // If the parameter declaration `Method(JSMarshalAs<JSType.Any>] long value)` is used, then a compile time error is generated: "Type long is not supported by source-generated JavaScript interop...."
    // Instead, map the long parameter explicitly to either a JSType.Number or JSType.BigInt.
    // Note there could potentially be runtime overflow errors in JS if the C# value is too large.
    [JSImport("PrimitivesShim.LogValueAndType", "PrimitivesShim")]
    public static partial void LogValueAndTypeForNumber([JSMarshalAs<JSType.Number>] long value);

    [JSImport("PrimitivesShim.LogValueAndType", "PrimitivesShim")]
    public static partial void LogValueAndTypeForBigInt([JSMarshalAs<JSType.BigInt>] long value);
}

public static class PrimitivesUsage
{
    public static async Task Run()
    {
        // Ensure JS ES6 module loaded
        await JSHost.ImportAsync("PrimitivesShim", "https://localhost/PrimitivesShim.js");

        // Call a proxy to a static JS method, console.log("")
        PrimitivesProxy.ConsoleLog("Printed from JSImport of console.log()");

        // Basic examples of JS interop with an integer:       
        PrimitivesProxy.IncrementCounter();
        int counterValue = PrimitivesProxy.GetCounter();
        PrimitivesProxy.LogInt(counterValue);
        PrimitivesProxy.LogString("I'm a string from .NET in your browser!");

        // Mapping some other .NET types to JS primitives:
        // See types table under https://learn.microsoft.com/en-us/aspnet/core/client-side/dotnet-interop?view=aspnetcore-8.0#javascript-interop-on-
        PrimitivesProxy.LogValueAndType(true);
        PrimitivesProxy.LogValueAndType(0x3A);// byte literal
        PrimitivesProxy.LogValueAndType('C');
        PrimitivesProxy.LogValueAndType((Int16)12);
        // Note: Javascript Number has a lower max value and can generate overflow errors
        PrimitivesProxy.LogValueAndTypeForNumber(9007199254740990L);// Int64/Long 
        PrimitivesProxy.LogValueAndTypeForBigInt(1234567890123456789L);// Int64/Long, JS BigInt supports larger numbers
        PrimitivesProxy.LogValueAndType(3.14f);// single floating point literal
        PrimitivesProxy.LogValueAndType(3.14d);// double floating point literal
        PrimitivesProxy.LogValueAndType("A string");
    }
}
// The example displays the following output in the browser's debug console:
//       Printed from JSImport of console.log()
//       1
//       I'm a string from .NET in your browser!
//       boolean true
//       number 58
//       number 67
//       number 12
//       number 9007199254740990
//       bigint 1234567890123456789n
//       number 3.140000104904175
//       number 3.14
//       string A string
```

## JS Date

Demonstrates JSImport of methods which have a JS Date object as its return or parameter.

```JS
// DateShim.js
let DateShim = globalThis.DateShim;
(function (DateShim) {
    
    DateShim.IncrementDay = function (date) {
        date.setDate(date.getDate() + 1);
        return date;
    };
    
    DateShim.LogValueAndType = (value) => {
        if (value instanceof Date) 
            console.log("Date:", value)
        else
            console.log("Not a Date:", value)
    };
        
})(DateShim);

export { DateShim };
```


```C#
using System.Runtime.InteropServices.JavaScript;
using System.Threading.Tasks;

public partial class DateProxy
{   
    [JSImport("DateShim.IncrementDay", "DateShim")]
    [return: JSMarshalAs<JSType.Date>] // Explicit JSMarshalAs for a return type.
    public static partial DateTime IncrementDay([JSMarshalAs<JSType.Date>] DateTime date);
    
    [JSImport("DateShim.LogValueAndType", "DateShim")]
    public static partial void LogValueAndType([JSMarshalAs<JSType.Date>] DateTime value);

}

public static class DateUsage
{
    public static async Task Run()
    {
        // Ensure JS ES6 module loaded
        await JSHost.ImportAsync("DateShim", "https://localhost:7017/DateShim.js");

        // Basic examples of interop with a C# DateTime and JS Date:
        DateTime date = new DateTime(1968, 12, 21, 12, 51, 0, DateTimeKind.Utc);            
        DateProxy.LogValueAndType(date);
        date = DateProxy.IncrementDay(date);        
        DateProxy.LogValueAndType(date);
    }
}
// The example displays the following output in the browser's debug console:
//     Date: Sat Dec 21 1968 07:51:00 GMT-0500 (Eastern Standard Time)
//     Date: Sun Dec 22 1968 07:51:00 GMT-0500 (Eastern Standard Time)  
```

## JS Object References

Whenever a JS method returns an object reference, it is represented in .NET with the JSObject type.  The original JS object continues its lifetime within the JS boundary, while .NET code can access and modify it by reference through the JSObject.  While the type itself exposes a limited API, the ability to hold a JS object reference as well as return or pass it across the interop boundary enables a great deal of capabilities.

The JSObject provides methods to access properties, but does not provide direct access to instance methods.  As the `Summarize()` method demonstrates below, instance methods can be accessed indirectly by implementing a static method that takes the instance to be acted on as a parameter.

```JS
// JSObjectShim.js
let JSObjectShim = {};
(function (JSObjectShim) {

    JSObjectShim.CreateObject = function () {
        return {
            name: "Example JS Object",
            answer: 41,
            question: null,
            summarize: function () {
                return `The question is "${this.question}" and the answer is ${this.answer}.`;
            }
        };
    };
    
    JSObjectShim.IncrementAnswer = function (object) {
        object.answer += 1;
        // We don't return the modified object, since the reference is modified.
    };

    // Proxy an instance method call.
    JSObjectShim.Summarize = function (object) {
        return object.summarize();        
    };

})(JSObjectShim);

export { JSObjectShim };
```

```C#
using System;
using System.Runtime.InteropServices.JavaScript;
using System.Threading.Tasks;

public partial class JSObjectProxy
{
    [JSImport("JSObjectShim.CreateObject", "JSObjectShim")]
    public static partial JSObject CreateObject();

    [JSImport("JSObjectShim.IncrementAnswer", "JSObjectShim")]
    public static partial void IncrementAnswer(JSObject jsObject);

    [JSImport("JSObjectShim.Summarize", "JSObjectShim")]
    public static partial string Summarize(JSObject jsObject);

    [JSImport("globalThis.console.log")]
    public static partial void ConsoleLog([JSMarshalAs<JSType.Any>] object value);

}

public static class JSObjectUsage
{
    public static async Task Run()
    {
        await JSHost.ImportAsync("JSObjectShim", "https://localhost:7017/JSObjectShim.js");

        JSObject jsObject = JSObjectProxy.CreateObject();
        JSObjectProxy.ConsoleLog(jsObject);
        JSObjectProxy.IncrementAnswer(jsObject);
        // Note: We did not retrieve an updated object, and will see the change reflected in our existing instance.
        JSObjectProxy.ConsoleLog(jsObject);

        // JSObject exposes several methods for interacting with properties:
        jsObject.SetProperty("question", "What is the answer?");
        JSObjectProxy.ConsoleLog(jsObject);

        // We can't directly JSImport an instance method on the jsObject, but we can
        // pass the object reference and have the JS shim call the instance method.
        string summary = JSObjectProxy.Summarize(jsObject);
        Console.WriteLine("Summary: " + summary);

    }
}
// The example displays the following output in the browser's debug console:
//     {name: 'Example JS Object', answer: 41, question: null, Symbol(wasm cs_owned_js_handle): 5, summarize: ƒ}
//     {name: 'Example JS Object', answer: 42, question: null, Symbol(wasm cs_owned_js_handle): 5, summarize: ƒ}
//     {name: 'Example JS Object', answer: 42, question: 'What is the answer?', Symbol(wasm cs_owned_js_handle): 5, summarize: ƒ}
//     Summary: The question is "What is the answer?" and the answer is 42.
```

## Asynchronous Interop

Many Javascript APIs are asynchronous and signal completion through either a callback, promise, or async method.  Ignoring asynchonous capabilities is often not an option, as subsequent code may depend upon the completion of the asynchronous operation, and thus must be awaited.

JS methods using the `async` keyword or returning a promise can be awaited in C# by a method returning a Task.  Note as demonstrated below, the `async` keyword is not used on the C# method with the JSImport attribute, because it does not use the `await` keyword within it.  However, consuming code calling the method would typically use the `await` keyword and be marked as `async` as demonstrated in the `PromisesUsage` example.

Javascript with a callback, such as a `setTimeout()`, can be wrapped in a Promise before returning from Javascript.  Wrapping a callback in a promise as demonstated in `Wait2Seconds()` is only appropriate when the callback is called exactly once.  Otherwise, a C# Action can be passed to listen for a callback that may be called zero or many times, which is demonstrated in [Subscribing to JS Events](#Subscribing-to-JS-Events).

```JS
// PromisesShim.js
let PromisesShim = {};
(function (PromisesShim) {

    PromisesShim.Wait2Seconds = function () {
        // This also demonstrates wrapping a callback-based API in a promise to make it awaitable.        
        return new Promise((resolve, reject) => {
            setTimeout(() => {                
                resolve();// resolve promise after 2 seconds
            }, 2000);
        });
    };
    
    // Returning a value via resolve() in a promise
    PromisesShim.WaitGetString = function () {
        return new Promise((resolve, reject) => {
            setTimeout(() => {
                resolve("String From Resolve");// return a string via promise
            }, 500);
        });
    };

    PromisesShim.WaitGetDate = function () {        
        return new Promise((resolve, reject) => {
            setTimeout(() => {
                resolve(new Date('1988-11-24'))// return a date via promise
            }, 500);
        });
    };

    // awaitable fetch()
    PromisesShim.FetchCurrentUrl = function () {
        // This method returns the promise returned by .then(*.text())
        // and .NET in turn awaits the returned promise.
        return fetch(globalThis.window.location, { method: 'GET' })
            .then(response => response.text());
    };

    // .NET can await JS functions using the async/await JS syntax:
    PromisesShim.AsyncFunction = async function () {
        await PromisesShim.Wait2Seconds();
    };

    // A Promise.reject() can be used to signal failure, 
    // and will be bubbled to .NET code as a JSException.
    PromisesShim.ConditionalSuccess = function (shouldSucceed) {        
       
        return new Promise((resolve, reject) => {
            setTimeout(() => {
                if (shouldSucceed)                
                    resolve();// success
                else                     
                    reject("Reject: ShouldSucceed == false");// failure
            }, 500);

        });

    };
    
})(PromisesShim);

export { PromisesShim };
```


```C#
using System;
using System.Diagnostics;
using System.Runtime.InteropServices.JavaScript;
using System.Threading.Tasks;

public partial class PromisesProxy
{
    // Do not use the async keyword in the C# method signature. Returning Task or Task<T> is sufficient.

    // When calling asynchrounous JS methods, we often want to wait until the JS method completes execution.
    // For example, if loading a resource or making a request, we likely want the following code to be able to assume the action is completed.

    // If the JS method or shim returns a promise, then C# can treat it as an awaitable Task.

    // For a promise with void type, declare a Task return type:
    [JSImport("PromisesShim.Wait2Seconds", "PromisesShim")]
    public static partial Task Wait2Seconds();

    [JSImport("PromisesShim.WaitGetString", "PromisesShim")]
    public static partial Task<string> WaitGetString();

    // Some return types require a [return: JSMarshalAs...] declaration indicating
    // the type mapping returned within the promise corresponding to Task<T>.
    [JSImport("PromisesShim.WaitGetDate", "PromisesShim")]
    [return: JSMarshalAs<JSType.Promise<JSType.Date>>()]
    public static partial Task<DateTime> WaitGetDate();

    [JSImport("PromisesShim.FetchCurrentUrl", "PromisesShim")]    
    public static partial Task<string> FetchCurrentUrl();

    [JSImport("PromisesShim.AsyncFunction", "PromisesShim")]
    public static partial Task AsyncFunction();

    [JSImport("PromisesShim.ConditionalSuccess", "PromisesShim")]
    public static partial Task ConditionalSuccess(bool shouldSucceed);

}

public static class PromisesUsage
{
    public static async Task Run()
    {
        await JSHost.ImportAsync("PromisesShim", "https://localhost:7017/PromisesShim.js");
                
        Stopwatch sw = new Stopwatch();
        sw.Start();

        await PromisesProxy.Wait2Seconds();// await promise
        Console.WriteLine($"Waited {sw.Elapsed.TotalSeconds:#.0} seconds.");

        sw.Restart();
        string str = await PromisesProxy.WaitGetString();// await promise with string return
        Console.WriteLine($"Waited {sw.Elapsed.TotalSeconds:#.0} seconds for WaitGetString: '{str}'");

        sw.Restart();
        DateTime date = await PromisesProxy.WaitGetDate();// await promise with string return
        Console.WriteLine($"Waited {sw.Elapsed.TotalSeconds:#.0} seconds for WaitGetDate: '{date}'");

        string responseText = await PromisesProxy.FetchCurrentUrl();// await a JS fetch        
        Console.WriteLine($"responseText.Length: {responseText.Length}");
                
        sw.Restart();

        await PromisesProxy.AsyncFunction();// await an async JS method
        Console.WriteLine($"Waited {sw.Elapsed.TotalSeconds:#.0} seconds for AsyncFunction.");

        try {
            // Handle a promise rejection
            await PromisesProxy.ConditionalSuccess(shouldSucceed: false);// await an async JS method            
        }
        catch(JSException ex) // Catch javascript exception
        {
            Console.WriteLine($"Javascript Exception Caught: '{ex.Message}'");
        }       
        
    }
    // The example displays the following output in the browser's debug console:
    // Waited 2.0 seconds.
    // Waited .5 seconds for WaitGetString: 'String From Resolve'
    // Waited .5 seconds for WaitGetDate: '11/24/1988 12:00:00 AM'
    // responseText.Length: 582
    // Waited 2.0 seconds for AsyncFunction.
    // Javascript Exception Caught: 'Reject: ShouldSucceed == false'

}
```

## Subscribing to JS Events

.NET code can subscribe to and handle JS events by passing a C# Action to a JS method to act as a handler.  The JS shim code handles subscribing to the event.

One nuance of `removeEventListener()` is it requires a reference to the function previously passed to `addEventListener()`.  When a C# Action is passed across the interop boundary, it gets wrapped in a JS proxy object.  The consequence of this is when passing the same C# Action to both `addEventListener` and `removeEventListener`, two different JS proxy objects wrapping the Action will be generated.  These references are different, thus `removeEventListener` will not be able to find the event listener to remove.  To address this problem, the following examples wrap the C# Action in a JS function, then return that reference as a JSObject from the subscribe call to be passed later to the unsubsribe call.  Because it is returned and passed as a JSObject, the same reference is used for both calls, and the event listener can be removed.

```JS
// EventsShim.js
let EventsShim = {}; //globalThis.EventsShim || {};
(function (EventsShim) {

    EventsShim.SubscribeEventById = function (elementId, eventName, listenerFunc) {
        const elementObj = document.getElementById(elementId);

        // Need to wrap the Managed C# action in JS func (only because it is being returned)
        let handler = function (event) {            
            listenerFunc(event.type, event.target.id);// decompose object to primitives
        }.bind(elementObj);        

        elementObj.addEventListener(eventName, handler, false);        
        return handler;// return JSObject reference so it can be used for removeEventListener later
    }

    // Param listenerHandler must be the JSObject reference returned from the prior SubscribeEvent call
    EventsShim.UnsubscribeEventById = function (elementId, eventName, listenerHandler) {
        const elementObj = document.getElementById(elementId);
        elementObj.removeEventListener(eventName, listenerHandler, false);
    }

    EventsShim.TriggerClick = function (elementId) {
        const elementObj = document.getElementById(elementId);
        elementObj.click();
    }


    EventsShim.GetElementById = function (elementId) {
        return document.getElementById(elementId);
    }

    EventsShim.SubscribeEvent = function (elementObj, eventName, listenerFunc) {
        // Need to wrap the Managed C# action in JS func
        let handler = function (e) {
            listenerFunc(e);
        }.bind(elementObj);

        elementObj.addEventListener(eventName, handler, false);
        return handler;// return JSObject reference so it can be used for removeEventListener later
    }

    EventsShim.UnsubscribeEvent = function (elementObj, eventName, listenerHandler) {        
        return elementObj.removeEventListener(eventName, listenerHandler, false);
    }

    // TODO: Move to troubleshooting
    EventsShim.SubscribeEventFailure = function (elementObj, eventName, listenerFunc) {
        // It's not strictly required to wrap the C# action listenerFunc in a JS function.
        elementObj.addEventListener(eventName, listenerFunc, false);
        // However, if you need to return the wrapped proxy object you will get an error when it tries to wrap the existing proxy in an additional proxy:
        return listenerFunc; // Error: "JSObject proxy of ManagedObject proxy is not supported."
    }

    EventsShim.SubscribeEventByIdWithLogging = function (elementId, eventName, listenerFunc) {

        console.log("Begin");
        const elementObj = document.getElementById(elementId);
        console.log("elementObj:", elementObj);
        // Need to wrap the Managed C# action in JS func (only because it is being returned)
        let handler = function (event) {
            console.log(event);
            listenerFunc(event.type, event.target.id);// decompose object to primitives
        }.bind(elementObj);
        console.log(handler);

        elementObj.addEventListener(eventName, handler, false);
        console.log("done");
        return handler;// return JSObject reference so it can be used for removeEventListener later
    }
    
})(EventsShim);

export { EventsShim };
```

```C#
using System;
using System.Runtime.InteropServices.JavaScript;
using System.Threading.Tasks;

public partial class EventsProxy
{
    [JSImport("EventsShim.SubscribeEventById", "EventsShim")]
    public static partial JSObject SubscriveEventById(string elementId, string eventName, 
        [JSMarshalAs<JSType.Function<JSType.String, JSType.String>>] 
        Action<string, string> listenerFunc);

    [JSImport("EventsShim.UnsubscribeEventById", "EventsShim")]
    public static partial void UnsubscriveEventById(string elementId, string eventName, JSObject listenerHandler);

    [JSImport("EventsShim.TriggerClick", "EventsShim")]
    public static partial void TriggerClick(string elementId);

    [JSImport("EventsShim.GetElementById", "EventsShim")]
    public static partial JSObject GetElementById(string elementId);

    [JSImport("EventsShim.SubscribeEvent", "EventsShim")]
    public static partial JSObject SubscribeEvent(JSObject htmlElement, string eventName, 
        [JSMarshalAs<JSType.Function<JSType.Object>>] 
        Action<JSObject> listenerFunc);

    [JSImport("EventsShim.UnsubscribeEvent", "EventsShim")]
    public static partial void UnsubscribeEvent(JSObject htmlElement, string eventName, JSObject listenerHandler);

}

public static class EventsUsage
{
    public static async Task Run()
    {
        await JSHost.ImportAsync("EventsShim", "https://localhost:7017/EventsShim.js");

        Action<string, string> listenerFunc = (eventName, elementId) =>
            Console.WriteLine($"In C# event listener: Event {eventName} from ID {elementId}");
        
        JSObject listenerHandler1 = EventsProxy.SubscriveEventById("btn1", "click", listenerFunc);
        JSObject listenerHandler2 = EventsProxy.SubscriveEventById("btn2", "click", listenerFunc);
        Console.WriteLine("Subscribed to btn1 & 2.");
        EventsProxy.TriggerClick("btn1");
        EventsProxy.TriggerClick("btn2");

        EventsProxy.UnsubscriveEventById("btn2", "click", listenerHandler2);
        Console.WriteLine("Unsubscribed btn2.");
        EventsProxy.TriggerClick("btn1");
        EventsProxy.TriggerClick("btn2");// Doesn't trigger because unsubscribed
        EventsProxy.UnsubscriveEventById("btn1", "click", listenerHandler1);
        // Pitfall: Using a different handler for unsubscribe will silently fail.
        // EventsProxy.UnsubscriveEventById("btn1", "click", listenerHandler2); 
        

        // With JSObject as event target and event object
        Action<JSObject> listenerFuncForElement = (eventObj) =>
        {
            string eventType = eventObj.GetPropertyAsString("type");
            JSObject target = eventObj.GetPropertyAsJSObject("target");
            Console.WriteLine($"In C# event listener: Event {eventType} from ID {target.GetPropertyAsString("id")}");
        };

        JSObject htmlElement = EventsProxy.GetElementById("btn1");
        JSObject listenerHandler3 = EventsProxy.SubscribeEvent(htmlElement, "click", listenerFuncForElement);
        Console.WriteLine("Subscribed to btn1.");
        EventsProxy.TriggerClick("btn1");
        EventsProxy.UnsubscribeEvent(htmlElement, "click", listenerHandler3);
        Console.WriteLine("Unsubscribed btn1.");
        EventsProxy.TriggerClick("btn1");

    }
    // The example displays the following output in the browser's debug console:
    // Subscribed to btn1 & 2.
    // In C# event listener: Event click from ID btn1
    // In C# event listener: Event click from ID btn2
    // Unsubscribed btn2.
    // In C# event listener: Event click from ID btn1    
    // Subscribed to btn1.
    // In C# event listener: Event click from ID btn1    
    // Unsubscribed btn1.
}
```

# Performance Considerations for Interop

Marshalling of calls and the overhead of tracking objects across the interop boundary is more expensive than native .NET operations, but for moderate usage should still demonstrate acceptable performance for a typical web application.

Object proxies such as JSObject which maintain references across the interop boundary have additional memory overhead, and impact how garbage collection affects these objects.  Additionally, since memory pressure from Javascript and .NET is not shared, it is possible in some scenarios to exhaust available memory without a garbage collection being triggerred.  This risk is significant when an excessive number of large objects are referenced across interop by relatively small JSObject's, or vice versa where large .NET objects are referenced by JS proxies.  In such cases it is advisable to follow deterministic disposal patterns with `using` scopes leveraging JSObject's `IDisposable` interface.

The below benchmarks (leveraging prior example code) demonstrate that interop operations are roughly an order of magnitude slower than those that remain within the .NET boundary, but are still relatively fast.  Additionally, consider that a user's device capabilities will impact performance.

```C#
using System;
using System.Diagnostics;

public static class JSObjectBenchmark
{
    public static void Run()
    {
        Stopwatch sw = new Stopwatch();
        var jsObject = JSObjectProxy.CreateObject();
        sw.Start();
        for (int i = 0; i < 1000000; i++)
        {
            JSObjectProxy.IncrementAnswer(jsObject);
        }
        sw.Stop();
        Console.WriteLine($"JS interop elapsed time: {sw.Elapsed.TotalSeconds:#.0000} seconds at {sw.Elapsed.TotalMilliseconds / 1000000d:#.000000} ms per operation");

        var pocoObject = new PocoObject { Question = "What is the answer?", Answer = 41 };
        sw.Restart();
        for (int i = 0; i < 1000000; i++)
        {
            pocoObject.IncrementAnswer();
        }
        sw.Stop();
        Console.WriteLine($".NET elapsed time: {sw.Elapsed.TotalSeconds:#.0000} seconds at {sw.Elapsed.TotalMilliseconds / 1000000d:#.000000} ms per operation");

        Console.WriteLine($"Begin Object Creation");

        sw.Restart();
        for (int i = 0; i < 1000000; i++)
        {
            var jsObject2 = JSObjectProxy.CreateObject();
            JSObjectProxy.IncrementAnswer(jsObject2);
        }
        sw.Stop();
        Console.WriteLine($"JS interop elapsed time: {sw.Elapsed.TotalSeconds:#.0000} seconds at {sw.Elapsed.TotalMilliseconds / 1000000d:#.000000} ms per operation");

        sw.Restart();
        for (int i = 0; i < 1000000; i++)
        {
            var pocoObject2 = new PocoObject { Question = "What is the answer?", Answer = 0 };
            pocoObject2.IncrementAnswer();
        }
        sw.Stop();
        Console.WriteLine($".NET elapsed time: {sw.Elapsed.TotalSeconds:#.0000} seconds at {sw.Elapsed.TotalMilliseconds / 1000000d:#.000000} ms per operation");
    }
    
    public class PocoObject // Plain old CLR object
    {
        public string Question { get; set; }
        public int Answer { get; set; }

        public void IncrementAnswer() => Answer += 1;        
    }
}
// The example displays the following output in the browser's debug console:
// JS interop elapsed time: .2536 seconds at .000254 ms per operation
// .NET elapsed time: .0210 seconds at .000021 ms per operation
// Begin Object Creation
// JS interop elapsed time: 2.1686 seconds at .002169 ms per operation
// .NET elapsed time: .1089 seconds at .000109 ms per operation
```




