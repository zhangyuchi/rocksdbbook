### What is a Bloom Filter?
For any arbitrary set of keys, an algorithm may be applied to create a bit array called a Bloom filter. Given an arbitrary key, this bit array may be used to determine if the key *may exist* or *definitely does not exist* in the key set. 

In RocksDB, every SST file contains a Bloom filter, which is used to determine if the file may contain the key we're looking for.

For a more detailed explanation of how Bloom filters work, see this [Wikipedia article](http://en.wikipedia.org/wiki/Bloom_filter).

#### Life Cycle
In RocksDB, each SST file has a corresponding Bloom filter. It is created when the SST file is written to storage, and is stored as part of the associated SST file. Bloom filters are constructed for files in all levels in the same way. 

Bloom filters may only be created from a set of keys - there is no operation to combine Bloom filters. When we combine two SST files, a new Bloom filter is created from the keys of the new file. 

When we open an SST file, the corresponding Bloom filter is also opened and loaded in memory. When the SST file is closed, the Bloom filter is removed from memory. 

To cache the Bloom filter in block cache, use: `BlockBasedTableOptions::cache_index_and_filter_blocks=true,`  

#### Memory Usage
A Bloom filter may only be built if all keys fit in memory. It may seem that building Bloom filters for large SST files is impossible, but a separate Bloom filter is created for each data block, which contains only a small range of keys.
 
Details for the format of block based table filters may be found [here](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format#filter-meta-block).

The block size is defined in `Options::block_size`. The default value is 4K. 

When building an SST file, key-value pairs are added in a sequence. When there are enough key-value pairs ( < 4K), RocksDB creates a Bloom filter for each of them and writes it to file. Therefore, only a small fraction of keys are stored in-memory to build the Bloom filter.

We are working on a new Bloom filter for block based tables that contains a filter for all keys in the SST file. This may improve RocksDB read performance, but will require storing hashes of all keys for building. Details may be found in next section.

### New Bloom filter format
[Our original](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format#filter-meta-block) Bloom filter scheme creates a filter for each data block, and therefore does not require a lot of memory for construction. 

We are working on a new Bloom filter option called "full filter" which contains a filter for all keys in the SST file. This may improve read performance, because it avoids traveling in a complicated SST format. However, it requires more memory to build, because all keys must be in-memory for a given SST file. 

Users will be able to specify which kind of Bloom filter to use in `tableOptions::FilterBlockType`. RocksDB will use the original Bloom filter format by default.

The full filter block is formatted as follows:

    [filter for all keys in SST file]

(1) The filter is a large bit array that may be used to check for all keys in an SST file. 

(2) `num probes` specifies the number of hash functions used to create the Bloom filter. For both the original and full Bloom filter formats, it is attached at the end of each filter.

####Optimization for New Filter Format
(1) Stores hashes of keys in-memory to reduce memory consumption. Users should still think twice before using it for large SST files. 

(2) Write/Read multiple probes (bits generated by different hash functions) in one CPU cache line. Thus write/read on filter is faster.

####Usage of New Bloom Filter
By default, the bloom filter remains the original format. To enable new filter format, you just need to add a parameter when creating FilterPolicy like:

    NewBloomFilterPolicy(10, false).
 
The second parameter means "not use original filter format".
 
When reading filter block, RocksDB could tell the filter format and create the appropriate reader.

####Customize your own FilterPolicy
Original filter policy interface is too fixed and not suitable for new filter format and optimization. We define two more interfaces in filter policy (include/rocksdb/filter_policy.h) :

    FilterBitsBuilder* GetFilterBitsBuilder()
    FilterBitsReader* GetFilterBitsReader(const Slice& contents)
 
In this way, the new filter policy would function as a factory for FilterBitsBuilder and FilterBitsReader (also defined in include/rocksdb/filter_policy.h). FilterBitsBuilder provides interface for key storage and filter generation and FilterBitsReader provides interface to check if a key may exist in filter.

Notice: This two new interface just works for new filter format. Original filter format still use original method to customize.