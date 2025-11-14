# Placeholder Dataset VFX Workflow

This note explains how the placeholder dataset workflow (branches `30-add-placeholder-dataset` and `cursor/document-vfx-graph-placeholder-product-ac9e`) relates to `main`, how to run it, and how to move VFX work back into the dataset-driven build.

## Branch Map

- `main` – fully dataset-driven pipeline. `ParticleAnimationController` streams a `GraphicsBuffer` plus timestep metadata into the production graph `Assets/Dataset_visualization/Dataset_Visual.vfx`. `/.worktrees/main/Assets/Scripts/DataBufferBinder.cs`249:268  
- `30-add-placeholder-dataset` (and this doc branch) – same controller, but `InitializeDataset()` can bypass data ingestion and spawn standalone VFX Graph assets from `Assets/PlaceholderVisualization`. `Assets/Scripts/DataBufferBinder.cs`361:401

## Runtime Modes

- **Dataset mode (`main` parity).** When `useTestData` is false and a dataset buffer exists, the controller uploads the particle buffer and sets `PointCount`, `CurrentTimestep`, and `ColorScheme` before restarting the graph. This is the contract the production graph expects. `/.worktrees/main/Assets/Scripts/DataBufferBinder.cs`249:268
- **Placeholder mode (placeholder branches only).** Invoking `InitializeDataset()` while `placeholderVfxAsset` (or the pollutant overrides) is assigned disables the dataset-driven `VisualEffect`, clears prior instances, and marks the controller as `usingPlaceholder`. Timesteps then drive the spawned placeholder effects through `placeholderStepProperty` (defaults to `CurrentTimestep`). `Assets/Scripts/DataBufferBinder.cs`361:401

## Asset Layout

- Placeholder graphs live under `Assets/PlaceholderVisualization/` (e.g., `Pollutant1_Sphere.vfx`, `Pollutant2_Cube.vfx`). Their prefabs can be staged in `Assets/Scenes/PlaceholderVisualization.unity`.
- Production graphs live under `Assets/Dataset_visualization/` and already expose the dataset contract (`DataBuffer`, `PointCount`, `CurrentTimestep`, `ColorScheme`). `/.worktrees/main/Assets/Dataset_visualization/Dataset_Visual.vfx`240:299

## Porting Placeholder Effects Into `main`

1. **Expose the production contract in the graph.** Add Blackboard entries that match `Dataset_Visual.vfx` (`GraphicsBuffer DataBuffer`, `uint PointCount`, `int CurrentTimestep`, `int ColorScheme`). Wire `DataBuffer` into a `Set Position (Attribute from Buffer)` or equivalent to read particle positions. Placeholder graphs currently expose only local scalars (e.g., `Spawnrate`, `MaxTimesteps`, `MainColor`), so this step is mandatory before the dataset controller can drive them. `Assets/PlaceholderVisualization/Pollutant1_Sphere.vfx`258:339
2. **Bind controller parameters.** After copying the updated `.vfx` asset into `main`, open the prefab, assign it to the dataset `VisualEffect`, and ensure the controller continues to call `SetGraphicsBuffer("DataBuffer", visualBuffer)` and `SetInt("ColorScheme", currentColorScheme)`—no code changes are needed if the property names match. `/.worktrees/main/Assets/Scripts/DataBufferBinder.cs`249:268
3. **Keep GUIDs when copying assets.** Use Unity’s package export (or copy the `.vfx` + `.meta`) so prefab references survive merges.
4. **Document new Blackboard fields.** Add sticky notes inside the graph describing expected value ranges so the dataset engineers know how to drive them.

## Using the Placeholder Scene

1. Checkout `30-add-placeholder-dataset` (or this branch) and open `Assets/Scenes/PlaceholderVisualization.unity`.
2. Assign the pollutant graph assets to `ParticleAnimationController` (sphere, cube, plume). Timesteps can be driven via UI buttons; motion smoothing is controlled by the `placeholder*` fields in the inspector.
3. To preview multiple pollutants, toggle them on via the UI—each variant instantiates its own `VisualEffect` and is advanced via `ApplyPlaceholderStep()`. `Assets/Scripts/DataBufferBinder.cs`328:350

## UI Integration Notes

- `main` expects `CanvasHelper` to call `ParticleAnimationController.SetColorScheme()` so dataset rendering changes immediately with the active pollutant. `/.worktrees/main/Assets/Scripts/CanvasHelper.cs`18:205
- The placeholder branch currently ships a simplified `CanvasHelper` that only mixes colors locally and never reaches into the controller. `Assets/Scripts/CanvasHelper.cs`6:96  
  When porting UI refinements back to `main`, re-introduce the controller reference so toggles continue to drive `ColorScheme`.

## Checklist Before Promoting a Placeholder Effect

- [ ] Graph exposes `DataBuffer`, `PointCount`, `CurrentTimestep`, `ColorScheme`.
- [ ] Tested inside the placeholder scene with `placeholderUseScriptMotion` both on and off.
- [ ] Copied into `main` with GUIDs preserved and verified that `ParticleAnimationController` renders it using real dataset buffers.
- [ ] Documented any new inspector fields or graph parameters in this guide or within the graph.
