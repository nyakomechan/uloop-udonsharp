# UdonSharp Operations

Code examples for UdonSharp-specific operations using `uloop execute-dynamic-code`.

## Important Notes

- UdonSharp uses a **proxy + backing dual-component pattern**: every `UdonSharpBehaviour` (proxy) has a hidden `UdonBehaviour` (backing) on the same GameObject. The proxy is what you see in the Inspector; the backing is what runs at runtime.
- All snippets use `UdonSharpEditorUtility` from the `UdonSharpEditor` namespace (Editor-only assembly).
- PowerShell quoting: double any inner double quotes. `""like this""` inside a single-quoted `--code` string.
- `using` directives must appear at the top of the snippet.
- Replace `MyScript` / `MyNamespace.MyScript` with the actual type name of the UdonSharpBehaviour subclass in your project.
- **Prefer `string.Format()` over `$"..."` string interpolation**: C# string interpolation (`$"..."`) works in principle but is harder to read and debug when converted to PowerShell quoting. All code examples in this reference use `string.Format()` for clarity.

---

## 1. Add UdonSharp Component to GameObject

Attach a UdonSharpBehaviour subclass to a selected GameObject, creating the backing UdonBehaviour automatically.

> **Prerequisite**: Before attaching, the UdonSharpProgramAsset for the script must exist. For new scripts, run `UdonSharpCompilerV1.CompileSync()` and check if a ProgramAsset was auto-generated (see section 12). If not, create it manually (see section 10). Attaching without a ProgramAsset causes `NullReferenceException`.
>
> **Security restriction**: `System.Type.GetType()` is blocked by `uloop execute-dynamic-code`. Use the generic method `AddUdonSharpComponent<T>()` instead of the `Type.GetType` + `Undo.AddComponent` approach.

### Add by generic type (recommended)

Use this when the type is known at edit time. This is the simplest and most reliable method.

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;

// Replace TestCounter with the actual UdonSharpBehaviour subclass type
// Add a using directive for the namespace if needed (e.g., using TestScripts;)
GameObject obj = GameObject.Find("MyObject");
if (obj == null) return "GameObject not found";

var proxy = obj.AddUdonSharpComponent<TestCounter>();
return string.Format("Added TestCounter to {0}", obj.name);
```

### Add by generic type to selected GameObject

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEditor;
using UnityEngine;

GameObject obj = Selection.activeGameObject;
if (obj == null) return "No GameObject selected";

var proxy = UdonSharpComponentExtensions.AddUdonSharpComponent<InteractToggle>(obj);
return string.Format("Added InteractToggle to {0}", obj.name);
```

### Add by Type.GetType (NOT recommended — blocked by execute-dynamic-code security)

This approach uses `System.Type.GetType()` which is blocked by `uloop execute-dynamic-code` security restrictions. Shown here for reference only. **Use the generic method above instead.**

```csharp
// WARNING: System.Type.GetType is BLOCKED by execute-dynamic-code security!
// Use AddUdonSharpComponent<T>() instead.
using UdonSharp;
using UdonSharpEditor;
using UnityEditor;
using UnityEngine;

GameObject obj = Selection.activeGameObject;
if (obj == null) return "No GameObject selected";

System.Type scriptType = System.Type.GetType("MyScript, Assembly-CSharp");
if (scriptType == null) return "Type MyScript not found";

var proxy = (UdonSharpBehaviour)Undo.AddComponent(obj, scriptType);
UdonSharpEditorUtility.RunBehaviourSetupWithUndo(proxy);

if (EditorApplication.isPlaying)
    UdonSharpEditorUtility.GetBackingUdonBehaviour(proxy).InitializeUdonContent();

return string.Format("Added {0} to {1}", scriptType.Name, obj.name);
```

---

## 2. Read Field Values from UdonSharpBehaviour

Read public field values from a proxy. In Edit mode, fields are read directly from the proxy. In Play mode, sync from backing first.

> **PlayMode limitation**: If the backing UdonBehaviour has not loaded its program (console warning: "Could not load the program"), `CopyUdonToProxy` will throw `NullReferenceException`. In this case, reading fields directly from the proxy will return initial values, not runtime values. See the Troubleshooting section for details.

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;

GameObject obj = GameObject.Find("MyObject");
if (obj == null) return "GameObject not found";

var proxy = obj.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found on object";

if (EditorApplication.isPlaying)
    UdonSharpEditorUtility.CopyUdonToProxy(proxy, ProxySerializationPolicy.All);

// Read public fields via reflection
System.Reflection.FieldInfo field = proxy.GetType().GetField("myPublicField");
if (field == null) return "Field 'myPublicField' not found";

object value = field.GetValue(proxy);
return string.Format("myPublicField = {0}", value);
```

### Read all public fields

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;
using System.Text;

GameObject obj = GameObject.Find("MyObject");
if (obj == null) return "GameObject not found";

var proxy = obj.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found on object";

if (EditorApplication.isPlaying)
    UdonSharpEditorUtility.CopyUdonToProxy(proxy, ProxySerializationPolicy.All);

StringBuilder sb = new StringBuilder();
foreach (var field in proxy.GetType().GetFields(System.Reflection.BindingFlags.Public | System.Reflection.BindingFlags.Instance))
{
    if (field.IsDefined(typeof(HideInInspector), false)) continue;
    sb.AppendLine(string.Format("{0} ({1}) = {2}", field.Name, field.FieldType.Name, field.GetValue(proxy)));
}
return sb.ToString();
```

---

## 3. Write Field Values to UdonSharpBehaviour

Set a public field value on the proxy, then sync to backing.

> **PlayMode limitation**: If the backing UdonBehaviour has not loaded its program, `CopyUdonToProxy` and `CopyProxyToUdon` will throw `NullReferenceException`. In this case, writing to proxy fields works but changes will not propagate to the runtime. See the Troubleshooting section for details.

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEditor;
using UnityEngine;

GameObject obj = GameObject.Find("MyObject");
if (obj == null) return "GameObject not found";

var proxy = obj.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found on object";

// Read before writing
if (EditorApplication.isPlaying)
    UdonSharpEditorUtility.CopyUdonToProxy(proxy, ProxySerializationPolicy.All);

// Set the field value
System.Reflection.FieldInfo field = proxy.GetType().GetField("myPublicField");
if (field == null) return "Field 'myPublicField' not found";

Undo.RecordObject(proxy, "Set myPublicField");
field.SetValue(proxy, 42); // Replace 42 with desired value

// Sync proxy -> backing
UdonSharpEditorUtility.CopyProxyToUdon(proxy, ProxySerializationPolicy.All);

// Also mark the backing as dirty for scene persistence
UdonBehaviour backing = UdonSharpEditorUtility.GetBackingUdonBehaviour(proxy);
if (backing != null) EditorUtility.SetDirty(backing);

return string.Format("Set myPublicField = {0}", field.GetValue(proxy));
```

### Set GameObject reference field

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEditor;
using UnityEngine;

GameObject obj = GameObject.Find("MyObject");
if (obj == null) return "GameObject not found";

var proxy = obj.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found";

if (EditorApplication.isPlaying)
    UdonSharpEditorUtility.CopyUdonToProxy(proxy, ProxySerializationPolicy.All);

GameObject targetRef = GameObject.Find("TargetObject");
if (targetRef == null) return "Target reference not found";

System.Reflection.FieldInfo field = proxy.GetType().GetField("targetField");
if (field == null) return "Field 'targetField' not found";

Undo.RecordObject(proxy, "Set targetField");
field.SetValue(proxy, targetRef);

UdonSharpEditorUtility.CopyProxyToUdon(proxy, ProxySerializationPolicy.All);

UdonBehaviour backing = UdonSharpEditorUtility.GetBackingUdonBehaviour(proxy);
if (backing != null) EditorUtility.SetDirty(backing);

return string.Format("Set targetField = {0}", targetRef.name);
```

---

## 4. Get / Check UdonSharpProgramAsset

Get the program asset for a UdonSharpBehaviour and check compile status.

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;

GameObject obj = GameObject.Find("MyObject");
if (obj == null) return "GameObject not found";

var proxy = obj.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found";

UdonSharpProgramAsset programAsset = UdonSharpEditorUtility.GetUdonSharpProgramAsset(proxy);
if (programAsset == null) return "No UdonSharpProgramAsset found for this behaviour";

string info = string.Format("Program Asset: {0}\n", programAsset.name);
info += string.Format("Source Script: {0}\n", programAsset.sourceCsScript != null ? programAsset.sourceCsScript.name : "NULL");
info += string.Format("Script Version: {0}\n", programAsset.ScriptVersion);
info += string.Format("Compiled Version: {0}\n", programAsset.CompiledVersion);
info += string.Format("Sync Mode: {0}\n", programAsset.behaviourSyncMode);
info += string.Format("Has Interact Event: {0}", programAsset.hasInteractEvent);

return info;
```

### Check if any UdonSharp script has errors

```csharp
using UdonSharp;
using UnityEngine;

bool hasErrors = UdonSharpProgramAsset.AnyUdonSharpScriptHasError();
return string.Format("UdonSharp compile errors: {0}", hasErrors);
```

---

## 5. Compile All UdonSharp Scripts

Trigger compilation of all UdonSharp C# scripts to Udon Assembly.

```csharp
using UdonSharp;
using UdonSharp.Compiler;
using UnityEngine;

UdonSharpProgramAsset.CompileAllCsPrograms();
return "UdonSharp compilation triggered (async)";
```

### Synchronous compile (wait for completion)

```csharp
using UdonSharp;
using UdonSharp.Compiler;
using UnityEngine;

UdonSharpCompilerV1.CompileSync();
bool hasErrors = UdonSharpProgramAsset.AnyUdonSharpScriptHasError();
return string.Format("Compile finished. Errors: {0}", hasErrors);
```

---

## 6. Sync Proxy ↔ Backing

### Copy proxy → backing (apply inspector changes to runtime)

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;

GameObject obj = GameObject.Find("MyObject");
if (obj == null) return "GameObject not found";

var proxy = obj.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found";

UdonSharpEditorUtility.CopyProxyToUdon(proxy, ProxySerializationPolicy.All);
return "Proxy data copied to backing UdonBehaviour";
```

### Copy backing → proxy (read runtime state into inspector)

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;

GameObject obj = GameObject.Find("MyObject");
if (obj == null) return "GameObject not found";

var proxy = obj.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found";

UdonSharpEditorUtility.CopyUdonToProxy(proxy, ProxySerializationPolicy.All);

// Optionally read fields after sync
System.Reflection.FieldInfo field = proxy.GetType().GetField("mySyncedField");
object val = field != null ? field.GetValue(proxy) : "N/A";

return string.Format("Backing data copied to proxy. mySyncedField = {0}", val);
```

---

## 7. Remove UdonSharp Component

Safely remove both the proxy and its backing UdonBehaviour.

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;

GameObject obj = GameObject.Find("MyObject");
if (obj == null) return "GameObject not found";

var proxy = obj.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found on object";

string name = proxy.GetType().Name;
UdonSharpEditorUtility.DestroyImmediate(proxy);
return string.Format("Removed {0} and its backing UdonBehaviour from {1}", name, obj.name);
```

### Remove all UdonSharpBehaviours from a GameObject

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;

GameObject obj = Selection.activeGameObject;
if (obj == null) return "No GameObject selected";

UdonSharpBehaviour[] behaviours = obj.GetComponents<UdonSharpBehaviour>();
if (behaviours.Length == 0) return "No UdonSharpBehaviours found";

int count = 0;
foreach (var b in behaviours)
{
    UdonSharpEditorUtility.DestroyImmediate(b);
    count++;
}
return string.Format("Removed {0} UdonSharpBehaviour(s) from {1}", count, obj.name);
```

---

## 8. List All UdonSharpBehaviours in Scene

Enumerate all UdonSharpBehaviour components with their backing UdonBehaviour status.

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;
using UnityEngine.SceneManagement;
using System.Text;

StringBuilder sb = new StringBuilder();
for (int s = 0; s < SceneManager.sceneCount; s++)
{
    Scene scene = SceneManager.GetSceneAt(s);
    if (!scene.isLoaded) continue;

    foreach (GameObject root in scene.GetRootGameObjects())
    {
        foreach (UdonSharpBehaviour usb in root.GetComponentsInChildren<UdonSharpBehaviour>(true))
        {
            UdonBehaviour backing = UdonSharpEditorUtility.GetBackingUdonBehaviour(usb);
            UdonSharpProgramAsset asset = UdonSharpEditorUtility.GetUdonSharpProgramAsset(usb);
            sb.AppendLine(string.Format("{0}/{1}", usb.gameObject.GetPath(), usb.GetType().Name));
sb.AppendLine(string.Format("  Backing: {0}", backing != null ? backing.name : "MISSING"));
sb.AppendLine(string.Format("  ProgramAsset: {0}", asset != null ? asset.name : "NONE"));
sb.AppendLine(string.Format("  Enabled: {0}", usb.enabled));
        }
    }
}
return sb.Length > 0 ? sb.ToString() : "No UdonSharpBehaviours found in scene";
```

> Note: `GameObject.GetPath()` is a Unity extension that may not exist. Use this helper instead:

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;
using UnityEngine.SceneManagement;
using System.Text;

string GetPath(GameObject go)
{
    string path = go.name;
    while (go.transform.parent != null)
    {
        go = go.transform.parent.gameObject;
        path = go.name + "/" + path;
    }
    return path;
}

StringBuilder sb = new StringBuilder();
for (int s = 0; s < SceneManager.sceneCount; s++)
{
    Scene scene = SceneManager.GetSceneAt(s);
    if (!scene.isLoaded) continue;

    foreach (GameObject root in scene.GetRootGameObjects())
    {
        foreach (UdonSharpBehaviour usb in root.GetComponentsInChildren<UdonSharpBehaviour>(true))
        {
            UdonBehaviour backing = UdonSharpEditorUtility.GetBackingUdonBehaviour(usb);
            UdonSharpProgramAsset asset = UdonSharpEditorUtility.GetUdonSharpProgramAsset(usb);
            sb.AppendLine(string.Format("{0}/{1}", GetPath(usb.gameObject), usb.GetType().Name));
sb.AppendLine(string.Format("  Backing: {0}", backing != null ? "OK" : "MISSING"));
sb.AppendLine(string.Format("  ProgramAsset: {0}", asset != null ? asset.name : "NONE"));
sb.AppendLine(string.Format("  Enabled: {0}", usb.enabled));
        }
    }
}
return sb.Length > 0 ? sb.ToString() : "No UdonSharpBehaviours found in scene";
```

---

## 9. Check / Set Sync Mode

### Check sync mode

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;

GameObject obj = GameObject.Find("MyObject");
if (obj == null) return "GameObject not found";

var proxy = obj.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found";

UdonSharpProgramAsset asset = UdonSharpEditorUtility.GetUdonSharpProgramAsset(proxy);
UdonBehaviour backing = UdonSharpEditorUtility.GetBackingUdonBehaviour(proxy);

return string.Format("Asset Sync Mode: {0}\nRuntime Sync Method: {1}", asset.behaviourSyncMode, backing.SyncMethod);
```

---

## 10. Create UdonSharpProgramAsset for a Script

Create a program asset for an existing C# script. This is what the UdonSharpProgramAssetAutoGenerator does automatically on import, but can be done manually.

```csharp
using UdonSharp;
using UnityEditor;
using UnityEngine;

string scriptPath = "Assets/MyScripts/MyScript.cs";
MonoScript script = AssetDatabase.LoadAssetAtPath<MonoScript>(scriptPath);
if (script == null) return string.Format("Script not found at {0}", scriptPath);

System.Type scriptClass = script.GetClass();
if (scriptClass == null) return string.Format("Could not resolve class from {0}", scriptPath);
if (!typeof(UdonSharpBehaviour).IsAssignableFrom(scriptClass)) return string.Format("{0} is not a UdonSharpBehaviour", scriptClass.Name);

if (UdonSharpEditorUtility.GetUdonSharpProgramAsset(scriptClass) != null)
    return string.Format("Program asset already exists for {0}", scriptClass.Name);

string assetPath = System.IO.Path.ChangeExtension(scriptPath, ".asset");
if (AssetDatabase.LoadMainAssetAtPath(assetPath) != null)
    return string.Format("Asset already exists at {0}", assetPath);

UdonSharpProgramAsset programAsset = ScriptableObject.CreateInstance<UdonSharpProgramAsset>();
programAsset.sourceCsScript = script;

AssetDatabase.CreateAsset(programAsset, assetPath);
AssetDatabase.ImportAsset(assetPath, ImportAssetOptions.ForceSynchronousImport);

return string.Format("Created UdonSharpProgramAsset at {0}", assetPath);
```

### Create program asset and compile

```csharp
using UdonSharp;
using UdonSharp.Compiler;
using UnityEditor;
using UnityEngine;

string scriptPath = "Assets/MyScripts/MyScript.cs";
MonoScript script = AssetDatabase.LoadAssetAtPath<MonoScript>(scriptPath);
if (script == null) return string.Format("Script not found at {0}", scriptPath);

System.Type scriptClass = script.GetClass();
if (scriptClass == null || !typeof(UdonSharpBehaviour).IsAssignableFrom(scriptClass))
    return string.Format("Invalid script class at {0}", scriptPath);

if (UdonSharpEditorUtility.GetUdonSharpProgramAsset(scriptClass) != null)
    return string.Format("Program asset already exists for {0}", scriptClass.Name);

string assetPath = System.IO.Path.ChangeExtension(scriptPath, ".asset");
UdonSharpProgramAsset programAsset = ScriptableObject.CreateInstance<UdonSharpProgramAsset>();
programAsset.sourceCsScript = script;
AssetDatabase.CreateAsset(programAsset, assetPath);
AssetDatabase.ImportAsset(assetPath, ImportAssetOptions.ForceSynchronousImport);

UdonSharpCompilerV1.CompileSync();
bool hasErrors = UdonSharpProgramAsset.AnyUdonSharpScriptHasError();
return string.Format("Created asset at {0}. Compiled. Errors: {1}", assetPath, hasErrors);
```

---

## 11. Play Mode Field Read/Write

In Play mode, UdonBehaviour heap values are the source of truth. Always sync before reading and after writing.

> **PlayMode limitation**: If the backing UdonBehaviour has not loaded its program (console warning: "Could not load the program"), `CopyUdonToProxy` and `CopyProxyToUdon` will throw `NullReferenceException`. This typically occurs outside the VRChat client. In this case, you can still read/write proxy fields directly, but values will be initial/inspector values, not runtime values. See the Troubleshooting section for a workaround.

### Read a synced field in Play mode

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;

GameObject obj = GameObject.Find("MyObject");
var proxy = obj?.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found";

// Sync backing -> proxy before reading
UdonSharpEditorUtility.CopyUdonToProxy(proxy, ProxySerializationPolicy.All);

System.Reflection.FieldInfo field = proxy.GetType().GetField("syncedValue");
object val = field?.GetValue(proxy);
return string.Format("syncedValue = {0}", val);
```

### Write a synced field in Play mode

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;

GameObject obj = GameObject.Find("MyObject");
var proxy = obj?.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found";

// Sync backing -> proxy first to get current state
UdonSharpEditorUtility.CopyUdonToProxy(proxy, ProxySerializationPolicy.All);

// Set value
System.Reflection.FieldInfo field = proxy.GetType().GetField("syncedValue");
field?.SetValue(proxy, 100);

// Sync proxy -> backing to push changes to runtime
UdonSharpEditorUtility.CopyProxyToUdon(proxy, ProxySerializationPolicy.All);

return string.Format("Set syncedValue = {0}", field?.GetValue(proxy));
```

---

## 12. List All UdonSharp Script Types and Their Program Assets

Useful for discovering what UdonSharp types exist in the project.

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEditor;
using System.Text;

UdonSharpProgramAsset[] allAssets = UdonSharpProgramAsset.GetAllUdonSharpPrograms();
StringBuilder sb = new StringBuilder();

foreach (UdonSharpProgramAsset asset in allAssets)
{
    string className = asset.sourceCsScript != null ? asset.sourceCsScript.GetClass()?.FullName ?? "unknown" : "NO SCRIPT";
    sb.AppendLine(string.Format("{0}: {1}", asset.name, className));
    sb.AppendLine(string.Format("  ScriptVersion: {0}, CompiledVersion: {1}", asset.ScriptVersion, asset.CompiledVersion));
    sb.AppendLine(string.Format("  SyncMode: {0}, HasInteractEvent: {1}", asset.behaviourSyncMode, asset.hasInteractEvent));
}

return sb.Length > 0 ? sb.ToString() : "No UdonSharpProgramAssets found";
```

---

## 13. Validate and Repair Scene UdonSharpBehaviours

Run the same validation and repair that the editor manager does on scene open.

```csharp
using UdonSharpEditor;
using UdonSharp;
using UnityEngine;
using System.Text;

UdonBehaviour[] allBehaviours = Object.FindObjectsByType<UdonBehaviour>(FindObjectsSortMode.None);
StringBuilder sb = new StringBuilder();
int issuesFixed = 0;

foreach (UdonBehaviour ub in allBehaviours)
{
    if (!UdonSharpEditorUtility.IsUdonSharpBehaviour(ub)) continue;

    UdonSharpBehaviour proxy = UdonSharpEditorUtility.GetProxyBehaviour(ub);

    if (proxy == null)
    {
        sb.AppendLine(string.Format("MISSING PROXY: {0}/{1}", ub.gameObject.name, ub.name));
        continue;
    }

    if (UdonSharpEditorUtility.BehaviourNeedsSetup(proxy))
    {
        sb.AppendLine(string.Format("NEEDS SETUP: {0}/{1}", proxy.gameObject.name, proxy.GetType().Name));
        UdonSharpEditorUtility.RunBehaviourSetup(proxy);
        issuesFixed++;
    }

    UdonSharpProgramAsset asset = UdonSharpEditorUtility.GetUdonSharpProgramAsset(proxy);
    if (asset == null)
        sb.AppendLine(string.Format("NO PROGRAM ASSET: {0}/{1}", proxy.gameObject.name, proxy.GetType().Name));
}

sb.AppendLine(string.Format("\nIssues fixed: {0}", issuesFixed));
return sb.ToString();
```

---

## 14. Repopulate Backing UdonBehaviour Variables from Proxy

Useful when a backing UdonBehaviour's public variables are out of sync with the proxy (e.g., after manual edits to the UdonBehaviour).

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;

GameObject obj = GameObject.Find("MyObject");
var proxy = obj?.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found";

UdonBehaviour backing = UdonSharpEditorUtility.GetBackingUdonBehaviour(proxy);
if (backing == null) return "No backing UdonBehaviour found";

// Full sync: proxy -> backing
UdonSharpEditorUtility.CopyProxyToUdon(proxy, ProxySerializationPolicy.All);

// Force-serialize the backing's public variables
var serializeMethod = typeof(UdonBehaviour).GetMethod("SerializePublicVariables", System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance);
serializeMethod?.Invoke(backing, null);

EditorUtility.SetDirty(backing);
return string.Format("Repopulated backing UdonBehaviour variables for {0}", proxy.GetType().Name);
```

---

## 15. Check UdonSharp Compile Status

```csharp
using UdonSharp;
using UdonSharpEditor;
using UnityEngine;

bool hasErrors = UdonSharpProgramAsset.AnyUdonSharpScriptHasError();

UdonSharpProgramAsset[] allAssets = UdonSharpProgramAsset.GetAllUdonSharpPrograms();
int total = allAssets.Length;
int outOfDate = 0;

foreach (var asset in allAssets)
{
    if (asset.ScriptVersion < UdonSharpProgramVersion.CurrentVersion ||
        asset.CompiledVersion < UdonSharpProgramVersion.CurrentVersion)
        outOfDate++;
}

return string.Format("Compile errors: {0}\nTotal scripts: {1}\nOut of date: {2}", hasErrors, total, outOfDate);
```

---

## Troubleshooting

### NullReferenceException on AddUdonSharpComponent

**Symptom**: `AddUdonSharpComponent<T>()` or `RunBehaviourSetup` throws `NullReferenceException`.

**Cause**: The UdonSharpProgramAsset for the script does not exist. This happens when a new script is added and `CompileSync()` does not auto-generate the ProgramAsset.

**Fix**: Create the ProgramAsset manually (see section 10), then retry the attach.

```csharp
using UdonSharp;
using UdonSharp.Compiler;
using UnityEditor;
using UnityEngine;

string scriptPath = "Assets/Scripts/MyScript.cs";
MonoScript script = AssetDatabase.LoadAssetAtPath<MonoScript>(scriptPath);
System.Type scriptClass = script.GetClass();
if (UdonSharpEditorUtility.GetUdonSharpProgramAsset(scriptClass) == null)
{
    string assetPath = System.IO.Path.ChangeExtension(scriptPath, ".asset");
    UdonSharpProgramAsset programAsset = ScriptableObject.CreateInstance<UdonSharpProgramAsset>();
    programAsset.sourceCsScript = script;
    AssetDatabase.CreateAsset(programAsset, assetPath);
    AssetDatabase.ImportAsset(assetPath, ImportAssetOptions.ForceSynchronousImport);
    UdonSharpCompilerV1.CompileSync();
    return string.Format("Created ProgramAsset at {0}", assetPath);
}
return "ProgramAsset already exists";
```

### DangerousApiCall: Type.GetType blocked

**Symptom**: `DangerousApiCall: Dangerous API detected: Type.GetType` error when using `System.Type.GetType()`.

**Cause**: `uloop execute-dynamic-code` security policy blocks `Type.GetType`.

**Fix**: Use the generic method `AddUdonSharpComponent<T>()` instead (see section 1). When the type is not known at compile time, use `MonoScript.GetClass()` to resolve it.

### CopyUdonToProxy / CopyProxyToUdon NullReferenceException in PlayMode

**Symptom**: `CopyUdonToProxy` or `CopyProxyToUdon` throws `NullReferenceException` during PlayMode, especially with the console warning "Could not load the program; the UdonBehaviour on 'GameObject' will not run."

**Cause**: The backing UdonBehaviour has not loaded its Udon program. This occurs outside the VRChat client where the Udon runtime is not available.

**Workaround**: Read/write proxy fields directly (without sync). Note that values read this way are inspector defaults, not runtime values:

```csharp
using UdonSharp;
using UnityEngine;
using TestScripts; // Replace with your namespace

GameObject obj = GameObject.Find("MyObject");
if (obj == null) return "GameObject not found";

// Get the proxy directly (no CopyUdonToProxy)
var proxy = obj.GetComponent<MyScript>() as UdonSharpBehaviour;
if (proxy == null) return "MyScript not found";

// Read fields via reflection (returns initial/inspector values, not runtime values)
System.Reflection.FieldInfo field = proxy.GetType().GetField("myPublicField");
object value = field != null ? field.GetValue(proxy) : "N/A";
return string.Format("myPublicField = {0} (note: initial value, not runtime)", value);
```

### New script ProgramAsset not auto-generated

**Symptom**: After creating a new UdonSharp script and running `UdonSharpCompilerV1.CompileSync()`, no ProgramAsset appears for the new script.

**Cause**: `CompileSync()` only compiles existing ProgramAssets. New scripts require explicit ProgramAsset creation (what the UdonSharpProgramAssetAutoGenerator normally does on import).

**Fix**: Create the ProgramAsset manually using section 10, then compile.