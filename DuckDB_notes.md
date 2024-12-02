
![image](https://github.com/user-attachments/assets/db71ea75-1622-4c93-a521-8113b587560d)
## CatalogEntry
```
class CatalogEntry {
public:
	CatalogEntry(CatalogType type, Catalog &catalog, string name);
	CatalogEntry(CatalogType type, string name, idx_t oid);
	virtual ~CatalogEntry();

	//! The oid of the entry
	idx_t oid;
	//! The type of this catalog entry
	CatalogType type;
	//! Reference to the catalog set this entry is stored in
	optional_ptr<CatalogSet> set;
	//! The name of the entry
	string name;
	//! Whether or not the object is deleted
	bool deleted;
	//! Whether or not the object is temporary and should not be added to the WAL
	bool temporary;
	//! Whether or not the entry is an internal entry (cannot be deleted, not dumped, etc)
	bool internal;
	//! Timestamp at which the catalog entry was created
	atomic<transaction_t> timestamp;
	//! (optional) comment on this entry
	Value comment;
	//! (optional) extra data associated with this entry
	unordered_map<string, string> tags;

private:
	//! Child entry
	unique_ptr<CatalogEntry> child;
	//! Parent entry (the node that dependents_map this node)
	optional_ptr<CatalogEntry> parent;
```
Each object is stored as a CatalogEntry, which has a unique oid allocated by this "catalog.GetDatabase().GetDatabaseManager().NextOid()". 
The CatalogEntry class serves as the base class for the real entry, such as IndexCatalogEntry, DuckTableEntry and DuckSchemaEntry. 

## DuckTableEntry
```
private:
	//! A reference to the underlying storage unit used for this table
	shared_ptr<DataTable> storage;
	//! Manages dependencies of the individual columns of the table
	ColumnDependencyManager column_dependency_manager;
};
```
The table entry saves its data in the field **storage** with type **DataTable**.

## DataTable
![image](https://github.com/user-attachments/assets/1aeb6425-8880-4f03-af61-a229dfbc4d64)

```
private:
	//! The table info
	shared_ptr<DataTableInfo> info;
	//! The set of physical columns stored by this DataTable
	vector<ColumnDefinition> column_definitions;
	//! Lock for appending entries to the table
	mutex append_lock;
	//! The row groups of the table
	shared_ptr<RowGroupCollection> row_groups;
	//! Whether or not the data table is the root DataTable for this table; the root DataTable is the newest version
	//! that can be appended to
	atomic<bool> is_root;
```
In the **info** duckdb stores a **DataIOManager**.
In the **row_groups** duckdb stores the actual data blocks. 

## RowGroupCollection
```
	//! BlockManager
	BlockManager &block_manager;
	//! The number of rows in the table
	atomic<idx_t> total_rows;
	//! The data table info
	shared_ptr<DataTableInfo> info;
	//! The column types of the row group collection
	vector<LogicalType> types;
	idx_t row_start;
	//! The segment trees holding the various row_groups of the table
	shared_ptr<RowGroupSegmentTree> row_groups;
	//! Table statistics
	TableStatistics stats;
	//! Allocation size, only tracked for appends
	idx_t allocation_size;
```


## TableIOManager
```
//! The database instance of the table
	AttachedDatabase &db;
	//! The table IO manager
	shared_ptr<TableIOManager> table_io_manager;
	//! Lock for modifying the name
	mutex name_lock;
	//! The schema of the table
	string schema;
	//! The name of the table
	string table;
	//! The physical list of indexes of this table
	TableIndexList indexes;
	//! Index storage information of the indexes created by this table
	vector<IndexStorageInfo> index_storage_infos;
	//! Lock held while checkpointing
	StorageLock checkpoint_lock;
```
This is an abstract class, and only one class, i.e. SingleFileTableIOManager, inherits from it.

## SingleFileTableIOManager
It contains a **BlockManager**

## BlockManager
An abstract class, two classes inherites from it. **SingleFileBlockManager** and **InMemoryBlockManager**.

## SingleFileBlockManager
```
private:
	AttachedDatabase &db;
	//! The active DatabaseHeader, either 0 (h1) or 1 (h2)
	uint8_t active_header;
	//! The path where the file is stored
	string path;
	//! The file handle
	unique_ptr<FileHandle> handle;
	//! The buffer used to read/write to the headers
	FileBuffer header_buffer;
	//! The list of free blocks that can be written to currently
	set<block_id_t> free_list;
	//! The list of blocks that were freed since the last checkpoint.
	set<block_id_t> newly_freed_list;
	//! The list of multi-use blocks (i.e. blocks that have >1 reference in the file)
	//! When a multi-use block is marked as modified, the reference count is decreased by 1 instead of directly
	//! Appending the block to the modified_blocks list
	unordered_map<block_id_t, uint32_t> multi_use_blocks;
	//! The list of blocks that will be added to the free list
	unordered_set<block_id_t> modified_blocks;
	//! The current meta block id
	idx_t meta_block;
	//! The current maximum block id, this id will be given away first after the free_list runs out
	block_id_t max_block;
	//! The block id where the free list can be found
	idx_t free_list_id;
	//! The current header iteration count
	uint64_t iteration_count;
	//! The storage manager options
	StorageManagerOptions options;
	//! Lock for performing various operations in the single file block manager
	mutex block_lock;
```
