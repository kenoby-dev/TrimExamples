---
layout: default
---

# RM8 .NET SDK Examples

using HP.HPTRIM.SDK;
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace TrimUtilities
{
  public class Repository
  {
    public Database TrimDb { get; set; }
    public string WorkgroupServer { get; set; }
    public string DatasetId { get; set; }

    public Repository() { }
    public Repository(string workgroupServer, string datasetId)
      : this()
    {
      this.WorkgroupServer = workgroupServer;
      this.DatasetId = datasetId;
    }

    public bool Connect()
    {
      return this.Connect(this.WorkgroupServer, this.DatasetId);
    }

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

    public RecordType GetRecordType(string recordTypeName)
    {
      EnsureDb();

      return GetRecordTypes().First(x => x.Name.Equals(recordTypeName, StringComparison.OrdinalIgnoreCase));
    }

    public IEnumerable<RecordType> GetRecordTypes()
    {
      EnsureDb();

      TrimMainObjectSearch recordTypes = new TrimMainObjectSearch(this.TrimDb, BaseObjectTypes.RecordType);
      recordTypes.SelectAll();

      return recordTypes.Cast<RecordType>();
    }

    public IEnumerable<string> GetRecordTypeNames()
    {
      return GetRecordTypes().Select(a => a.Name);
    }

    public IEnumerable<Classification> GetClassifications()
    {
      EnsureDb();

      TrimMainObjectSearch classifications = new TrimMainObjectSearch(this.TrimDb, BaseObjectTypes.Classification);
      classifications.SelectAll();

      return classifications.Cast<Classification>();
    }

    public Classification GetClassification(string classificationName)
    {
      EnsureDb();

      return GetClassifications().First(c => c.Name.Equals(classificationName, StringComparison.OrdinalIgnoreCase));
    }

    public IEnumerable<string> GetClassificationNames()
    {
      EnsureDb();

      return GetClassifications().Select(c => c.Name);
    }

    public Record GetRecord(string recordNr)
    {
      EnsureDb();
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

    public IEnumerable<FieldDefinition> GetFields()
    {
      EnsureDb();

      TrimMainObjectSearch fields = new TrimMainObjectSearch(this.TrimDb, BaseObjectTypes.FieldDefinition);
      fields.SelectAll();

      return fields.Cast<FieldDefinition>();
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
      var value = record.GetFieldValue(fieldDef);

      return value.AsString();
    }

    public IEnumerable<Location> GetLocations()
    {
      EnsureDb();

      TrimMainObjectSearch locations = new TrimMainObjectSearch(this.TrimDb, BaseObjectTypes.Location);
      locations.SelectAll();

      return locations.Cast<Location>();
    }

    public Location GetLocation(string location)
    {
      EnsureDb();

      return GetLocations().First(x => x.Name.Equals(location, StringComparison.OrdinalIgnoreCase));
    }


    public string CreateContainer(string classificationName, string containerRecordType)
    {
      EnsureDb();

      Classification classification = GetClassification(classificationName);
      RecordType recordType = GetRecordTypes().First(a => a.Name.Equals(containerRecordType, StringComparison.OrdinalIgnoreCase));
      return CreateRecord(recordType, null, classification);
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

    private void EnsureDb()
    {
      if (TrimDb == null || !TrimDb.IsConnected)
      {
        Connect();
      }
    }
  }
}

