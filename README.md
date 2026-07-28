# Semantic Kernel Console App

A minimal console chat agent built with [Microsoft Semantic Kernel](https://learn.microsoft.com/semantic-kernel/), backed by [GitHub Models](https://github.com/marketplace/models). It demonstrates **automatic function calling**: the model decides on its own when to invoke your C# code to read or change application state.

The sample domain is a set of mock light bulbs. Ask it to turn a light on and it will call into `LightsPlugin` to do it.

---

## Prerequisites

- .NET 10 SDK
- A GitHub account (Copilot Free is enough)
- A **fine-grained** personal access token with **Account permissions → Models → Read-only**

> Classic PATs may work, but GitHub documents `models:read` — a fine-grained permission — as the requirement. Don't grant `repo` scope; this app never touches a repository.

## Setup

Create your token at [github.com/settings/personal-access-tokens](https://github.com/settings/personal-access-tokens), then put it in `Properties/launchSettings.json`:

```json
{
  "profiles": {
    "Semantic Kernal Console App": {
      "commandName": "Project",
      "environmentVariables": {
        "GITHUB_TOKEN": "your_token_here"
      }
    }
  }
}
```

Or set it in the shell instead:

```powershell
$env:GITHUB_TOKEN = "your_token_here"
```

`launchSettings.json` is listed in `.gitignore` because it holds a live credential. **Keep it that way** — Visual Studio's stock `.gitignore` does *not* exclude this file, and it is one of the most common ways PATs end up in public repositories.

## Run

```powershell
dotnet run --project "Semantic Kernal Console App.csproj"
```

Press Enter on an empty line to exit.

```
User > Please turn on the porch light
Assistant > The Porch Light has been turned on.

User > what's the status of all the lamps?
Assistant > - Table Lamp: Off
            - Porch Light: On
            - Chandelier: Off
```

## Project structure

| File | Purpose |
|---|---|
| `Program.cs` | Kernel setup, chat loop, conversation history |
| `LightsPlugin.cs` | The native plugin — two `[KernelFunction]`s over mock light data |
| `Properties/launchSettings.json` | Holds `GITHUB_TOKEN` (git-ignored) |

## How it works

**1. The kernel connects to GitHub Models.** GitHub Models is OpenAI-compatible, so it uses `AddOpenAIChatCompletion` with a custom endpoint — *not* `AddAzureOpenAIChatCompletion`, despite the endpoint historically living on Azure infrastructure.

```csharp
var builder = Kernel.CreateBuilder().AddOpenAIChatCompletion(modelId, endpoint, apiKey);
```

**2. The plugin is registered as a tool.** Reflection turns each `[KernelFunction]` into a schema the model can see. The `[Description]` attributes are not comments — they are the entire basis on which the model decides what to call, so their wording directly affects accuracy.

**3. `FunctionChoiceBehavior.Auto()` lets the model choose.** Semantic Kernel then invokes the chosen method, feeds the result back, and loops until the model returns plain text.

Note what this means: registering a plugin **adds** a capability, it does not **restrict** the model. `gpt-4o-mini` remains a general-purpose model and will happily answer questions unrelated to lights. To scope it, add a system message via `history.AddSystemMessage(...)` — though that is a soft steer, not a security boundary.

**4. One user message can cost several API requests.** "Turn on all the lights" typically means: one request to list lights, one to change them, one to summarize. This matters for the rate limits below.

## Limits

`gpt-4o-mini` is a **Low**-tier model on GitHub Models. On the free plan:

| Limit | Value |
|---|---|
| Requests / minute | 15 |
| Requests / day | 150 |
| Tokens **in** per request | 8,000 |
| Tokens **out** per request | 4,000 |
| Concurrent requests | 5 |

Two consequences worth understanding:

- **The 8,000-token input cap is the binding constraint.** `gpt-4o-mini` natively supports 128k context; GitHub Models allows 8k. The full `ChatHistory` plus every function schema is re-sent on *every* request, so usage grows each turn. This app does not trim history — a long session will eventually exceed the cap. Add a `ChatHistoryTruncationReducer` if that becomes a problem.
- **~150 requests/day is really ~50 messages/day**, given the multi-request pattern described above.

GitHub Models is a prototyping tier with no SLA. Move to Azure OpenAI before building anything real on it.

## Notes

- Logging is set to `LogLevel.Trace`, which prints the full serialized `ChatHistory` on every call. It's useful for seeing exactly when function calls happen — watch for `FunctionCallContent` in the output. Lower it to `Information` once the novelty wears off; the per-call token counts still print at that level.
- Rate-limit responses (HTTP 429) are not handled and will terminate the chat loop.

## References

- [Semantic Kernel documentation](https://learn.microsoft.com/semantic-kernel/)
- [GitHub Models catalog](https://github.com/marketplace/models)
- [GitHub Models rate limits](https://docs.github.com/en/github-models/use-github-models/prototyping-with-ai-models#rate-limits)
