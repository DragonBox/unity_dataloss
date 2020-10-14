Statuses memory corruption / JsonUtility
=

tldr; something corrupts the memory, perhaps in Mono AOT, and affects deserialization of objects using SerializeReference in JsonUtility

We are investigating an issue related to memory corruption post deserialization using JsonUtility.

See Unity 1275317 (FIXME LINK MISSING AS ISSUE WAS TEMPORARILY CLOSED BY UNITY)

See also https://forum.unity.com/threads/serializereference-content-of-deserialized-list-partially-modified-post-deserialization.956043/

# Symptoms

We deserialize data from a json file stored on disk.

The data is deserialized into an instance of an class containing 2 serialized fields: 
* one `List<string>` tagged with the SerializeField attribute and
* one `List<Object>` tagged with the SerializeReference attribute.

```
[Serializable]
public class DataPersistenceObjectModel : ISerializationCallbackReceiver
{
	private Dictionary<string, System.Object> _data = new Dictionary<string, object>();
	[SerializeField] List<string> dataKeys = new List<string>();
	[SerializeReference] List<System.Object> dataValues = new List<System.Object>();

	public void OnBeforeSerialize()
	{
		dataKeys.Clear();
		dataValues.Clear();
		dataKeys.AddRange(Data.Keys);
		dataValues.AddRange(Data.Values);
	}

	static ObjectIDGenerator objectIDGenerator = new ObjectIDGenerator();

	public void OnAfterDeserialize()
	{
		if (_data == null)
			_data = new Dictionary<string, System.Object>();
		else
			_data.Clear();

		Dump("AfterDeserialize serialized keys and values", dataKeys, dataValues);

		for (int i = 0; i != System.Math.Min(dataKeys.Count, dataValues.Count); i++)
			_data.Add(dataKeys[i], dataValues[i]);
		dataKeys.Clear();
		dataValues.Clear();

		Dump("AfterDeserialize dictionary", _data.Keys, _data.Values);
	}

	public void Dump(string prefix, IEnumerable<string> keys, IEnumerable<object> values)
	{
		var dump = new StringBuilder("Dump - ").Append(prefix).AppendLine();
		for (var i = 0; i < keys.Count(); i++)
		{
			var key = keys.ElementAt(i);
			var value = values.ElementAt(i);
			var first = false;

			dump.Append("[").Append(i).Append("] ").Append("{").Append(key).Append(':').Append(value).Append("} id:")
				.AppendLine(objectIDGenerator.GetId(value, out first).ToString());
		}

		Debug.Log(dump.ToString());
	}

	public Dictionary<string, System.Object> Data => _data;
}
```

We check the data after we receive the OnAfterDeserialize callback and the data is correct.

Later during the runtime, we find out the data in the List<Object> has changed. The change is not always exactly the same, but is repeatable. References appear to be identical as if the memory had been modified: if we use C# [ObjectIdGenerator](https://github.com/microsoft/referencesource/blob/master/mscorlib/system/runtime/serialization/objectidgenerator.cs
) to track the values in the List<Object> we see that their references stay identical.

Here's a log where we dump the List<Object> at 2 different times. Each line logs the object index in the List, the associated key, the value type and the id of the object as identified by a static instance of an ObjectIdGenerator. 

:warning:Note the change of types for objects with ids 1 and 2.:warning:

```
Dump - AfterDeserialize serialized keys and values
[0] {DataPersistence.CurrentMajorVersion:DataPersistenceCleaner+VersionState} id:1
[1] {ProfileManagerState:WWTK.School.Profiles.ProfileManager+ProfileManagerState} id:2
[2] {AppVersion:WWTK.AppVersion.AppVersionManager+State} id:3
[3] {LoginAccess.Save:WWTK.LoginAccess.LoginAccessController+LoginAccessSave} id:4
[4] {AppUpdateNotifierState:WWTK.AppVersion.AppUpdateNotifierViewController+AppUpdateNotifierState} id:5
[5] {ActivityReportManager.ReportsToSendQueue:WWTK.School.Pedagogy.Reporting.ActivityReportManager+SaveQueue} id:6
[6] {ActivityReportManager.MaximumReportsReachedOccurences:WWTK.School.Pedagogy.Reporting.ActivityReportManager+SaveReachedCount} id:7

Dump - Invalid Cast: 
[0] {DataPersistence.CurrentMajorVersion:Self} id:1
[1] {ProfileManagerState:Zenject.InjectTypeInfo} id:2
[2] {AppVersion:WWTK.AppVersion.AppVersionManager+State} id:3
[3] {LoginAccess.Save:WWTK.LoginAccess.LoginAccessController+LoginAccessSave} id:4
[4] {AppUpdateNotifierState:WWTK.AppVersion.AppUpdateNotifierViewController+AppUpdateNotifierState} id:5
[5] {ActivityReportManager.ReportsToSendQueue:WWTK.School.Pedagogy.Reporting.ActivityReportManager+SaveQueue} id:6
[6] {ActivityReportManager.MaximumReportsReachedOccurences:WWTK.School.Pedagogy.Reporting.ActivityReportManager+SaveReachedCount} id:7
```

:warning:Conclusion: We assume the memory is somehow modified.:warning:

The problem is to find where and by who...

# Affected systems

## Platforms

Affects: Mac, Android, iOS.

:warning:Does not seem to affect Windows so far.:warning:

Not sure about Linux yet.

## Notable instances

:warning:We manage to reliably reproduce the issue on one particular Mac computer. Running the same build on other Mac does not reliably reproduce the issue or at all.:warning:
We've focused running builds on this computer.

We do not manage to reproduce it in editor yet.

# Main findings so far

The project uses [Zenject Baking](https://github.com/svermeulen/Extenject#reflection-baking). This consists of code weaving that is ran at build time. Note right now it only bakes Zenject classes.

The code for the code weaving can be found [here](https://github.com/svermeulen/Extenject/blob/master/UnityProject/Assets/Plugins/Zenject/OptionalExtras/ReflectionBaking/Common/ReflectionBakingModuleEditor.cs#L125-L160) and [here](https://github.com/svermeulen/Extenject/blob/master/UnityProject/Assets/Plugins/Zenject/OptionalExtras/ReflectionBaking/Common/ReflectionBakingModuleEditor.cs#L437). It is called at runtime [during the zenject 'get inject info'](https://github.com/svermeulen/Extenject/blob/dd91e0099af8092ce7bd49086125f84e529576e9/UnityProject/Assets/Plugins/Zenject/Source/Util/TypeAnalyzer.cs#L175-L192).

1. When we deactivate Zenject Baking it at build time, the problem disappears *on that computer*. It still exist for other users

2. When we keep Zenject Baking at build time, but do not let Zenject call the baked methods and rely on reflection instead the problem disappears.
  
3. If we let Zenject call the baked methods, and log + check the deserialized data consistency before calling any baked method, we detect memory corruption.

4. if we add a trigger to log data during the zenject code prior to calls to baked method, we get a native crash with stack pointing to  [mono_aot_get_cached_class_info](https://github.com/mono/mono/blob/7bf83ecd4ab44b19ca3712d55e9b48dab2591c59/mono/mini/aot-runtime.c#L2758). Full stack below

5. if we add more logging code in different places during zenject 'get inject info', we do not have memory corruption.

## Current assumption on the issue

To summarize, we are able to reliably alter between memory corruption, no memory corruption and native crashes in the Mono AOT depending on whether we activate/deactivate some Zenject reflection optimization code, add/remove logging, etc.

:warning:we believe there's an allocation bug in mono and it is somewhat triggered by the zenject startup that uses reflection. This would explain why on different computers, or different call stacks, we get either no memory corruption or native crashes.:warning:
  
  
# Appendixes

## Native crash 

```
Obtained 46 stack frames.
#0  0x0000011a058d25 in mono_aot_get_cached_class_info
#1  0x0000011a12876b in mono_class_init
#2  0x0000011a133550 in mono_class_is_assignable_from
#3  0x0000011a19ad8f in mono_object_handle_isinst
#4  0x0000011a19ae21 in mono_object_isinst_checked
#5  0x0000011a15daf9 in mono_marshal_isinst_with_cache
#6  0x0000011e489115 in  (wrapper managed-to-native) object:__icall_wrapper_mono_marshal_isinst_with_cache (object,intptr,intptr) {0x7fd224409558} + 0x65 (0x11e4890b0 0x11e4891a5) [0x11a722c80 - Unity Root Domain]
#7  0x00000126ebd87b in  WWTK.School.AppDataStorageInstaller/<>c__DisplayClass2_0:<InstallBindings>b__0 (object) {0x7fd223403400} + 0x24b (0x126ebd630 0x126ebda33) [0x11a722c80 - Unity Root Domain]
#8  0x00000126ebd5af in  (wrapper delegate-invoke) System.Action`1<T_REF>:invoke_void_T (T_REF) {0x6080008b8ea0} + 0xcf (0x126ebd4e0 0x126ebd62b) [0x11a722c80 - Unity Root Domain]
#9  0x00000126e90d63 in  Zenject.DiContainer:InstantiateExplicit (System.Type,bool,System.Collections.Generic.List`1<Zenject.TypeValuePair>,Zenject.InjectContext,object) {0x7fd223b7f7c8} + 0x173 (0x126e90bf0 0x126e90d7f) [0x11a722c80 - Unity Root Domain]
#10 0x00000126e90abb in  Zenject.DiContainer:InstantiateExplicit (System.Type,System.Collections.Generic.List`1<Zenject.TypeValuePair>) {0x7fd223b7ef00} + 0x8b (0x126e90a30 0x126e90ac9) [0x11a722c80 - Unity Root Domain]
#11 0x00000126e98ab3 in  Zenject.Context:InstallInstallers () {0x7fd224251320} + 0x33 (0x126e98a80 0x126e98abc) [0x11a722c80 - Unity Root Domain]
#12 0x0000011e48c16b in  Zenject.ProjectContext:InstantiateAndInitialize () {0x7fd22324d4c8} + 0x23b (0x11e48bf30 0x11e48c1c6) [0x11a722c80 - Unity Root Domain]
#13 0x00000119fd347c in mono_jit_runtime_invoke
#14 0x0000011a195f75 in do_runtime_invoke
#15 0x0000011a195ed3 in mono_runtime_invoke
```
