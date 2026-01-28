# RTX Remix Toolkit AI Coding Instructions

## Project Overview
This is **RTX Remix Toolkit**, an NVIDIA Omniverse-based modding tool for enhancing classic DirectX 8/9 games. The toolkit enables modders to swap game assets with high-fidelity replacements, adjust lighting, and leverage generative AI for texture remastering.

## Architecture: Flux vs. Lightspeed

The codebase follows a **two-repository pattern** within one monorepo:

- **Flux (`omni.flux.*`)**: Generic, reusable extensions (30+ extensions) usable across projects. Examples: `omni.flux.stage_manager.widget`, `omni.flux.properties_pane.properties.widget`
- **Lightspeed (`lightspeed.trex.*`)**: Remix-specific toolkit implementations (80+ extensions). Examples: `lightspeed.trex.project_wizard`, `lightspeed.trex.material.core`

**Pattern**: Create Flux extensions first for generic functionality, then create specialized Lightspeed extensions that consume Flux and add USD/Remix-specific logic.

## Extension Structure & Build System

**Location**: [source/extensions/](../source/extensions/) contains all extensions.

**Key build files**:
- `build.bat` / `build.sh` - delegates to `repo.bat build`
- `repo.bat` - main build orchestration tool; run `repo.bat -h` for all commands
- `prebuild.toml` - copies templates and resources to build output

**Each extension has**:
- `config/extension.toml` - package metadata, dependencies, Python modules
- `premake5.lua` - links data/docs/source folders to build target
- `lightspeed/` or `omni/` - Python source modules (must match `[[python.module]]` name in TOML)

**Build output**: `_build/windows-x86_64/release/` contains all built extensions and app executables.

## Critical Developer Workflows

**Building**:
```bash
.\build.bat          # Standard build
.\build.bat -r       # Rebuild from scratch
.\build.bat --clean  # Clean build artifacts
```

**Code quality** (optional pre-commit hooks):
```bash
.\format_code.bat    # Auto-format with ruff
.\lint_code.bat      # Lint check with ruff
.\install_hooks.bat  # Enable pre-commit hooks
```

**Running the app**:
- `_build/windows-x86_64/release/lightspeed.app.trex.bat` - End-user version
- `_build/windows-x86_64/release/lightspeed.app.trex_dev.bat` - Development version (extra features)
- `_build/windows-x86_64/release/lightspeed.app.trex.ingestcraft.bat` - Asset ingestion UI

**Debugging flags**:
```
--enable omni.kit.debug.pycharm       # Attach PyCharm debugger
--/telemetry/enableSentry=false       # Skip Sentry tickets during dev
--/rtx/verifyDriverVersion/enabled=false  # Skip driver version check
--/app/extensions/fsWatcherEnabled=0  # Disable file watcher
```

## Python Environment & Dependencies

**Dependency management strategy**:
The RTX Remix Toolkit uses a multi-layered dependency system:

- **[pyproject.toml](../pyproject.toml)**: Root project configuration with build system, package metadata, and dev tool settings
- **[deps/](../deps/)**: Host and target dependency specifications via packman:
  - `host-deps.packman.xml` - Build-time dependencies (Premake5, repo tools)
  - `kit-sdk.packman.xml` - Omniverse Kit SDK version specification
  - `target-deps.packman.xml` - Runtime dependencies (compiled libraries, tooling)
- **[deps/pip_flux.toml](../deps/pip_flux.toml)**: Flux extension Python dependencies (omni.flux.* packages)
- **[deps/pip_internal.toml](../deps/pip_internal.toml)**: Lightspeed extension Python dependencies (lightspeed.trex.* packages)

**Python environment configuration**:
Extensions automatically discover dependencies from `config/extension.toml` `[dependencies]` section:
```python
# Example from lightspeed.trex.asset_replacements.core.shared/config/extension.toml
[dependencies]
"lightspeed.common" = {}
"lightspeed.layer_manager.core" = {}
"omni.flux.validator.factory" = {}
"omni.flux.utils.common" = {}
```

**Installing dependencies after code changes**:
1. Modify extension's `config/extension.toml` dependencies if adding new imports
2. Run `.\build.bat` - build system automatically pulls packman dependencies and pip packages
3. Build output in `_build/windows-x86_64/release/` includes all resolved dependencies
4. For manual pip installation in dev environment:
   ```bash
   pip install -r requirements.docs.txt       # Documentation build dependencies
   pip install -e .                           # Install toolkit in editable mode
   ```

**Handling missing module warnings**:

When developing with Pylance/Pyright, you may see `reportMissingImports` warnings for modules that ARE available at runtime. Common scenarios:

1. **Omniverse Kit modules** (carb, omni.usd, omni.ui, etc.):
   - Not installed in your local Python environment - they're provided by Kit at runtime
   - Suppress with: `# pyright: ignore[reportMissingImports]`

2. **Extension dependencies not in current extension**:
   - Another extension provides the import at runtime via build system
   - Suppress with: `# pyright: ignore[reportMissingImports]`

3. **Optional/platform-specific imports**:
   ```python
   try:
       import pydantic  # Optional dependency
   except ImportError:
       # Handle gracefully or suppress warning
       pass  # pyright: ignore[reportMissingImports]
   ```

4. **Configure Pylance** for the workspace:
   - Create `.pylintrc` or `pyrightconfig.json` in workspace root
   - Set `python.analysis.extraPaths` to include `source/extensions/` for module discovery
   - Set `reportMissingImports = "warning"` to downgrade from error if needed

**Viewing resolved dependencies**:
Each extension's resolved dependencies can be inspected at:
- Build output: `_build/windows-x86_64/release/exts/<extension-name>/`
- Extension metadata: Check `config/extension.toml` and `[[dependencies]]` table
- Package information: `_build/windows-x86_64/release/extscache/` contains extension index

**Extension.toml dependency best practices**:
- List all direct imports explicitly in `[dependencies]`
- Use `lightspeed.common` as base for constants, enums, path validation
- Service extensions depend on minimal core/common (lighter dependencies)
- Widget extensions depend on services + core + utils (heavier, UI-related)
- Avoid circular dependencies - build system will fail if detected
- Example dependency chain: Widget → Service → Core → Common

**Python version compatibility**:
The toolkit targets Python 3.10+ (as per Omniverse Kit SDK requirements). Ensure:
- Type hints use Python 3.10+ syntax (no `from __future__ import annotations` needed for simple cases)
- Avoid deprecated modules (e.g., use `typing.Sequence` instead of collections.abc equivalents in some contexts)
- Test code paths with minimum supported Python version

## USD & OmniGraph Patterns

- **USD paths** follow strict conventions: `/RootNode/meshes/mesh_<HASH>`, `/RootNode/Looks/mat_<HASH>`, `/RootNode/lights/light_<HASH>` (16-char hex hashes)
- **Regex patterns** for path validation in [source/extensions/lightspeed.common/lightspeed/common/constants.py](../source/extensions/lightspeed.common/lightspeed/common/constants.py) - use these when parsing prim paths
- **Material binding**: `material:binding` relationship; textures use `inputs:diffuse_texture`, `inputs:normalmap_texture`, etc.
- **OmniGraph**: Logic nodes stored as `OmniGraphNode` prims in USD; see `OMNI_GRAPH_TYPE` constants

## Key Extension Patterns

**Service extensions** (e.g., `lightspeed.trex.project_manager.service`, `lightspeed.trex.texture_replacements.service`):
- Implement singleton services via `omni.flux.service.factory.IServiceProvider`
- Register in extension's `extension_factory()` function
- Handle state management, file I/O, business logic
- No UI components; expose methods for consumption by widgets
- Example: `lightspeed.project_manager.service` manages project lifecycle and file operations
- Test pattern: Service extensions define test dependencies like `omni.flux.utils.tests` for unit testing

**Widget extensions** (e.g., `lightspeed.trex.project_wizard.window`, `lightspeed.trex.properties_pane.widget`):
- Use `omni.ui` for UI construction; may delegate to Flux widget extensions like `omni.flux.wizard.widget`
- Consume service extensions for business logic via service factory
- Typically inherit from `WorkspaceWindowBase` for workspace integration
- Follow pattern: UI layer handles presentation, delegates to core/.service for logic
- Example: `lightspeed.trex.project_wizard.window` depends on `lightspeed.trex.project_wizard.core` (business logic)

**Core extensions** (e.g., `lightspeed.trex.capture.core.shared`, `lightspeed.trex.material.core.shared`):
- Shared data structures, business logic, and API
- Model layer for other extensions to import and use
- Named with `.core.shared` suffix for visibility
- Handle USD manipulation, path validation, prim operations
- Example: `lightspeed.trex.material.core.shared` provides `get_materials_from_prim()`, `get_material_layer_stack()`

**Event extensions** (e.g., `lightspeed.event.app_shutdown`, `lightspeed.event.app_start`):
- React to app lifecycle events via `GlobalEventNames` enum
- Publish/subscribe pattern: emit events via `lightspeed.events_manager`
- Event types: `CAPTURE_LAYER_IMPORTED`, `CONTEXT_CHANGED`, `PAGE_CHANGED`, `LOAD_PROJECT_PATH`, `LOGIC_GRAPH_CREATE_REQUEST`
- Subscribers return `True/False` to approve/interrupt actions (e.g., pending changes check on project load)

## Conventions & Important Constants

**File locations** (from [lightspeed.common/constants.py](../source/extensions/lightspeed.common/lightspeed/common/constants.py)):
- Assets: `<project>/rtx-remix/assets/{models,textures,ingested}/`
- Captures: `<project>/rtx-remix/captures/`
- Mods: `<project>/rtx-remix/mods/`
- USD files: `.usd` (binary), `.usda` (ASCII), `.usdc` (compressed)

**Texture handling**:
- `TextureInfo` objects specify compression format (BC7, BC5, BC4) by texture type (diffuse, normalmap, etc.)
- Auto-upscaling uses layer files: `autoupscale.usda`
- Texture import validates against schema: `MODEL_INGESTION_SCHEMA_PATH`, `MATERIAL_INGESTION_SCHEMA_PATH`

**Remix categories**: 30+ material categories (World UI, Sky, Particle, Decal, etc.) stored as `remix_category:<type>` attributes. Reference `REMIX_CATEGORIES` dict in `../source/extensions/lightspeed.common/lightspeed/common/constants.py` for validation logic.

**Window names**: `WindowNames` enum (Viewport, Stage Manager, Properties, Sidebar, etc.) for UI workspace integration.

## Common Data Flows & Workflows

**Project Loading Flow**:
1. User clicks "Open Project" → `lightspeed.trex.project_wizard.window` 
2. Widget emits `LOAD_PROJECT_PATH` event via `events_manager`
3. Event subscribers approve/reject (e.g., check pending changes)
4. `lightspeed.project_manager.service` loads project USD files
5. `CONTEXT_CHANGED` event emitted to notify other extensions
6. UI extensions query service for project state and update displays

**Material Assignment Flow**:
1. User selects mesh in `lightspeed.trex.selection_tree.widget`
2. `lightspeed.trex.properties_pane.material.widget` queries `material.core.shared` for bound materials
3. Widget calls `lightspeed.trex.material.core.shared` APIs (e.g., `get_materials_from_prim()`)
4. User edits material parameters → calls `lightspeed.tool.material.core` service
5. Service modifies USD prim relationships and attributes
6. Stage notifies dependent extensions via USD change notifications

**Texture Replacement Flow**:
1. `lightspeed.trex.ingestcraft.widget` ingests texture into `assets/ingested/`
2. `lightspeed.trex.texture_replacements.service` validates ingestion schema
3. Texture metadata stored in capture USD layers
4. At export, `lightspeed.trex.packaging.core` assembles final mod package with compressed textures

## Testing & Documentation

- **Tests**: Located as `[[test]]` sections in `extension.toml`; run via standard Kit testing
- **Docs**: Each extension has `docs/README.md`, `docs/CHANGELOG.md` (KeepAChangelog format)
- **Dev docs**: [docs_dev/](../docs_dev/) includes PYCHARM_GUIDE.md, DEBUGGING_GUIDE.md, TESTING_GUIDELINES.md
- **Contributing**: See [docs_dev/CONTRIBUTING.md](../docs_dev/CONTRIBUTING.md) for PR workflow and CLA signing

## Extension Development Checklist

When creating a new extension, follow this structure:

1. **Choose extension type**:
   - `lightspeed.trex.*.service` → Business logic, state management (no UI)
   - `lightspeed.trex.*.core.shared` → Shared APIs and data models
   - `lightspeed.trex.*.widget` → UI components using `omni.ui`
   - `omni.flux.*` → Reusable non-Remix functionality

2. **Create extension directory**: `source/extensions/<extension-name>/`
   ```
   config/extension.toml
   lightspeed/               # or 'omni/' for Flux
     trex/
       <module>/
         __init__.py
         extension.py        # Extension class
         setup.py            # Business logic (if service)
         widget.py           # UI code (if widget)
   docs/
     README.md
     CHANGELOG.md
   data/
     icon.png
     preview.png
   ```

3. **Configure extension.toml**:
   - Set `[[python.module]]` name to match directory structure
   - List all dependencies in `[dependencies]`
   - Add `[[test]]` sections with test dependencies
   - Use `category = "internal"` for toolkit-internal extensions

4. **Service extension pattern**:
   ```python
   from omni.flux.service.factory import IServiceProvider
   
   class MyService(IServiceProvider):
       def __init__(self):
           self._enabled = True
       
       def shutdown(self):
           self._enabled = False
   
   class MyServiceExtension(ext.IExtension):
       def on_startup(self, ext_id):
           service_factory.register(MyService, "lightspeed.my.service")
   ```

5. **Widget extension pattern**:
   ```python
   from lightspeed.trex.utils.widget.workspace import WorkspaceWindowBase
   from omni.flux.service.factory import ServiceFactory
   
   class MyWidget(WorkspaceWindowBase):
       def on_startup(self):
           self.service = ServiceFactory.get("lightspeed.my.service")
           # Build UI with omni.ui
   ```

6. **Hook into events** (optional):
   ```python
   from lightspeed.events_manager import on_event, post_event
   from lightspeed.common.constants import GlobalEventNames
   
   def subscribe_to_changes():
       on_event(GlobalEventNames.CONTEXT_CHANGED, self._on_context_changed)
   ```

## When Modifying Code

### Omniverse Kit SDK Integration Patterns

**Extension lifecycle**:
```python
import omni.ext
import carb
from omni.flux.service.factory import get_instance as _get_service_factory_instance

class MyExtension(omni.ext.IExtension):
    def on_startup(self, ext_id):
        carb.log_info(f"[{ext_id}] Startup")
        # Register services, plugins, connect to events
        _get_service_factory_instance().register_plugins([MyService])
    
    def on_shutdown(self):
        carb.log_info("[MyExtension] Shutdown")
        # Clean up resources
        _get_service_factory_instance().unregister_plugins([MyService])
```

**Context and stage access** (always use context names):
```python
import omni.usd
# Get default or named context
context = omni.usd.get_context("default")  # or your custom context name
stage = context.get_stage()
# Get edit target layer for modifications
edit_layer = stage.GetEditTarget().GetLayer()
```

**Logging** (carb is Omniverse's C++ API bridge):
```python
import carb
carb.log_info("Info message")
carb.log_warn("Warning message")
carb.log_error("Error message")
```

**UI workspace integration** (for widgets):
```python
from lightspeed.trex.utils.widget.workspace import WorkspaceWindowBase
from lightspeed.common.constants import WindowNames

class MyWidget(WorkspaceWindowBase):
    def __init__(self):
        super().__init__(
            window_name=WindowNames.PROPERTIES,
            default_width=400,
            default_height=600
        )
    
    def on_startup(self):
        # Initialize workspace window
        self._build_ui()
```

### Error Handling & Validation Strategies

**Custom exceptions** (domain-specific):
Domain-specific exceptions allow callers to handle errors appropriately. Example from [lightspeed.trex.asset_replacements.core.shared](../source/extensions/lightspeed.trex.asset_replacements.core.shared/lightspeed/trex/asset_replacements/core/shared/skeleton.py):
```python
class SkeletonAutoRemappingError(Exception):
    """Skeleton replacement does not have the proper attributes to be remapped as is."""
    pass

class SkeletonDefinitionError(Exception):
    """Missing part of SkelRoot, Skeleton or Binding"""
    pass

# Usage: raise with context about what went wrong
def __init__(self, skel_root: Usd.Prim, bound_prim: Usd.Prim):
    self._skel_root = UsdSkel.Root(self._skel_root_prim)
    if not self._skel_root:
        raise SkeletonDefinitionError(f"Skeleton root is not valid: {self._skel_root}")
    
    self._binding_api = UsdSkel.BindingAPI(bound_prim)
    if not self._binding_api:
        raise SkeletonDefinitionError(f"No skel binding found under: {bound_prim.GetPath()}")

# Callers catch and handle appropriately:
try:
    skel_replacement = SkeletonReplacementBinding(prim, ref_prim)
except SkeletonDefinitionError as err:
    detail_message += f"Could not remap skeleton for bound mesh: {err}\n"
    continue  # Skip this prim, continue with others
```

**Schema validation** (using validator factory):
```python
from omni.flux.validator.factory import get_instance as _get_validator_factory_instance

validator_factory = _get_validator_factory_instance()
# Schema paths are constants in lightspeed.common.constants
results = validator_factory.validate(asset_file, constants.MODEL_INGESTION_SCHEMA_PATH)

# Check for validation errors
for error in results.get_errors():
    carb.log_error(f"Validation error: {error}")
    raise AssetValidationError(str(error))

# Validation errors should bubble up with context:
if not stage:
    raise ValueError("No stage is currently loaded.")
if stage.GetRootLayer().anonymous:
    raise ValueError("No project is currently loaded.")
```

**Safe property access** (initialize defaults, cleanup with reset_default_attrs):
Pattern from [lightspeed.trex.asset_replacements.core.shared/setup.py](../source/extensions/lightspeed.trex.asset_replacements.core.shared/lightspeed/trex/asset_replacements/core/shared/setup.py):
```python
from omni.flux.utils.common import reset_default_attrs

class Setup:
    def __init__(self, context_name: str):
        # Always initialize _default_attr with all attributes used
        self._default_attr = {
            "_context_name": None,
            "_context": None,
            "_layer_manager": None,
        }
        # Set all attributes at once
        for attr, value in self._default_attr.items():
            setattr(self, attr, value)
        
        # Now safely assign values
        self._context_name = context_name
        self._context = omni.usd.get_context(context_name)
        self._layer_manager = _LayerManagerCore(context_name=context_name)
    
    def destroy(self):
        # Clean up: reset_default_attrs clears attributes defined in _default_attr
        if self._default_attr:
            reset_default_attrs(self)
```
**Key pattern**: If an attribute is added later without being in `_default_attr`, it won't be reset. Always initialize all attributes you'll use.

**Robust prim lookup** (validation before operations):
```python
def get_prim_safe(stage: Usd.Stage, path_str: str) -> Usd.Prim:
    """Get prim with strict validation"""
    from lightspeed.common.constants import COMPILED_REGEX_MESH_PATH
    
    # 1. Validate path format first
    if not COMPILED_REGEX_MESH_PATH.match(path_str):
        raise PathValidationError(f"Invalid path format: {path_str}")
    
    # 2. Get prim and check validity
    prim = stage.GetPrimAtPath(path_str)
    if not prim or not prim.IsValid():
        raise RuntimeError(f"Prim not found at {path_str}")
    
    # 3. Type-check if needed (example from code)
    if not prim.IsA(UsdGeom.Mesh):
        raise ValueError(f"Prim at {path_str} is not a Mesh")
    
    return prim

# Real example from codebase:
def is_ref_prim_path_valid(asset_path: str, prim_path: str, layer: Sdf.Layer, log_error=True) -> bool:
    """Validates reference prim exists before setting"""
    abs_new_asset_path = omni.client.normalize_url(layer.ComputeAbsolutePath(asset_path))
    # Check file exists
    _, entry = omni.client.stat(abs_new_asset_path)
    if not entry.flags & omni.client.ItemFlags.READABLE_FILE:
        return False
    
    # Open and check prim exists
    ref_stage = Usd.Stage.Open(abs_new_asset_path)
    if prim_path == "<Default Prim>":
        ref_root_prim = ref_stage.GetDefaultPrim()
        if ref_root_prim and ref_root_prim.IsValid():
            return True
        if log_error:
            carb.log_error(f"No default prim found in {abs_new_asset_path}")
        return False
    
    # Traverse and find prim
    for prim in ref_stage.TraverseAll():
        if str(prim.GetPath()) == prim_path:
            return True
    if log_error:
        carb.log_error(f"{prim_path} cannot be found in {abs_new_asset_path}")
    return False
```

**Event subscriber approval pattern** (control workflow):
Subscribers returning `True/False` allow other extensions to approve or block critical operations. This pattern enables cross-extension validation without direct coupling. Example event flow:
```python
# === EMIT SITE: Widget initiating project load ===
import carb
from lightspeed.events_manager import post_event
from lightspeed.common.constants import GlobalEventNames

class ProjectWizardWidget:
    def on_load_project_clicked(self, project_path: str):
        """User clicked "Load Project" button"""
        carb.log_info(f"User initiated project load: {project_path}")
        
        # Emit event - all subscribers can approve/reject
        results = post_event(GlobalEventNames.LOAD_PROJECT_PATH, project_path)
        
        # Check if any subscriber blocked the load
        if not all(results):  # If any returned False, block operation
            carb.log_warn(f"Project load blocked by subscriber check")
            return  # Operation interrupted
        
        # All subscribers approved - proceed with actual load
        carb.log_info(f"All subscribers approved - loading project: {project_path}")
        self._do_project_load(project_path)

# === SUBSCRIBER SITE 1: Unsaved changes checker ===
import carb
from lightspeed.events_manager import on_event
from lightspeed.common.constants import GlobalEventNames
from omni.flux.service.factory import ServiceFactory

class UnsavedChangesChecker:
    def __init__(self):
        # Register this callback as a subscriber
        on_event(GlobalEventNames.LOAD_PROJECT_PATH, self._on_project_load_attempt)
    
    def _on_project_load_attempt(self, project_path: str) -> bool:
        """
        Subscriber checks if there are unsaved changes.
        Returns True to approve load, False to block.
        """
        document_service = ServiceFactory.get("lightspeed.document_manager.service")
        
        if document_service.has_unsaved_changes():
            carb.log_warn(
                f"[UnsavedChangesChecker] Blocking project load - unsaved changes exist. "
                f"Save before loading: {project_path}"
            )
            return False  # BLOCK: Operation rejected
        
        carb.log_info("[UnsavedChangesChecker] No unsaved changes - load approved")
        return True  # APPROVE: Operation proceeds

# === SUBSCRIBER SITE 2: Another validation extension ===
import carb
from lightspeed.events_manager import on_event
from lightspeed.common.constants import GlobalEventNames

class ProjectValidator:
    def __init__(self):
        on_event(GlobalEventNames.LOAD_PROJECT_PATH, self._validate_project)
    
    def _validate_project(self, project_path: str) -> bool:
        """Another extension validates project structure"""
        if not self._is_valid_project(project_path):
            carb.log_error(
                f"[ProjectValidator] Blocking load - invalid project structure: {project_path}"
            )
            return False  # BLOCK
        
        carb.log_info(f"[ProjectValidator] Project structure valid - load approved")
        return True  # APPROVE

# Flow: emit_site → post_event() → calls all subscribers → collects bool results
# If ANY subscriber returns False → operation blocked
# If ALL subscribers return True → operation proceeds
```

**Accessing services from widgets**:
```python
from omni.flux.service.factory import ServiceFactory
service = ServiceFactory.get("lightspeed.project_manager.service")
project_path = service.get_current_project_path()
```

**Emitting events**:
```python
from lightspeed.common.constants import GlobalEventNames
from lightspeed.events_manager import post_event
post_event(GlobalEventNames.CAPTURE_LAYER_IMPORTED)
```

**Subscribing to events**:
```python
from lightspeed.events_manager import on_event
def on_project_load(path: str) -> bool:
    return True  # Approve load
on_event(GlobalEventNames.LOAD_PROJECT_PATH, on_project_load)
```

**Extension.toml dependency ordering**:
- List dependencies explicitly; build system respects order
- Circular dependencies cause build failures
- Service extensions should depend on minimal core/common
- Widget extensions depend on services + core + utils

### USD & Prim Manipulation

**Common USD operations**:
```python
from pxr import Usd, UsdShade
# Get current stage
stage = omni.usd.get_context().get_stage()
# Find prim by path
prim = stage.GetPrimAtPath("/RootNode/meshes/mesh_ABCD1234EFGH5678")
# Bind material to mesh
UsdShade.MaterialBindingAPI(mesh_prim).Bind(material_prim)
# Query material
material, _ = UsdShade.MaterialBindingAPI(mesh_prim).ComputeBoundMaterial()
```

**Path validation**:
- Use compiled REGEX constants from `lightspeed.common.constants`: `COMPILED_REGEX_MESH_PATH`, `COMPILED_REGEX_INSTANCE_PATH`
- All hashes are 16 uppercase hex characters: `[A-Z0-9]{16}`
- Child prims follow parent naming: `/RootNode/meshes/mesh_HASH/child_name`

**Layer management**:
- Override layers stored per project in `rtx-remix/mods/` 
- Sublayers referenced via `./SubUSDs/` relative path convention
- Use `get_material_layer_stack()` to walk material inheritance

## High-Value Extension Examples

### Material Service (`lightspeed.trex.material.core.shared`)
Handles USD material binding and composition:
```python
# get_materials_from_prim: Extract bound materials from geometry
materials = material_setup.get_materials_from_prim(mesh_prim)
# Returns list of Sdf.Path objects for bound materials

# ComputeBoundMaterial: Query material binding with composition
material, _ = UsdShade.MaterialBindingAPI(prim).ComputeBoundMaterial()
# Returns (UsdShade.Material, relationship_name) or (None, None)

# get_material_layer_stack: Walk layer inheritance
stacks = material_setup.get_material_layer_stack(material_path)
# Returns list of layers where material is defined (for overrides)
```
**Key pattern**: Layer stack walking crucial for understanding which USD layer a material modification should target.

### Asset Replacements Service (`lightspeed.trex.asset_replacements.core.shared`)
Manages skeleton/mesh replacement bindings with validation:
- **Reference validation**: Check if replacement asset exists and is accessible
- **Schema validation**: Uses `omni.flux.validator.factory` to validate ingested assets against `MODEL_INGESTION_SCHEMA_PATH`
- **Error handling**: Custom exceptions (`SkeletonAutoRemappingError`, `SkeletonDefinitionError`) for domain-specific errors
- **Pydantic models**: Uses Pydantic for schema validation (see `omni.flux.pip_archive` dependency for Pydantic)

### Layer Manager Service (`lightspeed.layer_manager.core`)
Manages USD layer editing and metadata:
```python
# Get USD context and current edit layer
stage = omni.usd.get_context(context_name).get_stage()
edit_layer = stage.GetEditTarget().GetLayer()

# Layer attributes stored as custom metadata
LSS_LAYER_GAME_NAME = "lss:gameName"
LSS_LAYER_MOD_NAME = "lss:modName"
LSS_LAYER_MOD_VERSION = "lss:modVersion"
```
**Key pattern**: Metadata stored on layer objects; use `layer.customLayerData` for persistence.



