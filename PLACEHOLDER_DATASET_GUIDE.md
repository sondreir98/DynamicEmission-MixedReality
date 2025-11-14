# Placeholder Dataset VFX Workflow

This note explains how the placeholder dataset workflow (branches `30-add-placeholder-dataset` and `cursor/document-vfx-graph-placeholder-product-ac9e`) relates to `main`, how to run it, and how to move VFX work back into the dataset-driven build.

## Branch Map

- `main` – fully dataset-driven pipeline. `ParticleAnimationController` reads either the baked dataset or the synthetic cube into a GPU `GraphicsBuffer`, then writes that buffer plus the scalar parameters (`PointCount`, `CurrentTimestep`, `ColorScheme`) into the exposed Blackboard fields on `Assets/Dataset_visualization/Dataset_Visual.vfx`. Inside the VFX Graph, those values decide which particles spawn and how they are coloured, so “streaming the buffer” simply means keeping those bindings up to date whenever the timestep changes. `/.worktrees/main/Assets/Scripts/DataBufferBinder.cs`249:268  
- `30-add-placeholder-dataset` (and this doc branch) – same controller, but calling `InitializeDataset()` while a placeholder asset is assigned short-circuits the normal data path: the script stops updating the dataset `VisualEffect`, flags itself as `usingPlaceholder`, and instantiates the pollutant-specific graphs from `Assets/PlaceholderVisualization`. From that point on, timestep changes are forwarded into those instantiated graphs instead of the dataset graph. `Assets/Scripts/DataBufferBinder.cs`361:401

## Runtime Modes

- **Dataset mode (`main` parity).** This is the default path: once a dataset buffer is present, the controller keeps re-binding it to the graph every time `currentTimestep` changes, ensuring the GPU always draws whichever slice of the dataset should currently be visible. Restarting the graph after each bind guarantees that dead particles from the previous timestep are flushed. `/.worktrees/main/Assets/Scripts/DataBufferBinder.cs`249:268
- **Placeholder mode (placeholder branches only).** Here the controller behaves more like a prefab spawner. Setting `usingPlaceholder = true` means `Update()` no longer touches the dataset buffer; instead it loops over each instantiated pollutant `VisualEffect`, sets the property named by `placeholderStepProperty` (defaulting to `CurrentTimestep`), and optionally eases their transforms via the motion helpers. `Assets/Scripts/DataBufferBinder.cs`328:350

## Asset Layout

- Placeholder graphs live under `Assets/PlaceholderVisualization/` (e.g., `Pollutant1_Sphere.vfx`, `Pollutant2_Cube.vfx`). Each graph is authored to be self-contained (its own spawn logic, colours, motion), so artists can drop the prefab into `Assets/Scenes/PlaceholderVisualization.unity` and iterate without any dataset dependencies.
- Production graphs live under `Assets/Dataset_visualization/` and already expose the dataset contract (`DataBuffer`, `PointCount`, `CurrentTimestep`, `ColorScheme`). Any asset meant to run on `main` must provide those properties so the controller can drive it with real data. `/.worktrees/main/Assets/Dataset_visualization/Dataset_Visual.vfx`240:299

## Porting Placeholder Effects Into `main`

1. **Expose the production contract in the graph.** On the placeholder branch these graphs usually only expose artistic knobs (spawn rate, colours, etc.). To make them data-driven, add the same Blackboard properties used by `Dataset_Visual.vfx` and rewire the spawn context so positions and attributes are pulled from `DataBuffer` instead of being procedurally generated. Until those properties exist, `ParticleAnimationController` has nothing to bind and the graph cannot be driven by data. `Assets/PlaceholderVisualization/Pollutant1_Sphere.vfx`258:339
2. **Bind controller parameters.** Once the graph accepts the dataset properties, copying it into `main` is mostly housekeeping: assign the `.vfx` to the production prefab or scene object, then let the controller continue calling `SetGraphicsBuffer("DataBuffer", visualBuffer)`, `SetUInt("PointCount", ...)`, `SetInt("CurrentTimestep", ...)`, and `SetInt("ColorScheme", ...)`. Because the method names stay identical, no C# changes are required. `/.worktrees/main/Assets/Scripts/DataBufferBinder.cs`249:268
3. **Keep GUIDs when copying assets.** Unity identifies assets by GUID, so exporting/importing as a Unity package—or copying both the `.vfx` and `.meta`—ensures any prefabs or scenes referencing the graph keep their links after the merge.
4. **Document new Blackboard fields.** Whenever you add extra exposed properties (e.g., “TrailLength”, “HeatmapIntensity”), leave a sticky note in the graph or update this doc so the backend team knows which dataset values should drive them.

## Using the Placeholder Scene

1. Checkout `30-add-placeholder-dataset` (or this branch) and open `Assets/Scenes/PlaceholderVisualization.unity`. This scene is lightweight on purpose: it contains no dataset prefab, only the controller plus UI canvases.
2. In the inspector, assign the pollutant graph assets (sphere, cube, plume) to the controller fields. Press Play and use the UI buttons to start/stop timesteps; sliders such as `placeholderMoveSmoothTime` immediately influence how the instantiated graphs behave.
3. Enable multiple pollutants through the toggles. For each toggle-on event the controller spawns or destroys a `VisualEffect`, tracks it in `activePollutants`, and advances it via `ApplyPlaceholderStep()`, mimicking how a future multi-buffer dataset implementation could work. `Assets/Scripts/DataBufferBinder.cs`328:350

## UI Integration Notes

- `main` expects `CanvasHelper` to hold a reference to `ParticleAnimationController` and call `SetColorScheme()` whenever a toggle changes. That call simply sets `currentColorScheme` and reinitializes the production VFX so the dataset immediately reflects the user’s selection. `/.worktrees/main/Assets/Scripts/CanvasHelper.cs`18:205
- The placeholder branch’s `CanvasHelper` was simplified to demonstrate colour mixing concepts without touching the controller (it just tracks toggles locally). Before merging any placeholder UI work back to `main`, re-add the controller reference or the toggles will no longer affect the dataset rendering. `Assets/Scripts/CanvasHelper.cs`6:96

## Checklist Before Promoting a Placeholder Effect

- [ ] Graph exposes `DataBuffer`, `PointCount`, `CurrentTimestep`, `ColorScheme`, and consumes them correctly.
- [ ] Behaviour verified inside the placeholder scene with `placeholderUseScriptMotion` both enabled and disabled (so transform math does not depend on a specific inspector setting).
- [ ] Asset copied to `main` with GUIDs preserved and validated against a real dataset buffer in `Assets/Scenes/DataVisualization.unity`.
- [ ] Any new inspector fields or Blackboard parameters are documented (sticky note or doc) so future developers know how to drive them with data.
