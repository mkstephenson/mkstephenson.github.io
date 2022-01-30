---
title: "Windows 10 IoT TCP Server"
date: "2017-10-15T13:59:56+02:00"
---

In the past I've been tinkering quite a bit with Windows 10 IoT and one thing that irritated me was the fact that back then you couldn't use the System.Net.HttpListener inside UWP Apps.

So why didn't I use NodeJS? Well, for one, it's JavaScript, which I avoid if I have the possibility of using C#. But the main reason is that the tooling around NodeJS UWP Apps was quite complex and a pain to get running smoothly, especially on Windows 10 IoT.

So I started looking around for a C# based solution but I only found solutions which suggested creating your own HTTP server based on the StreamSocketListener, so I did just that. There is a brilliant tutorial to be found at https://incredibits.io/project/windows-10-iot-internet-of-things-tips/windows-10-iot-raspberry-pi-web-server which I ended up taking and modifying slightly to make it easier to use. I took the base server, wrapped it it's own class and added a delegate which would be called when a request came in. For simplicity's sake, I simply dump the entire content of the TCP request into the delegate when a request comes in and leave processing of the request up to the implementer. I also added in some rudimentary logging which could be used to observe the incoming requests through the Realtime ETW tracing page which can be found in Windows 10 IoT.

Anyway, here is the code for the basic TCP server:

```cs
using System;
using System.IO;
using System.Runtime.InteropServices.WindowsRuntime;
using System.Text;
using Windows.Foundation.Diagnostics;
using Windows.Networking.Sockets;
using Windows.Storage.Streams;
namespace PowerSwitch
{
    public delegate string TcpRequestReceived(string request);
    // Basic Server which listens for TCP Requests and provides the user with the ability to craft own responses as strings
    public sealed class TcpServer
    {
        private StreamSocketListener fListener;
        private const uint BUFFER_SIZE = 8192;
        private LoggingChannel fLoggingTarget;
        private LoggingSession fLoggingSession;
        private Guid fTargetLoggingSessionGuid = new Guid(&lt;GUID_TEXT&gt;);
        public TcpRequestReceived RequestReceived { get; set; }
        public TcpServer() { }
        public void Initialise(int port)
        {
            fLoggingTarget = new LoggingChannel("HttpRequests", null, fTargetLoggingSessionGuid);
            fLoggingSession = new LoggingSession("HttpsRequestsLoggingSession");
            fLoggingSession.AddLoggingChannel(fLoggingTarget);
            fListener = new StreamSocketListener();
            fListener.BindServiceNameAsync(port.ToString());
            fListener.ConnectionReceived += async (sender, args) =>
            {
                HandleRequest(sender, args);
            };
        }
        private async void HandleRequest(StreamSocketListener sender, StreamSocketListenerConnectionReceivedEventArgs args)
        {
            StringBuilder request = new StringBuilder();
            using (IInputStream input = args.Socket.InputStream)
            {
                byte[] data = new byte[BUFFER_SIZE];
                IBuffer buffer = data.AsBuffer();
                uint dataRead = BUFFER_SIZE;
                while (dataRead == BUFFER_SIZE)
                {
                    await input.ReadAsync(buffer, BUFFER_SIZE, InputStreamOptions.Partial);
                    request.Append(Encoding.UTF8.GetString(data, 0, data.Length));
                    dataRead = buffer.Length;
                }
            }
            string requestString = request.ToString();
            fLoggingTarget.LogEvent("Request: " + requestString, null, LoggingLevel.Information);
            string response = RequestReceived?.Invoke(requestString);
            fLoggingTarget.LogEvent("Response: " + response, null, LoggingLevel.Information);
            using (IOutputStream output = args.Socket.OutputStream)
            using (Stream responseStream = output.AsStreamForWrite())
            {
                MemoryStream body;
                if (response != null)
                {
                    body = new MemoryStream(Encoding.UTF8.GetBytes(response)); 
                }
                else
                {
                    body = new MemoryStream(Encoding.UTF8.GetBytes("No response specified"));
                }
                var header = Encoding.UTF8.GetBytes($"HTTP/1.1 200 OK\r\nContent-Length: {body.Length}\r\nConnection: close\r\n");
                await responseStream.WriteAsync(header, 0, header.Length);
                await body.CopyToAsync(responseStream);
                await responseStream.FlushAsync();
            }
        }
    }
}
```

As you can see, this is a simple class which you instantiate, call the Initialise method then add a handler to the RequestReceived delegate. An implementation inside a Windows 10 IoT background task could look something like this.

```cs
public sealed class StartupTask : IBackgroundTask
{
    private BackgroundTaskDeferral fDef;
    private TcpServer fServer;
    public void Run(IBackgroundTaskInstance taskInstance)
    {
        fDef = taskInstance.GetDeferral();
        fServer = new TcpServer();
        fServer.RequestReceived = (request) =>
        {
            return "&lt;html&gt;&lt;body&gt;" + request + "&lt;/body&gt;&lt;/html&gt;";
        };
        fServer.Initialise(2501);
    }
}
```

This very simple example first grabs the deferral of the task instance (so that the application doesn't end immediately once the server is started), creates an instance of the server, adds the handler to the RequestReceived callback and then initialises it on port 2501 (Ghost in the Shell reference, I often use it when I don't want to use one of the standard ones like 8080 or 8888).

This will start the server listening on port 2501 and any requests will be routed through the request received handler. In this case it will just wrap the request in HTML tags, then send it back to the client.