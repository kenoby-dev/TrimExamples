# RM8 .NET SDK Examples

## Add the following *using* statements 

```
using HP.HPTRIM.SDK;
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
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
    TrimDb.Database.Disconnect();
    TrimDb = null;

    return true;
  }
```

## Working with Record Types

```
  public RecordType GetRecordType(string recordTypeName)
  {
    return GetRecordTypes().First(x => x.Name.Equals(recordTypeName, StringComparison.OrdinalIgnoreCase));
  }

  public IEnumerable<RecordType> GetRecordTypes()
  {
    TrimMainObjectSearch recordTypes = new TrimMainObjectSearch(this.TrimDb, BaseObjectTypes.RecordType);
    recordTypes.SelectAll();

    return recordTypes.Cast<RecordType>();
  }

  public IEnumerable<string> GetRecordTypeNames()
  {
    return GetRecordTypes().Select(a => a.Name);
  }
```

## Working with Classifications

```
  public IEnumerable<Classification> GetClassifications()
  {
    TrimMainObjectSearch classifications = new TrimMainObjectSearch(this.TrimDb, BaseObjectTypes.Classification);
    classifications.SelectAll();

    return classifications.Cast<Classification>();
  }

  public Classification GetClassification(string classificationName)
  {
    return GetClassifications().First(c => c.Name.Equals(classificationName, StringComparison.OrdinalIgnoreCase));
  }

  public IEnumerable<string> GetClassificationNames()
  {
    return GetClassifications().Select(c => c.Name);
  }
```

## Working with Fields

```
  public IEnumerable<FieldDefinition> GetFields()
  {
    TrimMainObjectSearch fields = new TrimMainObjectSearch(this.TrimDb, BaseObjectTypes.FieldDefinition);
    fields.SelectAll();

    return fields.Cast<FieldDefinition>();
  }

  public FieldDefinition GetField(string fieldName)
  {
    return GetFields().First(x => x.Name.Equals(fieldName, StringComparison.OrdinalIgnoreCase));
  }

  public string GetFieldValue(Record record, string fieldName)
  {
    var fieldDef = GetField(fieldName);
    var value = record.GetFieldValue(fieldDef);

    return value.AsString();
  }
```

## Working with Locations

```
  public IEnumerable<Location> GetLocations()
  {
    TrimMainObjectSearch locations = new TrimMainObjectSearch(this.TrimDb, BaseObjectTypes.Location);
    locations.SelectAll();

    return locations.Cast<Location>();
  }

  public Location GetLocation(string location)
  {
    return GetLocations().First(x => x.Name.Equals(location, StringComparison.OrdinalIgnoreCase));
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
    Record results = null;
    TrimMainObjectSearch records = new TrimMainObjectSearch(TrimDb, BaseObjectTypes.Record);
    TrimSearchClause numberSearch = new TrimSearchClause(TrimDb, BaseObjectTypes.Record, SearchClauseIds.RecordNumber);
    numberSearch.SetCriteriaFromString(recordNr);

    records.AddSearchClause(numberSearch);

    if (records.FastCount == 1)
    {
      foreach (Record rec in records)
        results = rec;
    }

    return results;
  }

  public string CreateRecord(
    string recordTypeName,
    string containerNr = null,
    Classification classification = null,
    Location homeLocation = null,
    Location owner = null,
    Location author = null)
  {
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
    Record rec = new Record(this.TrimDb, recordType);

    if (!string.IsNullOrEmpty(containerNr))
    {
      rec.Container = GetRecord(containerNr);
    }

    if (classification != null)
    {
      rec.Classification = classification;
    }

    if (author != null)
    {
      rec.Author = author;
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
    Record rec = GetRecord(recordNr);
    rec.Delete();
    return true;
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
    record.SetNotes(notes, NotesUpdateType.AppendWithUserStamp);
    record.Save();

    return true;
  }

  public bool AttachDocument(Record record, string filePath)
  {
    InputDocument doc = new InputDocument();
    doc.SetAsFile(filePath);

    record.SetDocument(doc, false, false, "Comments when adding a document");
    record.Verify(true);
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
    var isVerified = record.Verify(true);
    return isVerified;
  }
```

