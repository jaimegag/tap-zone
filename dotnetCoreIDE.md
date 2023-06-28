# How to guide to use TAP VSCode IDE integration with .NET Core apps

### 0. Prerequisites
- You already have the TAP Iterate/Full cluster in place
- You already have `tilt` [installed](https://docs.tilt.dev/install.html) in your workstation (v.0.30.12 or later)
- Install the `Tanzu Developer Tools` plugin corresponding to your TAP version in VSCode
- Install VSCode C# plugin
- Install .NET 6.0 SDK. This the [url](https://dotnet.microsoft.com/en-us/download/dotnet/thank-you/sdk-6.0.408-macos-arm64-installer) for MAC M1 chipsets.
- Restart/reload VSCode

### 1. Start from the Accelerators
- Go to the Accelerators view in the TAP GUI, or alternatively the Accelerator plugin in the IDE bt clicking on the Tanzu logo on your left menu.
- Choose one of the .NET Core / C# Accelerators. Forexample: click on "Steeltoe Weather Forecast" Accelerator.
- Fill the simple `Project Name` and `Application Name` fields and click on Next, then Generate Project.

### 2. Open the project in VSCode
- If you downloaded a zip file from the TAP GUI's Accelerator view, then find the downloaded zip file in your file system and unzip it, it will create a new folder.
- Open a new VSCode Window, then Open the folder of the project you just unzipped.
- Go to Extensions > Tanzu Developer Tools > Extension settings, then go to Workspace tab and update:
    - Tanzu: Namespace & Selected Namespaces: myapps
    - Tanzu: Source Image: harbor.rito.tkg-vsp-lab.hyrulelab.com/tap/workloads/weatherforecast-source (change to the actual registry/repository of your choice)

### 3. Adjust Application configuration
- If your Supply Chain in the Iterate/Full cluster has tests, then open `config/workload.yaml` and add label: `apps.tanzu.vmware.com/has-tests: "true"`. Save changes.
- Edit Tiltfile and add: `allow_k8s_contexts('zora-admin@zora')` after the `NAME` (replace with the active k8s context that points to your iterate/full cluster). This is to authorize Tilt to deploy in that cluster. Save changes.

### 4. Live Update from VSCode
- Change k8s context to your Iterate/Full cluster
- Right click on any file in the project, then click on `Tanzu: Live Update Start`
    - This will trigger deploying the app with special labels and flags for container image to support live updates. App will go through the supply chain. Once it finishes, it's ready for Live Updates
- Open the app in the browser:
    - Example: https://weatherforecast-steeltoe.myapps.tap.zora.tkg-vsp-lab.hyrulelab.com/
- Make any change on any file, code or static. Save files.
- Open a terminal in your VSCode window.
    - Run: `dotnet build`. This will update the `bin` folder.
    - Live Update plugin with Tilt will detect changes (in the bin folder) and inject only the changed static files or binaries in the container.
        - It will show something like this in the LiveUpdate terminal:
```
weatherforec… │ 1 File Changed: [bin/Debug/net6.0/StaticFiles/index.html]
weatherforec… │ Will copy 1 file(s) to container: [weatherforecast-steeltoe-00001-deployment-546ddfd575-gpq9b/workload]
weatherforec… │ - '/Users/jgonzalezagu/Code/demo/weatherforecast-steeltoe/bin/Debug/net6.0/StaticFiles/index.html' --> '/workspace/StaticFiles/index.html'
weatherforec… │   → Container weatherforecast-steeltoe-00001-deployment-546ddfd575-gpq9b/workload updated!
```

### 5. Add Catalog to TAP GUI to leverage App Live View
Register Entitiy URL: https://github.com/jaimegag/tap-zone/blob/main/catalogs/weather-forecast-catalog-info.yaml
