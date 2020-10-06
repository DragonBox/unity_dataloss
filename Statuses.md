Statuses memory corruption / JsonUtility
=

We are investigating an issue related to memory corruption post desynchronization using JsonUtility.

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

The code for the code weaving can be found [here](https://github.com/svermeulen/Extenject/blob/master/UnityProject/Assets/Plugins/Zenject/OptionalExtras/ReflectionBaking/Common/ReflectionBakingModuleEditor.cs#L125-L160) and [here](https://github.com/svermeulen/Extenject/blob/master/UnityProject/Assets/Plugins/Zenject/OptionalExtras/ReflectionBaking/Common/ReflectionBakingModuleEditor.cs#L437).

  :warning:When we deactivate Zenject Baking it at build time, the problem disappears:warning:

  :warning:When we keep Zenject Baking at build time, but do not let Zenject call the baked methods and rely on reflection instead the problem disappears.:warning:
