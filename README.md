# AI Web Chat App — Sample Repository

This repository contains a sample AI chat application project. The working application is inside the `ChatApp9/` folder. This root README provides a short overview and points to the project-specific README.

Overview

- Project: `ChatApp9` — a .NET-based sample that demonstrates an AI chat app built to show how to chat with custom data using an AI language model.

- The project is an early-preview template; follow the project README for configuration and usage details.

Where to start

1. See `ChatApp9/README.md` for provider configuration, API key instructions, and background information about the sample.

2. Open the solution file `ChatApp9.sln` with Visual Studio or use the .NET CLI from the repository root.

Quick commands (Windows PowerShell)

```powershell
# Restore and build
dotnet restore ChatApp9.sln
dotnet build ChatApp9.sln

# Run the app (from repository root)
dotnet run --project ChatApp9/ChatApp9.csproj
```

Notes

- Before running, configure any required API keys or endpoints as described in `ChatApp9/README.md` (for example, set the `GitHubModels:Token` in user secrets if you choose GitHub Models).

- The sample includes example PDF data in `ChatApp9/wwwroot/Data/` used by the ingestion pipeline.

License & Feedback

- This is a sample project for demonstration. See `ChatApp9/README.md` for more details and the survey link if you'd like to provide feedback.

## Hugging Face MCP integration (tutorial)

This section shows how to connect the sample app to a Hugging Face MCP server so the app can call Hugging Face models using the Model Context Protocol (MCP). The steps below are formatted as a short tutorial and expand the earlier notes.

Prerequisites

- A Hugging Face account with an access token that has the required permissions.

- The `ModelContextProtocol` library (the tutorial uses `0.3.0-preview.3` as an example).

- The sample project (this repo) or a new .NET Web/Blazor app.

Step 1 — Create or open the AI web app

- If you don't yet have a project, create one (example: a Blazor Server app):

```powershell
# from repository root or a directory where you want the app
dotnet new blazorserver -o ChatApp9
```

Step 2 — Add the MCP package

- Add the `ModelContextProtocol` package to your project. In the project file (`ChatApp9.csproj`) add:

```xml
<PackageReference Include="ModelContextProtocol" Version="0.3.0-preview.3" />
```

Or use the .NET CLI to add the package:

```powershell
dotnet add ChatApp9/ChatApp9.csproj package ModelContextProtocol --version 0.3.0-preview.3
```

Step 3 — Register a Hugging Face MCP client in `Program.cs`

- Open `ChatApp9/Program.cs` and register an MCP client that connects to the Hugging Face MCP endpoint. The example below shows how to register a keyed singleton service named `HuggingFaceMCP` so multiple components can request the same client by key.

Notes:

- This snippet expects your Hugging Face access token to be available from configuration (for example, environment variables or user secrets) under the key `HF_ACCESS_TOKEN`.

- The sample uses Server-Sent Events (SSE) transport to talk to the MCP server.

Add the following inside your `Program.cs` after `var builder = WebApplication.CreateBuilder(args);` and after any other `builder.Services` registrations:

```csharp
// add mcp client to connect to Hugging Face MCP Server from the GetHuggingFaceMcpClient method
// register the tools as a singleton so that they can be reused across requests
builder.Services.AddKeyedSingleton<IMcpClient>("HuggingFaceMCP", (sp, _) =>
{
 var deploymentName = "gpt-4.1-mini";

 // read the hfAccessToken from the configuration
 var hfAccessToken = builder.Configuration["HF_ACCESS_TOKEN"];

 // create MCP Client using Hugging Face endpoint
 var hfHeaders = new Dictionary<string, string>
 {
  { "Authorization", $"Bearer {hfAccessToken}" }
 };

 var clientTransport = new SseClientTransport(
  new()
  {
   Name = "HF Server",
   Endpoint = new Uri("https://huggingface.co/mcp"),
   AdditionalHeaders = hfHeaders
  });
 var client = McpClientFactory.CreateAsync(clientTransport).GetAwaiter().GetResult();
 return client;
});
```

Step 4 — Use the MCP client in your chat component (`Chat.razor`)

- Inject the keyed MCP client into the Blazor component and load available tools at initialization. The snippet below follows the pattern you shared, but with a best-practices improvement: prefer `OnInitializedAsync` to avoid blocking the UI thread.

In `Chat.razor` add the injected property near other `[Inject]` members:

```csharp
[Inject(Key = "HuggingFaceMCP")]
public IMcpClient HFMcpClient { get; set; } = default!;
```

Then, use an async initializer to list tools and populate your chat options. Replace or update your existing initializer with:

```csharp
protected override async Task OnInitializedAsync()
{
 messages.Add(new(ChatRole.System, SystemPrompt));

 // get the tools from the MCP client
 var tools = await HFMcpClient.ListToolsAsync();

 // ensure chatOptions.Tools is initialized and add MCP tools if available
 chatOptions.Tools ??= new List<AITool>();
 foreach (var t in tools)
 {
  chatOptions.Tools.Add(t);
 }

 // add the search tool (example using an AI function factory)
 chatOptions.Tools.Add(AIFunctionFactory.Create(SearchAsync));
}
```

Important notes and best practices

- Use `dotnet user-secrets` or environment variables to store secrets locally instead of hard-coding tokens. Example (from the repository root):

```powershell
cd ChatApp9
dotnet user-secrets init
dotnet user-secrets set "HF_ACCESS_TOKEN" "your_hf_token_here"
```

- Consider using `OnInitializedAsync` instead of `OnInitialized` to make the initialization asynchronous and avoid potential UI threading issues.

- Validate that `HFMcpClient` is not null before calling client methods. If the registration fails, you'll get helpful exceptions on app startup.

- The `deploymentName` variable in the example shows a chosen model deployment; make sure the deployment name exists and is accessible for your token.

Quick test run

1. Ensure `HF_ACCESS_TOKEN` is set (via environment variable or user secrets).
2. Build and run the app:

```powershell
dotnet build ChatApp9.sln
dotnet run --project ChatApp9/ChatApp9.csproj
```

3. Open the running app in your browser and exercise the chat UI — the MCP client will list tools and allow the app to call the model through the Hugging Face MCP endpoint.

Troubleshooting

- If the MCP client fails to connect, check that the token is correct and that `https://huggingface.co/mcp` is reachable from your environment.
- If tools are not listed, verify that the MCP server supports the `ListTools` operation and that your token has permissions to access the deployment.
- Look at the application logs or console output for exceptions thrown during `McpClientFactory.CreateAsync(...)` or while calling `ListToolsAsync()`.

If you'd like, I can also:

- Add an example feature flag to enable/disable Hugging Face integration at runtime.
- Create a small helper class to encapsulate MCP client creation and registration for cleaner `Program.cs`.
