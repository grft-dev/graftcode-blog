---
title: Monolith to Microservices Without Rewriting Your Integration Layers

---

Monolith to Microservices Without Rewriting Your Integration Layers

# Introduction

Moving from a monolithic architecture to microservices is rarely a clean rewrite. In real-world systems, the monolith often contains years of business logic, hard-won domain knowledge, and integrations that cannot simply be “turned off” overnight. The challenge is not deciding whether to migrate, but how to evolve the system without disrupting existing functionality, teams, or release cycles.

This is where many migration strategies struggle. Splitting a monolith usually means duplicating logic, rewriting components in parallel, or introducing temporary bridges that increase complexity instead of reducing it. As a result, teams either delay the migration indefinitely or commit to a risky big-bang refactor.

Graftcode approaches this problem from a different angle. Instead of forcing an immediate architectural split, it enables monolithic and microservice components to coexist and interoperate seamlessly. This allows teams to extract services incrementally, reuse existing code where it still makes sense, and gradually shift responsibilities out of the monolith — without breaking contracts or slowing down delivery.

In this article, we’ll explore how Graftcode can act as a practical transition layer between a monolithic system and a microservices architecture, making the migration process safer, more predictable, and easier to control.

[toc]

# Simple monolith

Let’s start by creating a simple monolithic application using .NET 9.0.
The solution consists of two projects: Monolith2MSSample (console app), which acts as a facade, and CurrencyConverter (class library), a module that we will later extract into a separate service.

Let's create `Facade` class in Monolith2MSSample app:
```csharp=
namespace Monolith2MSSample;

public static class Facade
{
    public static double GetCost(double amount, double value)
    {
        return amount * value;
    }

    public static double GetConvertedCost(double amount, double value)
    {
        var cost = amount * value;
        return CurrencyConverterService.ConvertEuroToUSD(cost);
    }
}

```

CurrencyConverter code:
```csharp=
namespace CurrencyConverter;

public static class CurrencyConverterService
{
    public static double ConvertEuroToUSD(double amount)
    {
        return amount * 1.19; 
    }
}
```
For the facade to use the functionality provided by `CurrencyConverter`, it must be added as a project reference in `csproj`:

```xml!
<ItemGroup>
  <ProjectReference Include="..\CurrencyConverter\CurrencyConverter.csproj" />
</ItemGroup>
```

And add `using` in `Facade` class:

```csharp!
using CurrencyConverter;
```

At this point, we have two components that could potentially be turned into **separate services**. However, doing so would require writing a significant amount of additional code to enable communication and interaction between them.

This is exactly the point where many migrations slow down or become overly complex. Introducing service boundaries usually means redesigning contracts, implementing communication layers, and handling serialization, versioning, and error propagation — all before any real architectural value is delivered.

Graftcode removes much of this overhead by allowing components to interact across service boundaries without requiring an immediate rewrite of integration logic. Instead of building custom communication code upfront, existing components can be reused and gradually moved into independent services while preserving their original interfaces and behavior.

# How to use Graftcode to create microservice?

Let's build our `CurrencyConverter`:
```bash!
dotnet build
```
Get `CurrencyConverter.dll` from `bin` directory. Download newest `gg` release from [Releases](https://github.com/grft-dev/graftcode-gateway/releases/) and copy `gg` and your `dll` file to **one directory**. Now you can run `gg`:
```bash!
./gg
```
It should detect your dll and runtime type. For more info see our [Readme](https://github.com/grft-dev/graftcode-gateway/blob/main/README.md).
You can access your Graftvision here: `http://localhost:81/GV`

![image](https://hackmd.io/_uploads/BJE14HLIbg.png)

There you can select package type (NuGet in our case) and add graft to your Facade:

```bash!
dotnet add package -s https://grft.dev/0d17c593-1e75-4695-87a4-e75d5583b25f__free graft.nuget.CurrencyConverter --version 1.0.0.0
```

Your `csproj` should look like this:
```xml=
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <!--<ItemGroup> REMOVE THIS PART
    <ProjectReference Include="..\CurrencyConverter\CurrencyConverter.csproj" />
  </ItemGroup>-->

  <ItemGroup>
    <PackageReference Include="graft.nuget.CurrencyConverter" Version="1.0.0.0" />
  </ItemGroup>

</Project>
```
Our `Facade` class must be changed to this:

```csharp=
using graft.nuget.CurrencyConverter; // <-- new using

namespace Monolith2MSSample;

public static class Facade
{
    [..]
}
```

Now in your `Program.cs` you can call `Facade`:

```csharp=
namespace Monolith2MSSample;

internal class Program
{
    private static void Main(string[] args)
    {
        GraftConfig.Host = "ws://localhost:80/ws";
        Console.WriteLine(Facade.GetConvertedCost(1, 2));
    }
}
```
In your console you should see `2.38` as a result.
At this stage, we now have two separate modules. Each of them can be deployed and run independently, for example inside its own Docker container, and invoked directly from the frontend.

# Let's create some frontend

Let’s add a simple example frontend and connect it to the facade. First, let’s modify the console application’s `.csproj` file.

```xml=
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <!--<OutputType>Exe</OutputType> REMOVE THIS PART -->
    <TargetFramework>net9.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="graft.nuget.CurrencyConverter" Version="1.0.0.0" />
  </ItemGroup>

</Project>
```

We no longer need the `Program.cs` file. Let's remove it. Without `Program.cs` we need to add static constructor to `Facade` class:

```csharp=
using graft.nuget.CurrencyConverter;

namespace Monolith2MSSample;

public static class Facade
{
    static Facade()
    {
        GraftConfig.Host = "ws://localhost:80/ws";
    }

    [..]
}

```

Now we can publish this library:

```bash!
dotnet publish -c Release
```

Copy `Monolith2MSSample.dll`, `graft.nuget.CurrencyConverter.dll` and any other DLLs  with `gg` to separate directory, where you can run it:

```bash!
./gg --port 90 --httpPort 91 --modules .\Monolith2MSSample.dll
```

The default ports are already in use by the first instance of `gg`, so we need to provide additional parameters to avoid port conflicts:

* `--modules` : you need to indicate which file should be hosted by `gg`
* `--port` : port used for communication (default: 80), in our case **90**
* `--httpPort` : port used for hosting Graftcode Vision (default:81), **91** in our example

At this stage we can build very simple frontend app:
```bash!
mkdir article-sample-client
cd article-sample-client
npm init -y
npm install --registry https://grft.dev/73bb839b-783f-4a4e-a4ff-02a031964bc1__free @graft/nuget-Monolith2MSSample@1.0.0
```

Last command `npm install --regisrty ...` can be retrieved directly from `localhost:91/GV`

Code for `app.js`:
```javascript=
import { Facade, GraftConfig } from '@graft/nuget-Monolith2MSSample';

GraftConfig.host = "ws://localhost:90/ws"; // your facade address

const amountInput = document.getElementById("amount");
const valueInput = document.getElementById("value");
const costOutput = document.getElementById("cost");
const convertedCostOutput = document.getElementById("convertedCost");
const button = document.getElementById("calculate");

button.addEventListener("click", async () => {
  try {
    const amount = Number(amountInput.value);
    const value = Number(valueInput.value);

    const cost = await Facade.GetCost(amount, value);
    const convertedCost = await Facade.GetConvertedCost(amount, value);

    costOutput.textContent = String(cost);
    convertedCostOutput.textContent = String(convertedCost);
  } catch (e) {
    console.error(e);
    costOutput.textContent = "ERROR";
    convertedCostOutput.textContent = e?.message ?? String(e);
  }
});
```

And simple `index.html`:
```html=
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Graftcode Cost Calculator</title>
  <style>
    body { font-family: sans-serif; max-width: 480px; }
    label { display: block; margin-top: 8px; }
    button { margin-top: 12px; }
    .result { margin-top: 16px; }
  </style>
</head>
<body>
  <h1>Cost Calculator</h1>

  <label>
    Amount
    <input id="amount" type="number" step="any" />
  </label>

  <label>
    Value
    <input id="value" type="number" step="any" />
  </label>

  <button id="calculate">Calculate</button>

  <div class="result">
    <div>Cost: <strong id="cost">—</strong></div>
    <div>Converted cost: <strong id="convertedCost">—</strong></div>
  </div>

  <script type="module" src="./app.js"></script>
</body>
</html>
```

`package.json` content:
```json=
{
  "name": "fe",
  "version": "1.0.0",
  "description": "",
  "main": "app.js",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "type": "module",
  "dependencies": {
    "@graft/nuget-Monolith2MSSample": "^1.0.0",
    "javonet-nodejs-sdk": "^2.6.13"
  },
  "devDependencies": {
    "vite": "^7.3.1"
  }
}

```

Now you can start your app:

```bash!
npm run dev
```

Under `localhost:3000` you should see simple calculator. It may look like this:

![image](https://hackmd.io/_uploads/rkaKLYPIZg.png)


# Conclusion

Migrating from a monolithic architecture to microservices is not a single technical decision, but a long-term evolutionary process. Systems, teams, and business requirements rarely change at the same pace, which is why rigid, all-or-nothing migration strategies so often fail.

Graftcode enables a more pragmatic approach. By allowing different parts of a system to run side by side — regardless of language, runtime, or architectural style — it removes the pressure to fully commit upfront. Teams can extract services when it makes sense, keep proven components in place, and evolve the architecture without interrupting delivery or increasing operational risk.

Rather than treating the monolith as something to be replaced, Graftcode turns it into a foundation that can be gradually reshaped. This makes it possible to move toward microservices with confidence, maintaining stability today while building flexibility for tomorrow.

If your goal is not just to adopt microservices, but to do so safely, incrementally, and on your own terms, Graftcode provides the missing link between where your system is now and where you want it to be.