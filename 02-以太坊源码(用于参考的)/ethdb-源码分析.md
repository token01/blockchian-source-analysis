go-ethereum所有的数据存储在levelDB这个Google开源的KeyValue文件数据库中，整个区块链的所有数据都存储在一个levelDB的数据库中，levelDB支持按照文件大小切分文件的功能，所以我们看到的区块链的数据都是一个一个小文件，其实这些小文件都是同一个levelDB实例。这里简单的看下levelDB的go封装代码。

levelDB官方网站介绍的特点

**特点**：

- key和value都是任意长度的字节数组；
- entry（即一条K-V记录）默认是按照key的字典顺序存储的，当然开发者也可以重载这个排序函数；
- 提供的基本操作接口：Put()、Delete()、Get()、Batch()；
- 支持批量操作以原子操作进行；
- 可以创建数据全景的snapshot(快照)，并允许在快照中查找数据；
- 可以通过前向（或后向）迭代器遍历数据（迭代器会隐含的创建一个snapshot）；
- 自动使用Snappy压缩数据；
- 可移植性；

**限制**：

- 非关系型数据模型（NoSQL），不支持sql语句，也不支持索引；
- 一次只允许一个进程访问一个特定的数据库；
- 没有内置的C/S架构，但开发者可以使用LevelDB库自己封装一个server；


源码所在的目录在ethereum/ethdb目录。代码比较简单， 分为下面三个文件
- dbtest
- leveldb
- memorydb
- remotedb
- batch.go
- database.go                  levelDB的封装代码
- iterator.go		   
- snapshot.go				   快照文件

## leveldb/leveldb.go
看下面的代码，基本上定义了KeyValue数据库的基本操作， Put， Get， Has， Delete等基本操作，levelDB是不支持SQL的，基本可以理解为数据结构里面的Map。

```
package leveldb

import (
	"fmt"
	"strconv"
	"strings"
	"sync"
	"time"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/ethdb"
	"github.com/ethereum/go-ethereum/log"
	"github.com/ethereum/go-ethereum/metrics"
	"github.com/syndtr/goleveldb/leveldb"
	"github.com/syndtr/goleveldb/leveldb/errors"
	"github.com/syndtr/goleveldb/leveldb/filter"
	"github.com/syndtr/goleveldb/leveldb/opt"
	"github.com/syndtr/goleveldb/leveldb/util"
)

const (
	// degradationWarnInterval specifies how often warning should be printed if the
	// leveldb database cannot keep up with requested writes.
	degradationWarnInterval = time.Minute

	// minCache is the minimum amount of memory in megabytes to allocate to leveldb
	// read and write caching, split half and half.
	minCache = 16

	// minHandles is the minimum number of files handles to allocate to the open
	// database files.
	minHandles = 16

	// metricsGatheringInterval specifies the interval to retrieve leveldb database
	// compaction, io and pause stats to report to the user.
	metricsGatheringInterval = 3 * time.Second
)

// Database is a persistent key-value store. Apart from basic data storage
// functionality it also supports batch writes and iterating over the keyspace in
// binary-alphabetical order.
type Database struct {
	fn string      // filename for reporting
	db *leveldb.DB // LevelDB instance

	compTimeMeter      metrics.Meter // Meter for measuring the total time spent in database compaction
	compReadMeter      metrics.Meter // Meter for measuring the data read during compaction
	compWriteMeter     metrics.Meter // Meter for measuring the data written during compaction
	writeDelayNMeter   metrics.Meter // Meter for measuring the write delay number due to database compaction
	writeDelayMeter    metrics.Meter // Meter for measuring the write delay duration due to database compaction
	diskSizeGauge      metrics.Gauge // Gauge for tracking the size of all the levels in the database
	diskReadMeter      metrics.Meter // Meter for measuring the effective amount of data read
	diskWriteMeter     metrics.Meter // Meter for measuring the effective amount of data written
	memCompGauge       metrics.Gauge // Gauge for tracking the number of memory compaction
	level0CompGauge    metrics.Gauge // Gauge for tracking the number of table compaction in level0
	nonlevel0CompGauge metrics.Gauge // Gauge for tracking the number of table compaction in non0 level
	seekCompGauge      metrics.Gauge // Gauge for tracking the number of table compaction caused by read opt

	quitLock sync.Mutex      // Mutex protecting the quit channel access
	quitChan chan chan error // Quit channel to stop the metrics collection before closing the database

	log log.Logger // Contextual logger tracking the database path
}

// New returns a wrapped LevelDB object. The namespace is the prefix that the
// metrics reporting should use for surfacing internal stats.
func New(file string, cache int, handles int, namespace string, readonly bool) (*Database, error) {
	return NewCustom(file, namespace, func(options *opt.Options) {
		// Ensure we have some minimal caching and file guarantees
		if cache < minCache {
			cache = minCache
		}
		if handles < minHandles {
			handles = minHandles
		}
		// Set default options
		options.OpenFilesCacheCapacity = handles
		options.BlockCacheCapacity = cache / 2 * opt.MiB
		options.WriteBuffer = cache / 4 * opt.MiB // Two of these are used internally
		if readonly {
			options.ReadOnly = true
		}
	})
}

// NewCustom returns a wrapped LevelDB object. The namespace is the prefix that the
// metrics reporting should use for surfacing internal stats.
// The customize function allows the caller to modify the leveldb options.
func NewCustom(file string, namespace string, customize func(options *opt.Options)) (*Database, error) {
	options := configureOptions(customize)
	logger := log.New("database", file)
	usedCache := options.GetBlockCacheCapacity() + options.GetWriteBuffer()*2
	logCtx := []interface{}{"cache", common.StorageSize(usedCache), "handles", options.GetOpenFilesCacheCapacity()}
	if options.ReadOnly {
		logCtx = append(logCtx, "readonly", "true")
	}
	logger.Info("Allocated cache and file handles", logCtx...)

	// Open the db and recover any potential corruptions
	db, err := leveldb.OpenFile(file, options)
	if _, corrupted := err.(*errors.ErrCorrupted); corrupted {
		db, err = leveldb.RecoverFile(file, nil)
	}
	if err != nil {
		return nil, err
	}
	// Assemble the wrapper with all the registered metrics
	ldb := &Database{
		fn:       file,
		db:       db,
		log:      logger,
		quitChan: make(chan chan error),
	}
	ldb.compTimeMeter = metrics.NewRegisteredMeter(namespace+"compact/time", nil)
	ldb.compReadMeter = metrics.NewRegisteredMeter(namespace+"compact/input", nil)
	ldb.compWriteMeter = metrics.NewRegisteredMeter(namespace+"compact/output", nil)
	ldb.diskSizeGauge = metrics.NewRegisteredGauge(namespace+"disk/size", nil)
	ldb.diskReadMeter = metrics.NewRegisteredMeter(namespace+"disk/read", nil)
	ldb.diskWriteMeter = metrics.NewRegisteredMeter(namespace+"disk/write", nil)
	ldb.writeDelayMeter = metrics.NewRegisteredMeter(namespace+"compact/writedelay/duration", nil)
	ldb.writeDelayNMeter = metrics.NewRegisteredMeter(namespace+"compact/writedelay/counter", nil)
	ldb.memCompGauge = metrics.NewRegisteredGauge(namespace+"compact/memory", nil)
	ldb.level0CompGauge = metrics.NewRegisteredGauge(namespace+"compact/level0", nil)
	ldb.nonlevel0CompGauge = metrics.NewRegisteredGauge(namespace+"compact/nonlevel0", nil)
	ldb.seekCompGauge = metrics.NewRegisteredGauge(namespace+"compact/seek", nil)

	// Start up the metrics gathering and return
	go ldb.meter(metricsGatheringInterval)
	return ldb, nil
}

// configureOptions sets some default options, then runs the provided setter.
func configureOptions(customizeFn func(*opt.Options)) *opt.Options {
	// Set default options
	options := &opt.Options{
		Filter:                 filter.NewBloomFilter(10),
		DisableSeeksCompaction: true,
	}
	// Allow caller to make custom modifications to the options
	if customizeFn != nil {
		customizeFn(options)
	}
	return options
}

// Close stops the metrics collection, flushes any pending data to disk and closes
// all io accesses to the underlying key-value store.
func (db *Database) Close() error {
	db.quitLock.Lock()
	defer db.quitLock.Unlock()

	if db.quitChan != nil {
		errc := make(chan error)
		db.quitChan <- errc
		if err := <-errc; err != nil {
			db.log.Error("Metrics collection failed", "err", err)
		}
		db.quitChan = nil
	}
	return db.db.Close()
}

// Has retrieves if a key is present in the key-value store.
func (db *Database) Has(key []byte) (bool, error) {
	return db.db.Has(key, nil)
}

// Get retrieves the given key if it's present in the key-value store.
func (db *Database) Get(key []byte) ([]byte, error) {
	dat, err := db.db.Get(key, nil)
	if err != nil {
		return nil, err
	}
	return dat, nil
}

// Put inserts the given value into the key-value store.
func (db *Database) Put(key []byte, value []byte) error {
	return db.db.Put(key, value, nil)
}

// Delete removes the key from the key-value store.
func (db *Database) Delete(key []byte) error {
	return db.db.Delete(key, nil)
}
```
## database.go
这个就是实际ethereum客户端使用的代码， 封装了levelDB的接口。 

	
	import (
		"strconv"
		"strings"
		"sync"
		"time"
	
		"github.com/ethereum/go-ethereum/log"
		"github.com/ethereum/go-ethereum/metrics"
		"github.com/syndtr/goleveldb/leveldb"
		"github.com/syndtr/goleveldb/leveldb/errors"
		"github.com/syndtr/goleveldb/leveldb/filter"
		"github.com/syndtr/goleveldb/leveldb/iterator"
		"github.com/syndtr/goleveldb/leveldb/opt"
		gometrics "github.com/rcrowley/go-metrics"
	)

使用了github.com/syndtr/goleveldb/leveldb的leveldb的封装，所以一些使用的文档可以在那里找到。可以看到，数据结构主要增加了很多的Mertrics用来记录数据库的使用情况，增加了quitChan用来处理停止时候的一些情况，这个后面会分析。如果下面代码可能有疑问的地方应该再Filter:                 filter.NewBloomFilter(10)这个可以暂时不用关注，这个是levelDB里面用来进行性能优化的一个选项，可以不用理会。

	
	type LDBDatabase struct {
		fn string      // filename for reporting
		db *leveldb.DB // LevelDB instance
	
		getTimer       gometrics.Timer // Timer for measuring the database get request counts and latencies
		putTimer       gometrics.Timer // Timer for measuring the database put request counts and latencies
		...metrics 
	
		quitLock sync.Mutex      // Mutex protecting the quit channel access
		quitChan chan chan error // Quit channel to stop the metrics collection before closing the database
	
		log log.Logger // Contextual logger tracking the database path
	}
	
	// NewLDBDatabase returns a LevelDB wrapped object.
	func NewLDBDatabase(file string, cache int, handles int) (*LDBDatabase, error) {
		logger := log.New("database", file)
		// Ensure we have some minimal caching and file guarantees
		if cache < 16 {
			cache = 16
		}
		if handles < 16 {
			handles = 16
		}
		logger.Info("Allocated cache and file handles", "cache", cache, "handles", handles)
		// Open the db and recover any potential corruptions
		db, err := leveldb.OpenFile(file, &opt.Options{
			OpenFilesCacheCapacity: handles,
			BlockCacheCapacity:     cache / 2 * opt.MiB,
			WriteBuffer:            cache / 4 * opt.MiB, // Two of these are used internally
			Filter:                 filter.NewBloomFilter(10),
		})
		if _, corrupted := err.(*errors.ErrCorrupted); corrupted {
			db, err = leveldb.RecoverFile(file, nil)
		}
		// (Re)check for errors and abort if opening of the db failed
		if err != nil {
			return nil, err
		}
		return &LDBDatabase{
			fn:  file,
			db:  db,
			log: logger,
		}, nil
	}


再看看下面的Put和Has的代码，因为github.com/syndtr/goleveldb/leveldb封装之后的代码是支持多线程同时访问的，所以下面这些代码是不用使用锁来保护的，这个可以注意一下。这里面大部分的代码都是直接调用leveldb的封装，所以不详细介绍了。 有一个比较有意思的地方是Metrics代码。

	// Put puts the given key / value to the queue
	func (db *LDBDatabase) Put(key []byte, value []byte) error {
		// Measure the database put latency, if requested
		if db.putTimer != nil {
			defer db.putTimer.UpdateSince(time.Now())
		}
		// Generate the data to write to disk, update the meter and write
		//value = rle.Compress(value)
	
		if db.writeMeter != nil {
			db.writeMeter.Mark(int64(len(value)))
		}
		return db.db.Put(key, value, nil)
	}
	
	func (db *LDBDatabase) Has(key []byte) (bool, error) {
		return db.db.Has(key, nil)
	}

### Metrics的处理
之前在创建NewLDBDatabase的时候，并没有初始化内部的很多Mertrics，这个时候Mertrics是为nil的。初始化Mertrics是在Meter方法中。外部传入了一个prefix参数，然后创建了各种Mertrics(具体如何创建Merter，会后续在Meter专题进行分析),然后创建了quitChan。 最后启动了一个线程调用了db.meter方法。
```
func NewCustom(file string, namespace string, customize func(options *opt.Options)) (*Database, error) {
	options := configureOptions(customize)
	logger := log.New("database", file)
	usedCache := options.GetBlockCacheCapacity() + options.GetWriteBuffer()*2
	logCtx := []interface{}{"cache", common.StorageSize(usedCache), "handles", options.GetOpenFilesCacheCapacity()}
	if options.ReadOnly {
		logCtx = append(logCtx, "readonly", "true")
	}
	logger.Info("Allocated cache and file handles", logCtx...)

	// Open the db and recover any potential corruptions
	db, err := leveldb.OpenFile(file, options)
	if _, corrupted := err.(*errors.ErrCorrupted); corrupted {
		db, err = leveldb.RecoverFile(file, nil)
	}
	if err != nil {
		return nil, err
	}
	// Assemble the wrapper with all the registered metrics
	ldb := &Database{
		fn:       file,
		db:       db,
		log:      logger,
		quitChan: make(chan chan error),
	}
	ldb.compTimeMeter = metrics.NewRegisteredMeter(namespace+"compact/time", nil)
	ldb.compReadMeter = metrics.NewRegisteredMeter(namespace+"compact/input", nil)
	ldb.compWriteMeter = metrics.NewRegisteredMeter(namespace+"compact/output", nil)
	ldb.diskSizeGauge = metrics.NewRegisteredGauge(namespace+"disk/size", nil)
	ldb.diskReadMeter = metrics.NewRegisteredMeter(namespace+"disk/read", nil)
	ldb.diskWriteMeter = metrics.NewRegisteredMeter(namespace+"disk/write", nil)
	ldb.writeDelayMeter = metrics.NewRegisteredMeter(namespace+"compact/writedelay/duration", nil)
	ldb.writeDelayNMeter = metrics.NewRegisteredMeter(namespace+"compact/writedelay/counter", nil)
	ldb.memCompGauge = metrics.NewRegisteredGauge(namespace+"compact/memory", nil)
	ldb.level0CompGauge = metrics.NewRegisteredGauge(namespace+"compact/level0", nil)
	ldb.nonlevel0CompGauge = metrics.NewRegisteredGauge(namespace+"compact/nonlevel0", nil)
	ldb.seekCompGauge = metrics.NewRegisteredGauge(namespace+"compact/seek", nil)

	// Start up the metrics gathering and return
	go ldb.meter(metricsGatheringInterval)
	return ldb, nil
}
```

这个方法每3秒钟获取一次leveldb内部的计数器，然后把他们公布到metrics子系统。 这是一个无限循环的方法， 直到quitChan收到了一个退出信号。

	// meter periodically retrieves internal leveldb counters and reports them to
	// the metrics subsystem.
	// This is how a stats table look like (currently):
	//下面的注释就是我们调用 db.db.GetProperty("leveldb.stats")返回的字符串，后续的代码需要解析这个字符串并把信息写入到Meter中。

	//   Compactions
	//    Level |   Tables   |    Size(MB)   |    Time(sec)  |    Read(MB)   |   Write(MB)
	//   -------+------------+---------------+---------------+---------------+---------------
	//      0   |          0 |       0.00000 |       1.27969 |       0.00000 |      12.31098
	//      1   |         85 |     109.27913 |      28.09293 |     213.92493 |     214.26294
	//      2   |        523 |    1000.37159 |       7.26059 |      66.86342 |      66.77884
	//      3   |        570 |    1113.18458 |       0.00000 |       0.00000 |       0.00000
	
	func (db *LDBDatabase) meter(refresh time.Duration) {
		// Create the counters to store current and previous values
		counters := make([][]float64, 2)
		for i := 0; i < 2; i++ {
			counters[i] = make([]float64, 3)
		}
		// Iterate ad infinitum and collect the stats
		for i := 1; ; i++ {
			// Retrieve the database stats
			stats, err := db.db.GetProperty("leveldb.stats")
			if err != nil {
				db.log.Error("Failed to read database stats", "err", err)
				return
			}
			// Find the compaction table, skip the header
			lines := strings.Split(stats, "\n")
			for len(lines) > 0 && strings.TrimSpace(lines[0]) != "Compactions" {
				lines = lines[1:]
			}
			if len(lines) <= 3 {
				db.log.Error("Compaction table not found")
				return
			}
			lines = lines[3:]
	
			// Iterate over all the table rows, and accumulate the entries
			for j := 0; j < len(counters[i%2]); j++ {
				counters[i%2][j] = 0
			}
			for _, line := range lines {
				parts := strings.Split(line, "|")
				if len(parts) != 6 {
					break
				}
				for idx, counter := range parts[3:] {
					value, err := strconv.ParseFloat(strings.TrimSpace(counter), 64)
					if err != nil {
						db.log.Error("Compaction entry parsing failed", "err", err)
						return
					}
					counters[i%2][idx] += value
				}
			}
			// Update all the requested meters
			if db.compTimeMeter != nil {
				db.compTimeMeter.Mark(int64((counters[i%2][0] - counters[(i-1)%2][0]) * 1000 * 1000 * 1000))
			}
			if db.compReadMeter != nil {
				db.compReadMeter.Mark(int64((counters[i%2][1] - counters[(i-1)%2][1]) * 1024 * 1024))
			}
			if db.compWriteMeter != nil {
				db.compWriteMeter.Mark(int64((counters[i%2][2] - counters[(i-1)%2][2]) * 1024 * 1024))
			}
			// Sleep a bit, then repeat the stats collection
			select {
			case errc := <-db.quitChan:
				// Quit requesting, stop hammering the database
				errc <- nil
				return
	
			case <-time.After(refresh):
				// Timeout, gather a new set of stats
			}
		}
	}

