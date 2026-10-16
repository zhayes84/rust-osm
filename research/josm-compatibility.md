# JOSM compatibility boundaries for a building-edit prototype

Research for [Map JOSM compatibility surfaces that constrain a cross-platform prototype](https://github.com/zhayes84/rust-osm/issues/4), 2026-10-16. Reference revision: [`JOSM/josm@d663b2a601ba0f94376f48ee216a16f4a60a493a`](https://github.com/JOSM/josm/tree/d663b2a601ba0f94376f48ee216a16f4a60a493a). Findings below describe that revision, not an assertion about every JOSM release or plugin.

## Answer

A meaningful first prototype needs an explicit **editing behavior contract**, an imagery/projection context, and a comparable visual workload. It does not yet need a Java plugin runtime, complete MapCSS or preset compatibility, all file formats, or a framework choice. This is a research recommendation, not an approved scope or implementation plan.

The user's reported task is moving nodes of existing buildings to accommodate additions or correct misalignment. They clarified that the pause occurs when grabbing a node, in roughly four to five populated city blocks, using Bing imagery and the default JOSM style. This focuses the initial investigation on acquisition/selection and the transition into dragging; it does not establish the cause or demonstrate a sustained-drag bottleneck. Shared topology, selection, and hit-testing remain relevant correctness contracts.

The user supplied an installed-plugin list, recorded in the workload ticket. Which plugin operations are used or essential in this session remains unknown; installation alone establishes neither a compatibility requirement nor a cause of the pause. No reproducible dataset, complete JOSM preferences, imagery offset, projection, or hardware profile was supplied. The workload description and settings above are user reports, not independently reproduced measurements.

## Four different compatibility promises

| Promise | Meaning here | Evidence and implication |
| --- | --- | --- |
| Behavioral compatibility | Familiar actions, shortcuts, selection, geometry edits, and undo results | `JosmAction` connects commands to toolbar entries, global shortcuts, layer context, and selection listeners. Matching a menu label alone would not reproduce when an action is enabled. [Action source][actions] |
| Data/configuration compatibility | Read or preserve OSM files, styles, presets, shortcut settings, imagery configuration, and sessions | Each is its own contract. A `.osm` document, a MapCSS style, a preset XML document, and a `.jos` session have different readers and semantics. [OSM reader][reader], [style loader][styles], [preset reader][presetreader], [session writer][session] |
| Plugin capability ports | Reproduce a useful plugin's user-visible editing operation | This requires identifying actual capabilities the user needs. It does not imply loading that plugin's Java binary. Installed plugins are now known; their used or essential capabilities have not yet been identified. |
| Unchanged Java plugin execution | Load existing plugin classes and satisfy their JOSM dependencies | JOSM loads plugin classes through class loaders; the plugin base exposes `MapFrame`, preference UI, and download-selector callbacks. This is a much broader boundary than a language-neutral edit command. [Plugin loader][pluginloader], [plugin callbacks][plugins] |

The last row does not prove that bridging is impossible. It shows why promising it now would constrain the host object model and UI without evidence that this prototype needs it. Running Java code, satisfying JOSM APIs, reproducing Swing integrations, and preserving behavior are separate questions.

## Source findings that affect the prototype

### Input, actions, selection, and topology

`Shortcut` gives actions stable identifiers and configurable assignments, resolves collisions, and maps command modifiers to Ctrl on Windows/Linux and Meta on macOS. A cross-platform requirement cannot be represented by one hard-coded Windows key table. Existing preference-file import is a separate question from preserving default gestures. [Shortcut source][shortcuts]

Select mode is bound to `S`. Its implementation distinguishes adding/toggling selection, rotating/scaling with modifiers, selecting among overlapping objects, moving selected geometry, and explicit merging. The merge-by-default preference changes drag semantics. Virtual midpoint dragging creates a real node and changes affected way node lists in an undoable sequence; shared segments can be affected together, with selected ways influencing the target set. These are direct reasons to specify the gesture being benchmarked and include connected geometry in correctness examples. [Select mode][select]

Hit-testing depends on screen distance, selected-object preference, node precedence, and way-segment geometry. `getNearestNodeOrWay` includes a 4-pixel node-precedence case and a short-segment midpoint comparison; it is not simply nearest world-coordinate point. A faster renderer with different picking behavior may make the editor feel less reliable. Preserve observable selection outcomes in the chosen cases before choosing a picking implementation. [Picking source][picking]

**Decision needed:** does the first scope cover only moving existing nodes, or also inserting a virtual midpoint and moving a whole building? The user's additions example may need insertion; the misalignment example may need either vertex correction or whole-object translation. A small agreed gesture matrix is more informative than declaring broad JOSM parity. Include selection, shared-node effects, final coordinates, modified state, and undo/redo in its oracle. This is a proposed correctness criterion, not a claim that all these operations must ship first.

### Imagery and coordinate context

JOSM has WMS, WMTS, and TMS imagery layer implementations and offset UI/bookmarks. `Projection` defines WGS84/projected conversion, world bounds, axis-order behavior, and bounds reprojection; its documentation explicitly notes that transformed bounds can curve. [Imagery source][imagery], [projection interface][projection]

**Implication:** correcting apparent misalignment requires knowing the imagery source and offset, selected projection, and viewport. The source supports treating imagery alignment and OSM geometry as distinct state; it does not establish whether the user's buildings or imagery are misplaced. A controlled cached imagery scenario can make comparisons reproducible, but imagery-on and imagery-off measurements answer different questions. Live network performance should not silently determine the node-drag result. Provider availability, attribution, access credentials, and permission to redistribute test imagery remain uninvestigated; no reuse is proposed yet.

**Decision needed:** choose a recorded imagery/projection scenario and coordinate-comparison tolerance for the first workload. Decide whether imagery is required for the initial interactive evaluation and how it is held constant between implementations. Full projection catalog or provider parity can remain deferred.

### Rendering styles and presets

The MapCSS loader accepts standalone sources and archives; evaluation depends on selectors, scale ranges, subparts, and cascades. The bundled default style contains building-area fills and building-specific symbols. Therefore, visually sparse geometry and JOSM's styled viewport are different rendering workloads. Supporting the same appearance for a bounded fixture does not establish support for arbitrary user MapCSS. [MapCSS implementation][styles], [default style][defaultstyle]

Preset XML is parsed separately, with schema validation available and archive support. Presets distinguish object types and matching criteria, build editing UI, and produce undoable property changes; they can also participate in relation creation and role suggestions. Thus presets are both a data format and a behavior/UI contract. [Preset parser][presetreader], [preset behavior][presets]

**Decision needed:** use the reported default JOSM style as the comparison baseline and identify the visual features it must exercise, especially selected nodes/ways, fills, strokes, and labels. Measure any deliberately simplified rendering as a separate experiment. Arbitrary MapCSS loading and full preset-dialog parity can be deferred if the selected task only moves geometry and preserves existing tags. Before later offering preset compatibility, choose and test a format/behavior subset explicitly.

### Validation and file/session fidelity

Validation combines Java tests and MapCSS tag checking. An example directly relevant to buildings, `RightAngleBuildingTest`, only checks usable closed building ways within downloaded coverage and uses configurable angle thresholds. Data completeness and preferences can therefore change validation output, even for identical visible geometry. Porting MapCSS alone would not provide the whole validator. [Validator registry][validator], [MapCSS checker][checker], [building-angle test][buildingtest]

The OSM reader consumes nodes, ordered way references, relation members, metadata, bounds, and editor policies. The writer records primitive IDs, tags, membership/roles, metadata, and editor modification/deletion state under the relevant mode. File fidelity must preserve the editable graph and meaningful metadata, not only coordinates. Compare semantic content, not XML byte order. [OSM reader][reader], [OSM writer][writer]

JOSM's file registry includes OSM XML, OSM change, PBF, GeoJSON, GPX, and other importers; import and export lists are not identical and are extensible. Sessions separately preserve layer state, viewport, and projection. PBF input support or loading an OSM extract would not imply editable-session parity. [File registry][formats], [session serialization][session]

**Decision needed:** choose the prototype's fixture/input contract and an independent way to compare edit results against JOSM. A small explicit OSM XML fixture set is a candidate, not a settled format choice. Declare handling of unsupported or incomplete objects. Full validator reproduction, upload, arbitrary import/export, and session migration can remain outside the first benchmark; correctness checks for its own edits cannot.

## Decisions to resolve before implementation

1. **User input:** refine the node-grab gesture and identify which installed-plugin operations, presets, or modified shortcuts are actually used or essential. Bing imagery and the default JOSM style are reported; the installed-plugin list alone does not establish needed capabilities.
2. **Joint decision:** choose a bounded behavioral oracle, including connected geometry, selection/hover outcomes, undo/redo, metadata preservation, and coordinate tolerances. Do not demand pixel-identical UI unless that is explicitly intended.
3. **Research plus workload selection:** establish imagery/projection/style conditions and record them alongside baseline results. An empty-background benchmark may isolate geometry work but cannot establish the performance of the reported workflow alone.
4. **Joint decision:** choose a fixture and result-exchange boundary. Keep data/configuration compatibility promises separate from behavioral equivalence.
5. **Cross-platform constraint:** carry action identity, platform modifiers, pointer interaction, keyboard layouts, and display scaling into acceptance criteria; use Windows for initial measurement as agreed. Behavior on macOS/Linux remains unverified until exercised there.

These questions constrain interfaces and benchmark validity without selecting a GUI toolkit, graphics API, concurrency model, plugin runtime, or storage layout. Full extension execution, complete format catalogs, all presets/styles, and complete validator parity need not block this bounded experiment unless the chosen real workflow depends on them.

## Provenance and limits

Primary source was retrieved from GitHub's contents API at the pinned SHA. Browser fetches of the pinned repository/raw URLs failed in this environment; the authenticated API reads succeeded. No JOSM execution, user-session reproduction, profiling, or cross-platform testing was performed. Claims about the source are observations; prototype scope and sequencing above are recommendations.

The pinned root `LICENSE` states GPL v2 or later for JOSM and integral parts, describes a GPL v3 consequence when Apache-licensed repository code is included, and explicitly excludes plugins from its blanket license statement. Inspected Java files carry GPL headers referring to that file. These are source facts, not a legal determination about a Rust reimplementation. [License][license]

Potential reference assets have exact provenance: `resources/styles/standard/elemstyles.mapcss` and `resources/data/defaultpresets.xml`. Their inspected headers identify format/documentation, not an independent permission grant. Icons, bundled dependencies, third-party styles, presets, imagery, and plugins need their own provenance before any reuse decision. This artifact incorporates no JOSM code or asset files and proposes no license selection. [Style asset][defaultstyle], [preset asset][defaultpreset], [license][license]

[actions]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/actions/JosmAction.java#L44-L54
[shortcuts]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/tools/Shortcut.java
[select]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/actions/mapmode/SelectAction.java
[picking]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/gui/NavigatableComponent.java#L1574-L1636
[imagery]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/gui/layer/ImageryLayer.java
[projection]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/data/projection/Projection.java
[styles]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/gui/mappaint/mapcss/MapCSSStyleSource.java
[defaultstyle]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/resources/styles/standard/elemstyles.mapcss
[presets]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/gui/tagging/presets/TaggingPreset.java
[presetreader]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/gui/tagging/presets/TaggingPresetReader.java
[defaultpreset]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/resources/data/defaultpresets.xml
[validator]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/data/validation/OsmValidator.java
[checker]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/data/validation/tests/MapCSSTagChecker.java
[buildingtest]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/data/validation/tests/RightAngleBuildingTest.java#L30-L69
[reader]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/io/OsmReader.java
[writer]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/io/OsmWriter.java
[formats]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/actions/ExtensionFileFilter.java#L66-L126
[session]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/io/session/SessionWriter.java
[plugins]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/plugins/Plugin.java
[pluginloader]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/src/org/openstreetmap/josm/plugins/PluginHandler.java#L854-L928
[license]: https://github.com/JOSM/josm/blob/d663b2a601ba0f94376f48ee216a16f4a60a493a/LICENSE
