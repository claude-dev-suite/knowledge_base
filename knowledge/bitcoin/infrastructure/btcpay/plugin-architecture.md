# BTCPay Server Plugin Architecture - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/btcpay`.
> Canonical source: https://docs.btcpayserver.org/Development/Plugins/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/btcpay/SKILL.md

## Concept

BTCPay Server's plugin system lets third parties extend the .NET host with
new payment methods (Liquid, Monero, USDT-Liquid), apps, integrations
(Shopify, WooCommerce hooks), and management screens - without forking the
core. Plugins are compiled .NET DLLs loaded at startup from a known
directory, declaring their identity and BTCPay version compatibility via a
manifest. This article covers the loader contract, the extension surfaces
(`IUIExtension`, `PaymentMethodHandler`, hooks), the build/package flow
using the `BTCPayServer.Plugins.Test` template, and how packaged `.btcpay`
files are installed via the admin UI.

## Walkthrough / mechanics

Plugin discovery: at host startup BTCPay scans
`/var/lib/btcpayserver/plugins/` (or `wwwroot/plugins/` in dev). Each
plugin is a directory containing a primary DLL plus its dependency tree.
The `BTCPayServerPlugin` base class is the entry point:

```csharp
public class HelloPlugin : BaseBTCPayServerPlugin
{
    public override string Identifier => "BTCPayServer.Plugins.Hello";
    public override string Name => "Hello Plugin";
    public override Version Version => new Version(1, 0, 0);
    public override string Description => "Demo plugin";

    public override IBTCPayServerPlugin.PluginDependency[] Dependencies =>
        new[] {
          new IBTCPayServerPlugin.PluginDependency
          { Identifier = "BTCPayServer", Condition = ">=2.0.0" }
        };

    public override void Execute(IServiceCollection services)
    {
        services.AddSingleton<IUIExtension>(
            new UIExtension("HelloNav", "header-nav"));
        services.AddSingleton<HelloService>();
    }
}
```

Extension surfaces:

| Surface | Interface / contract |
|---------|----------------------|
| Sidebar / nav | `IUIExtension("View", "header-nav")` |
| Store dashboard widget | `IUIExtension(..., "store-integrations-nav")` |
| Payment method | `PaymentMethodHandler<T>` |
| Server settings page | `IUIExtension(..., "server-nav")` |
| Background service | `IHostedService` registration |
| Database (per plugin) | `BTCPayDbContextFactory<MyContext>` |

Razor views live under `Views/HelloPlugin/*.cshtml` and are referenced by
the `IUIExtension` partial-view name.

Build a plugin from the official template:

```bash
dotnet new -i BTCPayServer.Plugins.Test
dotnet new btcpayplugin -n MyStore.Plugin
cd MyStore.Plugin
dotnet build -c Release
```

Package as `.btcpay`:

```bash
# script provided by template
./build-plugin.sh
# produces: bin/Release/MyStore.Plugin.btcpay
```

Internally the `.btcpay` file is a `.zip` containing the plugin DLLs, its
`Resources/`, the manifest, and an icon.

Install at runtime:

1. Server Settings -> Plugins -> Upload `.btcpay` file.
2. Restart BTCPay (button or `btcpay-restart.sh`).
3. Plugin row shows "Loaded vX.Y.Z" with toggle to disable.

## Worked example

Add a custom store-dashboard widget that pulls a live BTC/USD quote.

`Controllers/HelloController.cs`:

```csharp
[Route("plugins/hello")]
public class HelloController : Controller
{
    private readonly RateFetcher _rates;
    public HelloController(RateFetcher rates) { _rates = rates; }

    [HttpGet("widget/{storeId}")]
    public async Task<IActionResult> Widget(string storeId)
    {
        var rate = await _rates.FetchRate(
            new CurrencyPair("BTC", "USD"),
            new RateRules.Storage(), default);
        return PartialView("HelloWidget", rate.BidAsk?.Bid ?? 0m);
    }
}
```

`Views/Hello/HelloWidget.cshtml`:

```cshtml
@model decimal
<div class="widget-card">
  <h6>BTC/USD</h6>
  <p class="display-5">$@Model.ToString("N0")</p>
</div>
```

`Plugin.cs`:

```csharp
services.AddSingleton<IUIExtension>(
    new UIExtension("HelloWidget", "store-integrations-nav"));
```

After install + restart, every Store dashboard renders the widget pulling
the live rate from BTCPay's built-in `RateFetcher`.

| Lifecycle event | Hook |
|-----------------|------|
| Startup DI register | `Execute(IServiceCollection)` |
| Database migrations | `IBTCPayDbContextFactory.AddDbContext<T>` |
| Invoice paid | `EventAggregator.Subscribe<InvoiceEvent>` |
| Webhook delivered | `EventAggregator.Subscribe<WebhookDeliveryRequest>` |
| Plugin uninstalled | manifest `OnUninstall` (best-effort cleanup) |

## Common pitfalls

- BTCPay version drift - `Dependencies` must be set; if you ship a plugin
  built against 2.0.x and BTCPay updates to 2.1 with breaking API changes,
  the loader refuses to attach until you publish a new build.
- DLL hell - bundling System.* dependencies that conflict with the host;
  prefer `<PrivateAssets>all</PrivateAssets>` for shared libraries and let
  the host provide them.
- Migrations ordering - per-plugin DbContext migrations run only after the
  plugin loads cleanly; a failing `Execute()` skips migrations and the
  schema can drift.
- Caching of Razor views - in development set `RuntimeCompilation = true`
  via the test environment; in production rebuild and restart for any
  view change.
- Webhook plugin loops - subscribing to `InvoiceEvent` and creating a new
  invoice in the handler is an easy infinite loop; gate by metadata.
- Forgetting to handle Tor admin sessions - some plugin UIs assume HTTPS;
  test under `http://<onion>` to ensure no mixed-content errors.

## References

- Plugin docs: https://docs.btcpayserver.org/Development/Plugins/
- Plugin template: https://github.com/btcpayserver/BTCPayServer.Plugins.Test
- Source examples: https://github.com/btcpayserver/btcpayserver/tree/master/Plugins
- Greenfield events: https://docs.btcpayserver.org/Development/GreenFieldExample/
