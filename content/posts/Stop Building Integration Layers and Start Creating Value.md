---
title: Stop Building Integration Layers and Start Creating Value
tags: [toreview]
coverImage: 'https://academy.graftcode.com/_next/image?url=%2Fquick-start-image.png&w=3840&q=75'

---

# Stop Building Integration Layers and Start Creating Value

## Introduction
When building a new application, sooner or later you face the challenge: how to make all its components communicate with each other? The frontend needs to exchange data with the backend. Different backend services must share information between themselves. On top of that, there are external libraries, modules, services, or systems that also need to be integrated.

Traditionally, this means building additional integration layers – controllers, hooks, middleware, or adapters. It’s a time-consuming process that distracts from what really matters: business logic and meeting customer requirements.

Now imagine you could put all of this aside. Instead of writing endless layers of “glue code,” you can use an external component directly in your IDE – as if you had written it yourself, in the same project. With full autocompletion support, parameter typing, and all the conveniences of working with native code.

This is the revolution brought by **Graftcode**. It’s a technology integrator that eliminates the need for manually building bridges between different environments and languages. With it, every part of your application – regardless of the technology it was written in – becomes instantly accessible and ready to use.

In this article, we’ll show you how **Graftcode** changes the rules of the game. How it allows you to forget about tedious integration and focus on what truly matters: creating value for your users and customers.

[toc]

## Classic approach vs. Graftcode
The traditional approach to building applications requires creating intermediary layers that allow individual components to communicate with each other. GraftCode makes it possible to simply skip this entire step. In the picture below, you can see that with Graftcode, 30% of code that was created to communicate between differen services, can be made obsolete.

![image_1757534682973](https://hackmd.io/_uploads/Bk-abKtjlg.png)

Graftcode changes the integration story from:
* implement controllers
* implement request/response DTOs
* forward calls to business logic
* implement client
* parse responses to local DTOs/models
* invoke methods on client
* monitor for changes
* keep updating each layer with each update

To:
* expose public methods on business logic or simple plain object facades by adding Graftcode Gateway to your service
* run a package manager command that will create a strongly-typed and always in sync client that connects to an external service. This is something we call **Graft**.

Now, your application looks like this:

![image_1757449669716](https://hackmd.io/_uploads/HkzOGKtilg.png)

## And what does it look like in practice?

At the core of the solution lies **Graft** – a native, dynamically generated library that handles communication between your code and external components, such as a backend written in .NET or any other technology.

All data exchange is powered by our proprietary Hypertube™ protocol. This is what makes Graftcode both fast and lightweight. Hypertube™ operates at the level of native runtime integration and leverages binary messaging. As a result:

* performance of calls is 30% faster compared to WebService or gRPC,
* the amount of code that connects services is reduced by 30–60%,
* your business logic remains fully decoupled from the communication layer.

In practice, this means that when you invoke a method in an external component, you use it as if it were part of your own project – with no need for adapters, controllers, or additional “glue code.”

## How to create a Graft?

Let’s assume you’re building an application in React. Normally, when you want to use an external library, you simply run npm install – and here it looks almost the same. The only difference is in the command syntax:

```bash!
npm install --registry https://grft.dev/some-graftcode-project @graft/nuget-my-backend-lib
```

In practice, getting started with Graftcode feels no different from installing a typical *npm package* – with the crucial distinction that you can now directly use methods from libraries written in **.NET** (or any other technology) as if they were native parts of your **JavaScript** project.

Importantly, the exact same mechanism also applies inside your backend. This means that different backend components – written in different languages or running in different environments – can communicate with each other in exactly the same way. It doesn’t matter whether you’re invoking code from the frontend or integrating backend microservices – the pattern remains identical and just as simple.

Let’s assume your backend is written in .NET, and at the same time you want it to communicate with a chatbot built in Python. In a traditional approach, you would need to write a considerable amount of code – define communication contracts, build an API, or add another middleware – just to make these two worlds talk to each other.

With Graftcode, all of this extra work becomes unnecessary. Both components communicate as if they were written in the same language and running in the same environment. You simply call a Python method from your .NET code – with full type support, IDE autocompletion, and no additional bridges to maintain.

The benefit is obvious: instead of wasting time on integration and maintaining code responsible for communication between services, you can fully focus on building business functionality – the part that truly matters for your users and clients.

## How can I start grafting?

Imagine you’re building an application in .NET, but you’d like to integrate the popular ChatterBot from the Python ecosystem. In a traditional approach, you would need to prepare a separate Python script and expose a simple API just so your application could communicate with it. That means extra code, additional setup, and more components to maintain.

With GraftCode, the whole process becomes much simpler. You can call ChatterBot directly from your .NET project – as if it were a native part of your code. No extra integration layers, no custom APIs, just clean logic and fast execution.

### Creating .NET app

Let’s start by creating a new class library in .NET, which will serve as our backend. In the current folder, you can do this with the following command:

```bash!
dotnet new classlib -n MyBackendLib
```

This will generate a project called MyBackendLib, which we’ll soon connect to the frontend application through GraftCode.

### Creating Chatterbot Graft

Now it’s time to connect our project with a library written in Python. In this example, we’ll reference it directly, without writing any intermediate code or additional integration layers.

```bash!
dotnet add package -s https://grft.dev graft.pypi.chatterbot --version 1.2.7
```

### Use Chatterbot in .NET app

```csharp=
using Graft.Pypi.Chatterbot;

namespace MyBackendLib    
{
    public class MyChatbot
    {
        Chatbot _chatbot;
        
        public MyChatbot()
        {
            _chatbot = ChatBot('Ron Obvious');
            var trainer = ChatterBotCorpusTrainer(_chatbot);
            trainer.Train("chatterbot.corpus.english");
        }
        
        public string GetAnswer(string question)
        {
            return chatbot.GetResponse(question);
        }
    }
}
```

### Expose your .NET app

To expose this service to the outside world, we've created a dockerfile. This file tells Docker how to build and run your service with GraftCode Gateway. The dockerfile is simple and looks like this:

```dockerfile=
FROM pladynski/myrepo:graftcode
COPY ./src/MyBackendLib/bin/Release/net8.0/publish/ /usr/app/
CMD ["/usr/app/gg", "--runtime", "netcore", "--modules", "/usr/app/MyBackendLib.dll", "--GV"]
```

Now, let's build and run your Docker container with the following commands. Please note that you need to have Docker installed and running on your machine to execute these commands:

```dockerfile=
docker login -u pladynski@sdncenter.pl -p graftcode
dotnet build .\MyBackendLib.csproj
dotnet publish .\MyBackendLib.csproj
docker build --no-cache --pull -t mychatbot:test .
docker run -d -p 80:80 -p 81:81 --name graftcode_demo mychatbot:test
```

### Create React app

Let's create simple react application:
```bash!
npx create-react-app my-app
cd my-app
```

Now we can add our library **Graft**:
```bash!
# do weryfikacji, jak powinna wyglądać właściwa komenda
npm install --registry https://grft.dev/graftcode-demo__fa55e1ed-8373-4600-95eb-d028f21d2451 @graft/nuget-be
```

Open `src\App.jsx` and add on top of the file:
```javascript=
import { useState } from "react";
import { setConfig, MyBackendLib } from '@graft/nuget-mybelib'
```

Next, configure how the frontend connects to the backend service. Paste this snippet right below imports (it can be configured via environment variables, config files or shorten syntax too):

```javascript=
// adres ws do dodania
const config = {runtimes: {netcore: [{ name: "default", channel: {
        type: "webSocket",
        host: "", },}],},
};

setConfig(config)
```

Now we can call our backend method:
```javascript!
const answerPromise = MyChatbot.GetAnswer("What is the Answer to the Ultimate Question of Life, the Universe, and Everything?");
```

We also need to show the result:

```javascript=
function App() {
  const [data, setData] = useState(0);

  answerPromise.then(setData);

  return <h1>The Utlimate Answer is: {data}</h1>;
}
```

Final code should look like below:

```javascript=
import { useState } from "react";
import { setConfig, MyBackendLib } from '@graft/nuget-mybelib'

// adres ws do dodania
const config = {runtimes: {netcore: [{ name: "default", channel: {
        type: "webSocket",
        host: "", },}],},
};

setConfig(config)

const answerPromise = MyChatbot.GetAnswer("What is the Answer to the Ultimate Question of Life, the Universe, and Everything?");

function App() {
  const [data, setData] = useState(0);

  answerPromise.then(setData);

  return <h1>The Utlimate Answer is: {data}</h1>;
}

export default App
```

## Conclusion

GraftCode takes away one of the least enjoyable parts of development – writing endless integration layers and boilerplate code. Instead of struggling with communication details, you can focus on what truly brings joy – building logic, experimenting, and creating valuable features.

Rather than spending hours aligning protocols and maintaining helper code, you simply call a method from another library as if it were part of your own project. More joy in coding, less frustration, and a faster path from idea to working functionality.