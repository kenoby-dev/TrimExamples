# RM8 COM SDK Examples

## Add the following *using* statements 

```
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using TRIMSDK;
```

## Connect to a Workgroup Server and Dataset

```
  public bool Connect(string workgroupServer, string datasetId)
  {
    if (string.IsNullOrEmpty(workgroupServer))
    {
      throw new ArgumentNullException("workgroupServer");
    }

    if (string.IsNullOrEmpty(datasetId))
    {
      throw new ArgumentNullException("datasetId");
    }

    this.TrimDb = new Database();
    this.TrimDb.WorkgroupServerName = workgroupServer;
    this.TrimDb.Id = datasetId;
    this.TrimDb.Connect();

    return true;
  }
```

## Disconnect from RM8 server

```
  public bool Disconnect()
  {
    // Not connected.
    if (TrimDb == null || !TrimDb.IsConnected)
    {
      return true;
    }

    TrimDb.Id = this.DatasetId;
    TrimDb.Disconnect();
    TrimDb = null;

    return true;
  }
```

## Working with Record Types

```
  public RecordType GetRecordType(string recordTypeName)
  {
    EnsureDb();

    return TrimDb.GetRecordType(recordTypeName);
  }

  public IEnumerable<RecordType> GetRecordTypes()
  {
    EnsureDb();

    var list = new List<RecordType>();
    var recTypes = TrimDb.MakeRecordTypes();
    recTypes.SelectAll();

    for (int i = 0; i < recTypes.Count; i++)
    {
      list.Add(recTypes.Item(i));
    }

    return list;
  }

  public IEnumerable<string> GetRecordTypeNames()
  {
    return GetRecordTypes().Select(a => a.Name);
  }
```

## Working with Classifications

```
  public IEnumerable<TRIMSDK.Classification> GetClassifications()
  {
    EnsureDb();

    var list = new List<Classification>();
    var classifications = TrimDb.MakeClassifications();
    classifications.SelectAll();

    for (int i = 0; i < classifications.Count; i++)
    {
      list.Add(classifications.Item(i));
    }

    return list;
  }

  public TRIMSDK.Classification GetClassification(string classificationName)
  {
    EnsureDb();

    return GetClassifications().First(c => c.Name.Equals(classificationName, StringComparison.OrdinalIgnoreCase));
  }

  public IEnumerable<string> GetClassificationNames()
  {
    EnsureDb();

    return GetClassifications().Select(c => c.Name);
  }
```

## Working with Fields

```
  public IEnumerable<FieldDefinition> GetFields()
  {
    EnsureDb();

    var list = new List<FieldDefinition>();
    var fieldDefs = TrimDb.MakeFieldDefinitions();
    fieldDefs.SelectAll();

    for (int i = 0; i < fieldDefs.Count; i++)
    {
      list.Add(fieldDefs.Item(i));
    }

    return list;
  }

  public FieldDefinition GetField(string fieldName)
  {
    EnsureDb();

    return GetFields().First(x => x.Name.Equals(fieldName, StringComparison.OrdinalIgnoreCase));
  }

  public string GetFieldValue(Record record, string fieldName)
  {
    EnsureDb();

    var fieldDef = GetField(fieldName);
    var value = record.GetUserField(fieldDef);

    return value as string;
  }
```

## Working with Locations

```
  public IEnumerable<TRIMSDK.Location> GetLocations()
  {
    var list = new List<Location>();
    var locations = TrimDb.MakeLocations();
    locations.SelectAll();

    for (int i = 0; i < locations.Count; i++)
    {
      list.Add(locations.Item(i));
    }

    return list;
  }

  public Location GetLocation(string location)
  {
    return TrimDb.GetLocation(location);
  }
```

## Working with Containers

```
  public string CreateContainer(string classificationName, string containerRecordType)
  {
    Classification classification = GetClassification(classificationName);
    RecordType recordType = GetRecordTypes().First(a => a.Name.Equals(containerRecordType, StringComparison.OrdinalIgnoreCase));
    return CreateRecord(recordType, null, classification);
  }
```


## Working with Records

```
  public Record GetRecord(string recordNr)
  {
    EnsureDb();

    return TrimDb.GetRecord(recordNr);
  }
    
  public string CreateRecord(
    string recordTypeName,
    string containerNr = null,
    Classification classification = null,
    Location homeLocation = null,
    Location owner = null,
    Location author = null)
  {
    EnsureDb();

    RecordType recordType = GetRecordTypes().First(a => a.Name.Equals(recordTypeName, StringComparison.OrdinalIgnoreCase));
    return CreateRecord(recordType, containerNr, classification, homeLocation, owner, author);
  }

  public string CreateRecord(
    RecordType recordType,
    string containerNr = null,
    Classification classification = null,
    Location homeLocation = null,
    Location owner = null,
    Location author = null)
  {
    EnsureDb();

    Record rec = TrimDb.NewRecord(recordType);

    if (!string.IsNullOrEmpty(containerNr))
    {
      rec.Container = GetRecord(containerNr);
    }

    if (classification != null)
    {
      rec.Classification = classification;
    }

    //if (homeLocation != null)
    //{
      //rec.HomeLoc = homeLocation;
      //rec.SetHomeLocation(homeLocation);
    //}

    if (author != null)
    {
      rec.AuthorLoc = author;
    }

    if (owner != null)
    {
      rec.SetOwnerLocation(owner);
    }

    rec.Title = "This is a test record";
    rec.Save();
    return rec.Number;
  }

  public bool DeleteRecord(string recordNr)
  {
    EnsureDb();

    Record rec = GetRecord(recordNr);
    rec.Delete();
    return true;
  }

  public Location GetCurrentUser()
  {
    EnsureDb();

    return TrimDb.CurrentUser;
  }

  public bool SetLocation(Record record, string locationName)
  {
    var location = GetLocation(locationName);
    record.SetHomeLocation(location);
    record.Save();

    return true;
  }

  public bool SetContainer(Record record, string containerNr)
  {
    var container = GetRecord(containerNr);
    record.SetContainer(container, true);
    record.Save();

    return true;
  }

  public bool SetClassification(Record record, string classificationId)
  {
    var classification = GetClassification(classificationId);
    return SetClassification(record, classification);
  }

  public bool SetClassification(Record record, Classification classification)
  {
    record.Classification = classification;
    record.Save();

    return true;
  }

  public bool SetNotes(Record record, string notes)
  {
    record.SetNotes(notes, nuNotesUpdateType.nuAppendWithUserStamp);
    record.Save();

    return true;
  }

  public bool AttachDocument(Record record, string filePath)
  {
    InputDocument doc = new InputDocument();
    doc.SetAsFile(filePath);

    record.SetDocument(doc, false, false, "Comments when adding a document");
    record.Verify();
    record.Save();

    return true;
  }

  public string ExtractDocument(Record record, string directory, string fileName)
  {
    string fullPath = Path.Combine(directory, fileName) + "." + record.Extension;
    record.GetDocument(fullPath, false, null, null);

    return fullPath;
  }

  public bool Verify(Record record)
  {
    var isVerified = record.Verify();
    return isVerified;
  }
```  
