---
layout: post
title:  "Isolating WCF through Dependency Injection – Part I"
date:   2009-06-25
categories: programming .net
---

I have to admit that in the beginning I didn’t think **Inversion of Control (IoC) Containers** were that big of a deal. The idea of relying on an external agent to manage all the associations between objects in an application, sounded like it would bring more problems than advantages. Overkill. But I was wrong.

Since I began using IoC containers in my projects, I found that it naturally leads to loosely-coupled structures in the software. 
This is due to two key design principles that an IoC container will enforce you to follow:

  * Classes must explicitly **state their dependencies** with other classes as part of their public interface.
  * Classes never interact with each other directly, but only **through interfaces** that describe a set of capabilities in an abstract manner.

This decoupling contributes in making the software easily testable and ready for evolution.

### Giving up control of WCF clients

In my last project I was designing classes that would need to interact with different Web Services through WCF client proxies.
I needed to test those classes in isolation in my unit tests, without having to have my WCF service host process running in the background, so I figured I would let an instance of a WCF proxy be pushed from outside the classes through an interface, as a **dependency**, instead of creating it internally, like I would normally do.

<div class="note">
<p>
This idea fits well with the way tools like <em>svcutil</em> and <em>Visual Studio 2008</em> work when generating WCF client proxies, since they create a class as well as interfaces exposing the operations that can be invoked through the proxy.
</p>
</div>

So here is my first implementation:

```csharp
public class MailClient
{
    private IMailServiceClientChannel proxy;

    public MailClient(IMailServiceClientChannel proxy)
    {
        this.proxy = proxy;
    }

    public EmailMessage[] DownloadMessages(string smtpAddress)
    {
        // Validate the specified Email address

        Mailbox request = new Mailbox(smtpAddress);
        EmailMessage[] response = proxy.GetMessages(request)

        // Do some processing on the results

        return response;
    }
}
```

Here I am using the `IMailServiceClientChannel` interface that is automatically generated by Visual Studio when adding a service proxy with the [Add Service Reference][2] dialog. This interface contains the signatures of all the operations that are part of the service contract, as well as WCF-specific infrastructure methods to manage the proxy, like [Open][3] and [Close][4].

### Something is missing

This didn’t feel quite right. I was not treating the WCF proxy like I should since I was not closing it properly after being done with it, and not having any sort of error handling code.

However, at a second thought, I realized this wasn’t really the responsibility of my class. Since **I wasn’t creating the proxy instance myself but was instead receiving from the outside, I couldn’t really dispose it inside my method**, because it could still be needed by the caller. So here is an alternative implementation:

```csharp
public class MailClient
{
    private ChannelFactory<IMailServiceClientChannel> proxyFactory;

    public MailClient(ChannelFactory<IMailServiceClientChannel> proxyFactory)
    {
        this.proxyFactory = proxyFactory;
    }

    public EmailMessage[] DownloadMessages(string smtpAddress)
    {
        // Validate the specified Email address

        IMailServiceClientChannel proxy;

        try
        {
            proxy = proxyFactory.CreateChannel();

            Mailbox request = new Mailbox(smtpAddress);
            EmailMessage[] response = proxy.GetMessages(request)

            // Do some processing on the results

            proxy.Close();

            return response;
        }
        catch(Exception e)
        {
            proxy.Abort();

            throw new MailClientException("Failed to download Email messages", e);
        }
    }
}
```

This time instead of using a proxy instance, the class receives a **[ChannelFactory instance][5]** as an external dependency through the constructor.

<div class="note">
<p>
Channel factories in WCF are the facilities used to create the <em>pipelines</em> through which incoming and outgoing messages pass before being dispatched to the receiver or sent out over the wire.
</p>
</div>

This approach allows me to create a new proxy instances by invoking the [ChannelFactory.CreateChannel][6] method ad-hoc inside the class and to properly dispose them as soon as I no longer need them.

According to [sources inside the WCF team at Microsoft][7], it is [a best practice to create proxies by reusing the same ChannelFactory object][8], instead of creating a new proxy objects from the class generated by Visual Studio, since the cost of initializing the factory is paid only once.

This turns out to be a much better approach to assign WCF proxies to classes through **Dependency Injection (DI)** because now I can create instances of the **MailClient** class using my IoC of choice, which happens to be [Microsoft Unity][9], with a couple of lines of code:

```csharp
// Registers the ChannelFactory class with Unity
// and specifies the name of the endpoint to use
// as stated in the configuration file
var container = new UnityContainer();
container
    .RegisterType<ChannelFactory<IMailServiceClientChannel>>()
        .Configure<InjectedMembers>()
            .ConfigureInjectionFor<ChannelFactory<IMailServiceClientChannel>>(
                new InjectionConstructor("localMailServiceEndpoint"));

var client = container.Resolve();

// The object has now its dependencies already setup
// and is ready to be used
client.DownloadMessages("enrico@somedomain.com");
```

Here I’m registering the `ChannelFactory` class with the IoC container, telling it to pass the string `localMailServiceEndpoint` in the constructor whenever a new instance is created.
The ChannelFactory will use that value to look up in the **configuration file** the URL where the service is located and the protocol it should use to communicate with it, like in this sample WCF configuration:

```xml
<configuration>
    <system.serviceModel>
        <client>
            <endpoint
                name="localMailServiceEndpoint"
                address="http://localhost/mailservice"
                binding="wsHttpBinding"
                contract="IMailServiceClient"
                />
        </client>
    </system.serviceModel>
</configuration>
```

With this information in place, I’m now able to ask the container to construct an instance of the `MailClient` class, and it will automatically resolve the dependency on the WCF ChannelFactory for me.

### What about testability?

Well, the [ChannelFactory][5] is a concrete class and, although certanly possible, **can’t easily be faked out (through mocks or stubs)** by using one of the common mocking isolation frameworks available out there, since it requires the presence of a configuration file with the proper WCF-specific settings in order to work correctly. This makes the MailClient class still hard to test in isolation.

In my [next post][10] I will explain a way to modify this sample to be able to write unit test for classes that use WCF proxies.

/Enrico

[2]: http://msdn.microsoft.com/en-us/library/bb386382.aspx
[3]: http://msdn.microsoft.com/en-us/library/system.servicemodel.iclientchannel.open.aspx
[4]: http://msdn.microsoft.com/en-us/library/system.servicemodel.iclientchannel.close.aspx
[5]: http://msdn.microsoft.com/en-us/library/ms576132.aspx
[6]: http://msdn.microsoft.com/en-us/library/ms575250.aspx
[7]: http://blogs.msdn.com/wenlong/default.aspx
[8]: http://blogs.msdn.com/wenlong/archive/2007/10/27/performance-improvement-of-wcf-client-proxy-creation-and-best-practices.aspx
[9]: http://www.codeplex.com/unity
[10]: /2009/07/02/isolating-wcf-through-dependency-injection-part-ii/
