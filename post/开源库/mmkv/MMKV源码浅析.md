# 前言

MMKV作为腾讯开源的一个KV存储的库，还是很值得我们学习的。在我看来，主要有以下几点好处。

* 解决SharedPreference跨进程的问题
* 解决SharedPreference KV存储对内存性能的影响
* 优秀的读写性能

其实，我们工作中用到KV存储的地方还是很多的。像我以前，做数据存储的时候，用过NOSQL的数据库，但是那个数据库相对来说，修改性能较差。

而跨进程这一优秀的设计，是我们所需要的。

# 初始化

MMKV的初始化分为两步。

```
MMKV#initialize
//其他的创建MMKV对象的方法,如
MMKV#defaultMMKE

```

这里都是调用c++的方法去做初始化和MMKV对象的生成，我们直接看native-bridge的代码。
先看initialize

```
extern "C" JNIEXPORT JNICALL void
Java_com_tencent_mmkv_MMKV_initialize(JNIEnv *env, jobject obj, jstring rootDir) {
    if (!rootDir) {
        return;
    }
    const char *kstr = env->GetStringUTFChars(rootDir, nullptr);
    if (kstr) {
        MMKV::initializeMMKV(kstr);
        env->ReleaseStringUTFChars(rootDir, kstr);
    }
}
```

```
void MMKV::initializeMMKV(const std::string &rootDir) {
    static pthread_once_t once_control = PTHREAD_ONCE_INIT;
    pthread_once(&once_control, initialize);

    g_rootDir = rootDir;
    char *path = strdup(g_rootDir.c_str());
    mkPath(path);
    free(path);

    MMKVInfo("root dir: %s", g_rootDir.c_str());
}
```
* Java_com_tencent_mmkv_MMKV_initialize中调用MMKV的initializeMMKV进行初始化操作。


### initializeMMKV

```
void initialize() {
    g_instanceDic = new unordered_map<std::string, MMKV *>;
    g_instanceLock = ThreadLock();

    //testAESCrypt();

    MMKVInfo("page size:%d", DEFAULT_MMAP_SIZE);
}

void MMKV::initializeMMKV(const std::string &rootDir) {
    static pthread_once_t once_control = PTHREAD_ONCE_INIT;
    pthread_once(&once_control, initialize);

    g_rootDir = rootDir;
    char *path = strdup(g_rootDir.c_str());
    mkPath(path);
    free(path);

    MMKVInfo("root dir: %s", g_rootDir.c_str());
}
```

* 利用pthread_once进行once初始化(这个只会初始化一次),在initialize中，初始化了一个map以及一个ThreadLock(下面会说这个)
* 然后调用mkPath的方法去创建文件夹。

#### ThreadLock

```
ThreadLock::ThreadLock() {
    pthread_mutexattr_t attr;
    pthread_mutexattr_init(&attr);
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);

    pthread_mutex_init(&m_lock, &attr);

    pthread_mutexattr_destroy(&attr);
}

ThreadLock::~ThreadLock() {
    pthread_mutex_destroy(&m_lock);
}

void ThreadLock::lock() {
    auto ret = pthread_mutex_lock(&m_lock);
    if (ret != 0) {
        MMKVError("fail to lock %p, ret=%d, errno=%s", &m_lock, ret, strerror(errno));
    }
}

bool ThreadLock::try_lock() {
    auto ret = pthread_mutex_trylock(&m_lock);
    if (ret != 0) {
        MMKVError("fail to try lock %p, ret=%d, errno=%s", &m_lock, ret, strerror(errno));
    }
    return (ret == 0);
}

void ThreadLock::unlock() {
    auto ret = pthread_mutex_unlock(&m_lock);
    if (ret != 0) {
        MMKVError("fail to unlock %p, ret=%d, errno=%s", &m_lock, ret, strerror(errno));
    }
}
```

简单的对pthread_mutex_t进行初始化，这里是一个可重入锁。

#### mkPath

```
bool mkPath(char *path) {
    struct stat sb = {};
    bool done = false;
    char *slash = path;

    while (!done) {
        slash += strspn(slash, "/");
        slash += strcspn(slash, "/");

        done = (*slash == '\0');
        *slash = '\0';

        if (stat(path, &sb) != 0) {
            if (errno != ENOENT || mkdir(path, 0777) != 0) {
                MMKVWarning("%s", path);
                return false;
            }
        } else if (!S_ISDIR(sb.st_mode)) {
            MMKVWarning("%s: %s", path, strerror(ENOTDIR));
            return false;
        }

        *slash = '/';
    }

    return true;
}
```

* 创建对应的文件夹

### MMKV初始化

先看下DefaultMMKV的代码。
```
MMKV *MMKV::defaultMMKV(MMKVMode mode, string *cryptKey) {
    return mmkvWithID(DEFAULT_MMAP_ID, DEFAULT_MMAP_SIZE, mode, cryptKey);
}
```

* DEFAULT_MMAP_ID 为mmkv.default
* DEFAULT_MMAP_SIZE为操作系统内存页单页大小


接下来看MMKV的初始化代码。

```
MMKV::MMKV(const std::string &mmapID, int size, MMKVMode mode, string *cryptKey)
    : m_mmapID(mmapID)
    , m_path(mappedKVPathWithID(m_mmapID, mode))
    , m_crcPath(crcPathWithID(m_mmapID, mode))
    , m_metaFile(m_crcPath, DEFAULT_MMAP_SIZE, (mode & MMKV_ASHMEM) ? MMAP_ASHMEM : MMAP_FILE)
    , m_crypter(nullptr)
    , m_fileLock(m_metaFile.getFd())
    , m_sharedProcessLock(&m_fileLock, SharedLockType)
    , m_exclusiveProcessLock(&m_fileLock, ExclusiveLockType)
    , m_isInterProcess((mode & MMKV_MULTI_PROCESS) != 0)
    , m_isAshmem((mode & MMKV_ASHMEM) != 0) {
    m_fd = -1;
    m_ptr = nullptr;
    m_size = 0;
    m_actualSize = 0;
    m_output = nullptr;

    if (m_isAshmem) {
        m_ashmemFile = new MmapedFile(m_mmapID, static_cast<size_t>(size), MMAP_ASHMEM);
        m_fd = m_ashmemFile->getFd();
    } else {
        m_ashmemFile = nullptr;
    }

    if (cryptKey && cryptKey->length() > 0) {
        m_crypter = new AESCrypt((const unsigned char *) cryptKey->data(), cryptKey->length());
    }

    m_needLoadFromFile = true;

    m_crcDigest = 0;

    m_sharedProcessLock.m_enable = m_isInterProcess;
    m_exclusiveProcessLock.m_enable = m_isInterProcess;

    // sensitive zone
    {
        SCOPEDLOCK(m_sharedProcessLock);
        loadFromFile();
    }
}
```

* m_path
* m_crcpath
* m_metaFile
* m_fileLock 文件锁
* m_sharedProcessLock 共享进程锁
* m_exclusiveProcessLock 独占进程锁
* 其他的一些参数

我们可以看到，在初始化参数列表里面，有非常多的参数,主要是设置一些不要的文件路径以及初始化相关的几个锁(关于这几个锁，后面说一哈)。 这个方法中，如果是匿名共享模式，则创建匿名共享模式的MmapedFile对象。如果设置了cryptKey，则创建AESCypt，用于对数据进行加解密。

#### mappedKVPathWithID

```
static string mappedKVPathWithID(const string &mmapID, MMKVMode mode) {
    return (mode & MMKV_ASHMEM) == 0 ? g_rootDir + "/" + mmapID
                                     : string(ASHMEM_NAME_DEF) + "/" + mmapID;
}
```

* 如果不是匿名共享模式，则文件路径为 初始化路径/mmapId
* 如果是匿名共享模式，则为 /dev/ashme/mmapId

#### crcPathWithID

```
static string crcPathWithID(const string &mmapID, MMKVMode mode) {
    return (mode & MMKV_ASHMEM) == 0 ? g_rootDir + "/" + mmapID + ".crc" : mmapID + ".crc";
}
```
crc文件路径，不多说。


# 相关的锁

### ScopedLock 区域锁

代码很简单，就是利用。作用域以及构造和析构的特性，实现便利的使用。

```
#ifndef MMKV_SCOPEDLOCK_HPP
#define MMKV_SCOPEDLOCK_HPP

#include "MMKVLog.h"

template <typename T>
class ScopedLock {
    T *m_lock;

    // just forbid it for possibly misuse
    ScopedLock(const ScopedLock<T> &other) = delete;

    ScopedLock &operator=(const ScopedLock<T> &other) = delete;

public:
    ScopedLock(T *oLock) : m_lock(oLock) {
        assert(m_lock);
        lock();
    }

    ~ScopedLock() {
        unlock();
        m_lock = nullptr;
    }

    void lock() {
        if (m_lock) {
            m_lock->lock();
        }
    }

    bool try_lock() {
        if (m_lock) {
            return m_lock->try_lock();
        }
        return false;
    }

    void unlock() {
        if (m_lock) {
            m_lock->unlock();
        }
    }
};

#define SCOPEDLOCK(lock) _SCOPEDLOCK(lock, __COUNTER__)
#define _SCOPEDLOCK(lock, counter) __SCOPEDLOCK(lock, counter)
#define __SCOPEDLOCK(lock, counter) ScopedLock<decltype(lock)> __scopedLock##counter(&lock)

//#include <type_traits>
//#define __SCOPEDLOCK(lock, counter) ScopedLock<std::remove_pointer<decltype(lock)>::type> __scopedLock##counter(lock)

#endif //MMKV_SCOPEDLOCK_HPP
```

### FileLock

MMKV里面通过对fcntl函数的封装实现文件锁。代码不多说了。

### InterProcessLock

进程锁，通过对FileLock锁的封装实现进程锁。代码不多说。

# loadFromeFile

```
void MMKV::loadFromFile() {
    if (m_isAshmem) {
        loadFromAshmem();
        return;
    }

    m_metaInfo.read(m_metaFile.getMemory());

    m_fd = open(m_path.c_str(), O_RDWR | O_CREAT, S_IRWXU);
    if (m_fd < 0) {
        MMKVError("fail to open:%s, %s", m_path.c_str(), strerror(errno));
    } else {
        m_size = 0;
        struct stat st = {0};
        if (fstat(m_fd, &st) != -1) {
            m_size = static_cast<size_t>(st.st_size);
        }
        // round up to (n * pagesize)
        if (m_size < DEFAULT_MMAP_SIZE || (m_size % DEFAULT_MMAP_SIZE != 0)) {
            size_t oldSize = m_size;
            m_size = ((m_size / DEFAULT_MMAP_SIZE) + 1) * DEFAULT_MMAP_SIZE;
            if (ftruncate(m_fd, m_size) != 0) {
                MMKVError("fail to truncate [%s] to size %zu, %s", m_mmapID.c_str(), m_size,
                          strerror(errno));
                m_size = static_cast<size_t>(st.st_size);
            }
            zeroFillFile(m_fd, oldSize, m_size - oldSize);
        }
        m_ptr = (char *) mmap(nullptr, m_size, PROT_READ | PROT_WRITE, MAP_SHARED, m_fd, 0);
        if (m_ptr == MAP_FAILED) {
            MMKVError("fail to mmap [%s], %s", m_mmapID.c_str(), strerror(errno));
        } else {
            memcpy(&m_actualSize, m_ptr, Fixed32Size);
            MMKVInfo("loading [%s] with %zu size in total, file size is %zu", m_mmapID.c_str(),
                     m_actualSize, m_size);
            bool loaded = false;
            if (m_actualSize > 0) {
                if (m_actualSize < m_size && m_actualSize + Fixed32Size <= m_size) {
                    if (checkFileCRCValid()) {
                        MMKVInfo("loading [%s] with crc %u sequence %u", m_mmapID.c_str(),
                                 m_metaInfo.m_crcDigest, m_metaInfo.m_sequence);
                        MMBuffer inputBuffer(m_ptr + Fixed32Size, m_actualSize, MMBufferNoCopy);
                        if (m_crypter) {
                            decryptBuffer(*m_crypter, inputBuffer);
                        }
                        m_dic.clear();
                        MiniPBCoder::decodeMap(m_dic, inputBuffer);
                        m_output = new CodedOutputData(m_ptr + Fixed32Size + m_actualSize,
                                                       m_size - Fixed32Size - m_actualSize);
                        loaded = true;
                    }
                }
            }
            if (!loaded) {
                SCOPEDLOCK(m_exclusiveProcessLock);

                if (m_actualSize > 0) {
                    writeAcutalSize(0);
                }
                m_output = new CodedOutputData(m_ptr + Fixed32Size, m_size - Fixed32Size);
                recaculateCRCDigest();
            }
            MMKVInfo("loaded [%s] with %zu values", m_mmapID.c_str(), m_dic.size());
        }
    }

    if (!isFileValid()) {
        MMKVWarning("[%s] file not valid", m_mmapID.c_str());
    }

    m_needLoadFromFile = false;
}
```

代码略长，逻辑如下:

* 首先读取metainfo。
* 对文件进行对其操作，mmap文件大小为内存页的整数倍
* 读取文件实际大小，
* 创建MMBuffer，如果需要解密，调用decryptBuffer
* 解析MMBuffer到hashmap中。
* 设置CodeOutputData。范围为当前数据(fixed32size+实际大小)到文件末尾
* 如果当前文件没任何数据，
	* 写入实际大小为0
	* 设置CodedOutputData
	* 计算CRC

### decryptBuffer

```
void decryptBuffer(AESCrypt &crypter, MMBuffer &inputBuffer) {
    size_t length = inputBuffer.length();
    MMBuffer tmp(length);

    auto input = (unsigned char *) inputBuffer.getPtr();
    auto output = (unsigned char *) tmp.getPtr();
    crypter.decrypt(input, output, length);

    inputBuffer = std::move(tmp);
}
``` 

用AES(好像是裁剪的openssl中的aes部分)解密之后赋值给inputBuffer。

### MiniPBCoder::decodeMap

对现有的数据进行解析。

```
void MiniPBCoder::decodeOneMap(unordered_map<string, MMBuffer> &dic, size_t size) {
    if (size == 0) {
        auto length = m_inputData->readInt32();
    }
    while (!m_inputData->isAtEnd()) {
        const auto &key = m_inputData->readString();
        if (key.length() > 0) {
            auto value = m_inputData->readData();
            if (value.length() > 0) {
                dic[key] = move(value);
            } else {
                dic.erase(key);
            }
        }
    }
}
```



### recaculateCRCDigest 

直接调用 zlib中的crc32进行crc计算，主要用于对文件进行完整性校验。

# 存储数据

以encodeString为例。


```
extern "C" JNIEXPORT JNICALL jboolean Java_com_tencent_mmkv_MMKV_encodeString(
    JNIEnv *env, jobject obj, jlong handle, jstring oKey, jstring oValue) {
    MMKV *kv = reinterpret_cast<MMKV *>(handle);
    if (kv && oKey && oValue) {
        string key = jstring2string(env, oKey);
        string value = jstring2string(env, oValue);
        return (jboolean) kv->setStringForKey(value, key);
    }
    return (jboolean) false;
}
```

```
bool MMKV::setStringForKey(const std::string &value, const std::string &key) {
    if (key.empty()) {
        return false;
    }
    auto data = MiniPBCoder::encodeDataWithObject(value);
    return setDataForKey(std::move(data), key);
}
```

* 先调用MiniPBCoder::encodeDataWithObject进行序列化
* 然后调用setDataForKey进行存储

```
bool MMKV::setDataForKey(MMBuffer &&data, const std::string &key) {
    if (data.length() == 0 || key.empty()) {
        return false;
    }
    SCOPEDLOCK(m_lock);
    SCOPEDLOCK(m_exclusiveProcessLock);
    checkLoadData();

    // m_dic[key] = std::move(data);
    auto itr = m_dic.find(key);
    if (itr == m_dic.end()) {
        itr = m_dic.emplace(key, std::move(data)).first;
    } else {
        itr->second = std::move(data);
    }

    return appendDataWithKey(itr->second, key);
}
```

* 加锁
* 检查加载了的数据
* 存到map中
* 添加到文件最后

```
bool MMKV::appendDataWithKey(const MMBuffer &data, const std::string &key) {
    size_t keyLength = key.length();
    // size needed to encode the key
    size_t size = keyLength + pbRawVarint32Size((int32_t) keyLength);
    // size needed to encode the value
    size += data.length() + pbRawVarint32Size((int32_t) data.length());

    SCOPEDLOCK(m_exclusiveProcessLock);

    bool hasEnoughSize = ensureMemorySize(size);

    if (!hasEnoughSize || !isFileValid()) {
        return false;
    }
    if (m_actualSize == 0) {
        auto allData = MiniPBCoder::encodeDataWithObject(m_dic);
        if (allData.length() > 0) {
            if (m_crypter) {
                m_crypter->reset();
                auto ptr = (unsigned char *) allData.getPtr();
                m_crypter->encrypt(ptr, ptr, allData.length());
            }
            writeAcutalSize(allData.length());
            m_output->writeRawData(allData); // note: don't write size of data
            recaculateCRCDigest();
            return true;
        }
        return false;
    } else {
        writeAcutalSize(m_actualSize + size);
        m_output->writeString(key);
        m_output->writeData(data); // note: write size of data

        auto ptr = (uint8_t *) m_ptr + Fixed32Size + m_actualSize - size;
        if (m_crypter) {
            m_crypter->encrypt(ptr, ptr, size);
        }
        updateCRCDigest(ptr, size, KeepSequence);

        return true;
    }
}
```

* 计算大小，看一下有没有足够的大小
* 修改文件中所存的实际大小
* 将数据存储到文件中
* 更新crc值

# 总结

代码不是很难，但是却有一些比较好的地方。如下：

* 进程锁
* MINI protobuf

