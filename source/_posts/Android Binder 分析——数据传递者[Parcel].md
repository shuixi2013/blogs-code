title: Android Binder 分析——数据传递者（Parcel）
date: 2015-01-28 20:41:16
updated: 2017-02-07 21:47:50
categories: [Android Framework]
tags: [android]
---

前面 binder 原理和通信模型中在接口实现部分（Bp 和 Bn）中应该看到很多地方都有使用 parcel。这个 android 专门设计用来跨进程传递数据的，实现在 native，java 层有接口（基本上是 jni 马甲）。照例先说下源代码位置（4.4 的）：

```bash
# java parcel （MemoryFile 是封装好的匿名共享内存的接口）
frameworks/base/core/java/os/Parcel.java
frameworks/base/core/java/os/Parcelable.java
frameworks/base/core/java/os/ParcelFileDescriptor.java
frameworks/base/core/java/os/MemoryFile.java

# parcel jni 相关
frameworks/base/core/jni/android_os_Parcel.h
frameworks/base/core/jni/android_os_MemoryFile.h
frameworks/base/core/jni/android_os_Parcel.cpp

# native parcel 实现（Memory 相关的是封装好的匿名共享内存实现）
frameworks/native/include/binder/Parcel.h
frameworks/native/include/binder/IMemory.h
frameworks/native/include/binder/MemoryHeapBase.h
frameworks/native/include/binder/MemoryBase.h
frameworks/native/libs/binder/Parcel.cpp
frameworks/native/libs/binder/Memory.cpp
frameworks/native/libs/binder/MemoryHeapBase.cpp
frameworks/native/libs/binder/MemoryBase.cpp

# kernel binder 驱动
kernel/drivers/staging/android/binder.h
kernel/drivers/staging/android/binder.c
# kernel ahsmem 驱动
kernel/include/linux/ashmem.h
kernel/mm/ashmem.c
```

## 原理
在 java 层 parcel 有个接口叫 Parcelable，和 java 的 Serializable 很像，刚开始我还没搞明白这2个有什么区别（以前对 java 也不太熟）。这里简单说一下， Serializable 是 java 的接口，翻译过来是序列化的意思，就是通过实现这个接口能够让 java 的对象序列化能够永久保存在的存储介质上，然后反序列化就能从存储介质上实列化出 java 对象（通俗点，就是一个 save/load 的功能）。因为保存到了存储介质上，所以是可以跨进程的（一个进程把数据写入文件，另外一个去读）。但是为什么 android 还要搞一个 parcel 出来，是因为 java 的 Serializable 是通过存储介质的，所以速度慢。parcel 是基于内存传递的，比磁盘I/O要块，而且更加轻量级（这个我是从网上看到的，我没研究过 java 的 Serializable 代码）。

parcel 在内存中的结构是一块连续的内存，会动根据需要自动扩展大小（这个设计比较赞，一些对性能要求不是太高、或是小数据的地方，可以不用废脑想分配多大空间）。parcel 传递数据，可以分为3种，传递方式也不一样：

* **小型数据**： 从用户空间（源进程）copy 到 kernel 空间（binder 驱动中）再写回用户空间（目标进程，binder 驱动负责寻找目标进程）。
* **大型数据**： 使用 android 的匿名共享内存（Ashmem）传递
* **binder 对象**： kernel binder 驱动专门处理

下面逐一分析。这里我打算从 natvie 到 kernel 再到 java 的顺序进行，因为接着前面通信原型那里，所以从 natvie 开始会比较好，而且实现的地方也在 native。

## 小型数据
先来看看 Parcel.h 中几个比较关键的几个变量：

```cpp
    uint8_t*            mData;
    size_t              mDataSize;
    size_t              mDataCapacity;
    mutable size_t      mDataPos;
    size_t*             mObjects;
    size_t              mObjectsSize;
    size_t              mObjectsCapacity;
    mutable size_t      mNextObjectHint;
```

* mData： 数据指针，也是数据在本进程空间内的内存地址
* mDataSize： 存储的数据大小（使用的空间大小）
* mDataCapacity： 数据空间大小，如果不够的话，可以动态增长
* mDataPos： 数据游标，当前数据的位置，和读文件的游标类似，可以手动设置。声明了 mutbale 属性，可以学习下这个属性应该用在声明地方 ^_^。
* mObjects： `flat_binder_object` 对象的位置数据，注意这个是个指针（其实就是个数组），里面保存的不是数据，而且地址的偏移（后面再具体说）。
* mObjectsSize： 这个简单来说其实就是上面那个 objects 数组的大小。
* mObjectsCapacity： objects 偏移地址（再次强调一次是地址）的空间大小，同样可以动态增长
* mNextObjectHint： 可以理解为 objects 的 dataPos 。

还记得通信模型 IPCThreadState 中有2个 Parcel 变量： mIn、mOut，前面分析这2个东西是 binder 通信的时候打包数据用的。我们通过结合前面的例子来分析。


首先是初始化，IPCThreadState 是直接使用变量的（栈内存），使用默认构造函数：

```cpp
Parcel::Parcel()
{
    initState();
}

void Parcel::initState()
{
    mError = NO_ERROR;
    mData = 0;
    mDataSize = 0;
    mDataCapacity = 0;
    mDataPos = 0;
    ALOGV("initState Setting data size of %p to %d\n", this, mDataSize);
    ALOGV("initState Setting data pos of %p to %d\n", this, mDataPos);
    mObjects = NULL;
    mObjectsSize = 0;
    mObjectsCapacity = 0;
    mNextObjectHint = 0;
    mHasFds = false;
    mFdsKnown = true;
    mAllowFds = true;
    mOwner = NULL;
}
```

初始话很简单，几乎都是初始化为 0（NULL） 的。然后看看 IPCThreadState 使用的初始化：

```cpp
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()),
      mMyThreadId(androidGetTid()),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0)
{
    pthread_setspecific(gTLS, this);
    clearCaller();
    mIn.setDataCapacity(256);
    mOut.setDataCapacity(256);
}
```

首先调用 setDataCapacity 来初始化 parcel 的数据空间大小：

```cpp
status_t Parcel::setDataCapacity(size_t size)
{
    if (size > mDataCapacity) return continueWrite(size);
    return NO_ERROR;
}

status_t Parcel::continueWrite(size_t desired)
{
... ...
    if (mOwner) {
... ...
    } else if (mData) {
... ...
    } else {
        // This is the first data.  Easy!
        uint8_t* data = (uint8_t*)malloc(desired);
        if (!data) {
            mError = NO_MEMORY;
            return NO_MEMORY;
        }

        if(!(mDataCapacity == 0 && mObjects == NULL
             && mObjectsCapacity == 0)) {
            ALOGE("continueWrite: %d/%p/%d/%d", mDataCapacity, mObjects, mObjectsCapacity, desired);
        }

        mData = data;
        mDataSize = mDataPos = 0;
        ALOGV("continueWrite Setting data size of %p to %d\n", this, mDataSize);
        ALOGV("continueWrite Setting data pos of %p to %d\n", this, mDataPos);
        mDataCapacity = desired;
    }

    return NO_ERROR;
}
```

设置空间大小的，其实主要是调用到了 contiueWrite 函数。前面那个 size > mDataCapacity 判断意思是如果设置的大小比原来的要大，则需要调整申请的内存的大小，如果小的话，就直接使用原来的大小。

接下来看 contiueWrite，前面有个 object 的判断先不管。然后下面主要是分开3个分支，分别是：

* 分支一： 如果设置了 release 函数指针（mOwner是个函数指针），调用 release 函数进行处理。
* 分支二： 没有设置 release 函数指针，但是 mData 中存在数据，需要在原来的数据的基础上扩展存储空间。
* 分支三： 没有设置 release 函数指针，并且 mData 中不存在数据（就是注释中说的第一次使用， Easy -_-||），调用 malloc 申请内存块，保存在 mData。设置相应的设置 capacity、size、pos、object 的值。

这里先贴出分支三的代码，第一次使用，是走分支三的，其它2个后面再说。这里注意一点，这里只 malloc 了一个块内存，就是 mData 的，前面说 parcel 存储结构是一块连续的内存，mObjects 只是保存的只是地址的偏移，这里可以看到一些端倪（后面就能清楚）。


初始化了之后，我们看看怎么使用的，在通信模型中我们说道 Bp 端发起 IPC 调用，通过 IPCThreadState 对 binder 驱动写入请求数据发送到 Bn 端，我们回想下 Bp 端写数据的地方：

```cpp
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

... ...
       
    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));
                              
    return NO_ERROR;
}
```

先看前面 writeInt32 这个，对 parcel 写入一个 32bit 的 int 型数据。parcel 接口中有一类是专门针对基本类型（int、float、double、int数组），writeInt32、writeInt64、writeFloat、writeDouble、writeIntArray 这些（对应有 read 接口）。然后他们都是另一个函数：

```cpp
template<class T>
status_t Parcel::writeAligned(T val) {
    COMPILE_TIME_ASSERT_FUNCTION_SCOPE(PAD_SIZE(sizeof(T)) == sizeof(T));

    if ((mDataPos+sizeof(val)) <= mDataCapacity) {
restart_write:
        *reinterpret_cast<T*>(mData+mDataPos) = val;
        return finishWrite(sizeof(val));
    }

    status_t err = growData(sizeof(val));
    if (err == NO_ERROR) goto restart_write;
    return err;
}

status_t Parcel::finishWrite(size_t len)
{
    //printf("Finish write of %d\n", len);
    mDataPos += len;
    ALOGV("finishWrite Setting data pos of %p to %d\n", this, mDataPos);
    if (mDataPos > mDataSize) {
        mDataSize = mDataPos;
        ALOGV("finishWrite Setting data size of %p to %d\n", this, mDataSize);
    }
    //printf("New pos=%d, size=%d\n", mDataPos, mDataSize);
    return NO_ERROR;
}
```

writeAligned 看名字就知道要内存对齐，第一句好像就是验证下是否内存对齐的，好像能够根据编译选项判断，应该是如果打开某个编译选项，如果传过来的 size 没内存对齐直接报错吧，内存对齐的算法都是搞一些位运算（这里好像是4字节对齐吧）：

<pre>
#define PAD_SIZE(s) (((s)+3)&~3)
</pre>

int32、int64、float、double 都是4字节对齐的。接着往下看，有个判断当前 pos + 要写入的数据的所占用的空间是否比 capacity 大，就是看空间是不是够大。前面所了 parcel 能够根据需求自动增长空间，这里我们先看空间够的情况，就是走 if 里面：

<pre>
*reinterpret_cast<T*>(mData+mDataPos) = val;
</pre>

直接取当前地址强制转化指针类型，然后赋值（c/c++语言就是舒服）。然后调用 finishWrite 完成写入。finishWrite 就是把 mDataPos 和 mDataSize 值改了一下（加上刚刚写入数据的大小），从这里可以看得出，对于 write 来说，mDataPos = mDataSize。

然后我们看看当空间不够的情况，就是走 if 后面，有一个 growData 的函数，这个是用来调整内存空间的，然后一个 goto 跳转回 if 里面重写写入（parcel 的实现很多地方有 goto，其实 goto 在本函数里面用还好）。我们来看看 growData：

```cpp
status_t Parcel::growData(size_t len)
{
    size_t newSize = ((mDataSize+len)*3)/2;
    return (newSize <= mDataSize)
            ? (status_t) NO_MEMORY         
            : continueWrite(newSize);      
}
```

这里 parcel 的增长算法： ((mDataSize+len)*3)/2， 带一定预测性的增长，避免频繁的空间调整（每次调整需要重新 malloc 内存的，频繁的话会影响效率的）。然后这里有个判断 newSize < mDataSize 就认为 NO_MEMORY。这是所如果如果溢出了（是负数），就认为申请不到内存了。然后调用的函数是 continueWrite ，和前面 setCapacity 调用的是同一个。前面说这个函数有3个分支，这里我们就可以来看第2个分支了（mData 有数据的情况）：

```cpp
status_t Parcel::continueWrite(size_t desired)
{
    // If shrinking, first adjust for any objects that appear
    // after the new data size.
    size_t objectsSize = mObjectsSize;
    if (desired < mDataSize) {
        if (desired == 0) {
            objectsSize = 0;
        } else {
            while (objectsSize > 0) {
                if (mObjects[objectsSize-1] < desired)
                    break;
                objectsSize--;
            }
        }
    }

    if (mOwner) {
... ...
    } else if (mData) {
        if (objectsSize < mObjectsSize) {
            // Need to release refs on any objects we are dropping.
            const sp<ProcessState> proc(ProcessState::self());
            for (size_t i=objectsSize; i<mObjectsSize; i++) {
                const flat_binder_object* flat
                    = reinterpret_cast<flat_binder_object*>(mData+mObjects[i]);
                if (flat->type == BINDER_TYPE_FD) {
                    // will need to rescan because we may have lopped off the only FDs
                    mFdsKnown = false;
                }
                release_object(proc, *flat, this);
            }
            size_t* objects =
                (size_t*)realloc(mObjects, objectsSize*sizeof(size_t));
            if (objects) {
                mObjects = objects;
            }
            mObjectsSize = objectsSize;
            mNextObjectHint = 0;
        }

        // We own the data, so we can just do a realloc().
        if (desired > mDataCapacity) {
            uint8_t* data = (uint8_t*)realloc(mData, desired);
            if (data) {
                mData = data;
                mDataCapacity = desired;
            } else if (desired > mDataCapacity) {
                mError = NO_MEMORY;
                return NO_MEMORY;
            }
        } else {
            if (mDataSize > desired) {
                mDataSize = desired;
                ALOGV("continueWrite Setting data size of %p to %d\n", this, mDataSize);
            }
            if (mDataPos > desired) {
                mDataPos = desired;
                ALOGV("continueWrite Setting data pos of %p to %d\n", this, mDataPos);
            }
        }

    } else {
... ...
    }

    return NO_ERROR;
}
```

这里有关 object 的处理也是先放放，后面再一起说。看后面的，如果需要的空间比原来的大，那么调用 realloc 把空间调整一下。realloc 可以看 man，是说保留原来的内存空间，然后尝试在原来的空间后面扩展需要的内存空间。然后就是把 mDataPos 和 mDataSize 设置一下。如果是要的空间比原来的小，那就什么都不干，就是说用就当成小的用，内存还是以前那么大。


然后回到 IPCThreadState::writeTransactionData 我们看看后面那个 mOut.write： 

```cpp
status_t Parcel::write(const void* data, size_t len)
{
    void* const d = writeInplace(len);
    if (d) {
        memcpy(d, data, len);
        return NO_ERROR;
    }
    return mError;
}

void* Parcel::writeInplace(size_t len)
{
    // 4字节对齐，看样子字节对齐对效率还是有影响的
    const size_t padded = PAD_SIZE(len); 

    // sanity check for integer overflow
    if (mDataPos+padded < mDataPos) { 
        return NULL;
    }

    if ((mDataPos+padded) <= mDataCapacity) {
restart_write:
        //printf("Writing %ld bytes, padded to %ld\n", len, padded);
        uint8_t* const data = mData+mDataPos;

        // Need to pad at end?
        if (padded != len) {
#if BYTE_ORDER == BIG_ENDIAN
            static const uint32_t mask[4] = {
                0x00000000, 0xffffff00, 0xffff0000, 0xff000000
            };
#endif 
#if BYTE_ORDER == LITTLE_ENDIAN
            static const uint32_t mask[4] = {
                0x00000000, 0x00ffffff, 0x0000ffff, 0x000000ff
            };
#endif 
            //printf("Applying pad mask: %p to %p\n", (void*)mask[padded-len],
            //    *reinterpret_cast<void**>(data+padded-4));
            *reinterpret_cast<uint32_t*>(data+padded-4) &= mask[padded-len];
        }

        finishWrite(padded);
        return data;
    }

    status_t err = growData(padded);
    if (err == NO_ERROR) goto restart_write;
    return NULL;
}
```

这个函数首先调用 writeInplace。来看下 writeInplace，这个函数参数是一个大小，返回是一个地址。进去里面看下，除去字节对齐的部分，就是把 mDataPos 和 mDataSize 的值加上了传过去的 len 大小，然后返回 mData + len 的地址。注意这里也有空间不够的情况，和前面的处理一样，调用 growData 去调整空间（parcel 写的接口基本上都有这个 growData 的处理）。这个函数相当于是帮你把内存分配好，然后返回计算好的起始地址给你。然后回到 write 下面直接 memcpy，把传过来的地址中的数据复制过来。

这2个接口的示例很经典，一个是写基本类型，一个是写对象类型的。基本类型可以说是值类型，直接把值写入内存中；对象类型，是把对象的内存数据写进来，这个相当于 c++ 里面的深拷贝，复制数据。上面 IPCThreadState 写入的是 `binder_transaction_data` 这个结构体，后面具体说说 binder 通信之间的数据格式。现在再来看看 writeString8 这个接口，加深下理解：

```cpp
status_t Parcel::writeString8(const String8& str)
{
    status_t err = writeInt32(str.bytes()); 
    // only write string if its length is more than zero characters,
    // as readString8 will only read if the length field is non-zero.
    // this is slightly different from how writeString16 works.
    if (str.bytes() > 0 && err == NO_ERROR) {
        err = write(str.string(), str.bytes()+1);
    }
    return err;
}
```

String8 是 android 在 native 对 `char*` 封装了一下，有点像 java 的 String方便字符串操作的。这个也是对象类型，看到 parcel 先是把 String8 的 size 写进去，然后 write 把 `char*` 数据写在 size 后面的内存中。我们再来看看对应的 readString8：

```cpp
String8 Parcel::readString8() const
{
    int32_t size = readInt32();
    // watch for potential int overflow adding 1 for trailing NUL
    if (size > 0 && size < INT32_MAX) {
        const char* str = (const char*)readInplace(size+1);
        if (str) return String8(str, size);
    }
    return String8();
}

status_t Parcel::readInt32(int32_t *pArg) const
{
    return readAligned(pArg);
}

template<class T>
status_t Parcel::readAligned(T *pArg) const {
    COMPILE_TIME_ASSERT_FUNCTION_SCOPE(PAD_SIZE(sizeof(T)) == sizeof(T));

    if ((mDataPos+sizeof(T)) <= mDataSize) {
        const void* data = mData+mDataPos;
        mDataPos += sizeof(T);
        *pArg =  *reinterpret_cast<const T*>(data);
        return NO_ERROR;
    } else {
        return NOT_ENOUGH_DATA;        
    }
}

const void* Parcel::readInplace(size_t len) const
{
    if ((mDataPos+PAD_SIZE(len)) >= mDataPos && (mDataPos+PAD_SIZE(len)) <= mDataSize) {
        const void* data = mData+mDataPos;
        mDataPos += PAD_SIZE(len);     
        ALOGV("readInplace Setting data pos of %p to %d\n", this, mDataPos);
        return data;
    }
    return NULL;
}
```

读是先 readInt32 把 write 写入的 size 取出来，readInt32 是调用 readAligned 的。和前面 writeAligned 对应，先是判断下字节对齐，然后直接取 mData + mDataPos 地址的数据，转化成模版类型（再次感叹一次 c、c++ 爽）。注意一下这个函数最后移动了 mDataPos 的位置（对应 read 的数据的大小）。

然后是调用 readInplace ，有了前面的说明，你也应该知道这个是去返回对应的 writeInplace 的地址。里面果然是，同样注意最后移动了 mDataPos 的位置。然后去返回的地址取之前写入的 `char*` 数据，基于上面的数据重新构造出新的 String8 对象。这里你看出 pacrel 和 Serializable 很像，只不过 parcel 是在内存中捣腾，还有后面你会发现 parcel 还为 binder 做了一些别的事情。

还有前面说的 read 的接口回自动移动 mDataPos 的位置（parcel 所有 read 的接口都会自动移动 mDataPos），然后看前面的代码，你会发现，write 之后，到 read 的时候，能否取得到正确的数据，依赖于 mDataPos 的位置。这里就要求 binder 通信的时候，双方在用 parcel 读写数据的时候顺序一定要一致。例如说一个 IPC 调用，传递一个函数的参数： int、float、object，用 parcel 写顺序是： writeInt32、writeFloat、write，那么对方接到传过来的 parcel read 的顺序也必须为： readInt32、readFloat、read。就算其中某些参数你不用，你也要 read 一下，主要是要把 mDataPos 的位置移动对。

这里可以看得出 parcel 只是提供的是一块连续的内存块，至于往里面写什么东西，格式是怎么样的，取决于使用的人，所以使用人要要保证自己读得正确（要和写对应），例如前面说的 String8，前面一个 int32 是大小，后面这个大小的是 `char*` 数据，这个读的人必须按这个格式才重新创建出 String8。这个我们后面看 binder 中的使用能够看得出来。


接下来我们看看 binder 怎么把 parcel 打包的数据传递给另外一个进程的。这里我们结合下通信模型那分析的东西。首先 Bp 端调用 writeTransationData 把 IPC 请求打包发送到 binder 驱动。前面看到打包的是一个 cmd 和一个 `binder_transaction_data` 结构。先来看看这个结构（在 kernel binder.h 中）：

```cpp
struct binder_transaction_data {
    /* The first two are only used for bcTRANSACTION and brTRANSACTION,
     * identifying the target and contents of the transaction.
     */
    union {
        size_t  handle; /* target descriptor of command transaction */
        void    *ptr;   /* target descriptor of return transaction */
    } target;
    void        *cookie;    /* target object cookie */
    unsigned int    code;       /* transaction command */

    /* General information about the transaction. */
    unsigned int    flags;
    pid_t       sender_pid;
    uid_t       sender_euid;
    size_t      data_size;  /* number of bytes of data */
    size_t      offsets_size;   /* number of bytes of offsets */

    /* If this transaction is inline, the data immediately
     * follows here; otherwise, it ends with a pointer to
     * the data buffer.
     */
    union {
        struct {
            /* transaction data */
            const void  *buffer;
            /* offsets from buffer to flat_binder_object structs */
            const void  *offsets;
        } ptr;
        uint8_t buf[8];
    } data;
};
```

然后我们看看 Bp 端对这个结构体填充了什么：

```cpp
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = statusBuffer;
        tr.offsets_size = 0;
        tr.data.ptr.offsets = NULL;
    } else {
        return (mLastError = err);
    }

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```

`binder_transatcion_data` target 这个 union 先看注释的说明， target.handle 是说是 IPC 目标的标示，这个 handle 这个东西后面再细说。code 是 IPC 接口定义的接口的标示（例如 `START_ACTIVITY`, `GET_TASK` 之类的玩意）。然后是检查下 parcel 的错误状态，一般是没啥错误的。然后后面几个赋值，来看看从 parcel 取出的是什么东西：

```cpp
const uint8_t* Parcel::ipcData() const
{
    return mData;
}

size_t Parcel::ipcDataSize() const
{
    return (mDataSize > mDataPos ? mDataSize : mDataPos);
}

const size_t* Parcel::ipcObjects() const
{
    return mObjects;
}

size_t Parcel::ipcObjectsCount() const
{
    return mObjectsSize;
}
```

看注释 `binder_transatcion_data` 的 data 这个变量是个 union，远程传输的时候用的是 ptr 这个结构，里面保存的是数据的地址。ptr.buffer 是 parcel 的 ipcData() ，这个函数返回的是 mData 就是数据地址。注意一下这里 data.ptr 保存的是 IPC Bp 传入的那个 parcel，不是 IPCThreadState mOut 这个用来打包 binder 数据的 parcel（都是用 parcel 容易搞混）。这里 data 这个 parcel 是将 IPC 的接口函数的参数数据打包起来的，例如 int、string 之类的参数。Bn 端返回的数据也是通过 parcel 打包的。而 IPCThreadState 的 mOut 只是写入了 cmd 和 `binder_transatcion_data` 而已，而 `binder_transation_data` 保存了 IPC 中传递的真正数据的地址（从参数 parcel 或取的），仅仅是地址而已。所以开头为什么 mOut 和 mIn 只把空间大小设置为 256，刚开始以为是因为 parcel 可以动态增长空间，先在看来，其实根本用不了到 256，因为数据大小只有一个 int32 的 cmd 和 `binder_transation_data` 这个结构而已。算一下 int32 4字节，`binder_transation_data` 第一个 target union 2个都是4字节的地址，所以就是4字节，除去后面 data 的 union 其余的7个都是4字节的地址，后面那个 data union 算最大的数据，是 4字节x2，所以 `binder_transation_data` 结构占 40字节，加上 cmd 就是 44字节。回去通信模型那看看图，是不是 Bp 端发 `BC_TRANSACTION` write_size 是不是 44。mIn 从 Bn 那读回来的数据也是差不多的，所以 256 足够了，基本上不需要动态调整空间的。

好，回到赋值那，看看后面几个，`data_size` 是取 mDataSize, mDataPos 比较大的那个（估计是为了保险吧，对于写 mDataSize 应该等于 mDataPos），然后看看后面的把 parcel 的 mObjectsSize 和 mObjects 分别给了 `offset_size` 和 ptr.offsets，`offset_size` 还乘了个地址的大小。前面说过了 parcel 的 mObjects 保存的是偏移地址，parcel 的名字很奇怪，kernel 里面的数据结构用名字再次告诉了我们这个是偏移地址。这个到后面就清楚了。


好 mOut 把数据打包好了，到了 waitForResponse 循环调用 talkWithDriver 向 binder 驱动写数据，以及等待 Bn 端返回数据（忘记了的回通信模型看看流程图）。我们先来看第一次写通信命令写了什么东西进去：

```cpp
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD <= 0) {
        return -EBADF; 
    }
        
    binder_write_read bwr;
            
    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
        
    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
            
    bwr.write_size = outAvail;
    bwr.write_buffer = (long unsigned int)mOut.data();
                
    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (long unsigned int)mIn.data();
    } else { 
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
   
... ...

    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        ... ...
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
        ... ...
    } while (err == -EINTR);

... ...

    return err;
}
```

我们在 kernel 的 binder.h 看到 `BINDER_WRITE_READ` 的参数是 `binder_write_read` 这个结构体：

```cpp
#define BINDER_WRITE_READ           _IOWR('b', 1, struct binder_write_read)

/*
 * On 64-bit platforms where user code may run in 32-bits the driver must
 * translate the buffer (and local binder) addresses apropriately.
 */

struct binder_write_read {
    signed long write_size; /* bytes to write */
    signed long write_consumed; /* bytes consumed by driver */ 
    unsigned long   write_buffer;
    signed long read_size;  /* bytes to read */
    signed long read_consumed;  /* bytes consumed by driver */ 
    unsigned long   read_buffer;
};
```

`binder_write_read` 的结构并不复杂，就是一个数据地址，一个数据大小，一个数据确认处理的大小，分为2部分，write 和 read（看注释后面要支持 64bit binder 数据传输这里要改不少东西吧）。回来看下赋值。前面那个那个判断 mIn 中的是否有读的数据，是通过 mDataPos 的位置来判断的，就是说如果 mDataPos 的位置比 mDataSize 小，说明还有数据还没读完，前面说了 parcel 每调用一次 read 接口就会自动移动 mDataPos，如果正好把 read 次数（Bp 端读）对应上 write 次数（Bn 端写），那么 mDataPos 是正好等于 mDataSize 的。后面根据 (!doReceive || needRead) 决定 `write_size` 的大小，这个后面到 kernel 里可以知道 size 的大小是否为 0 决定了是否调用 binder 驱动的读写处理函数。如果 mIn 中还有数据还没读取完，needRead 为 true， doReceive 默认是 true（默认要接收 Bn 端返回的数据），所以如果还有 Bn 端发过来的数据还没读完，本次循环在 binder 驱动中是发不出数据的。这里开始是能没有读数据的，所以能发得出来，`write_size` 大小是 mOut parcel 的 mDataSize，`write_buffer` 是 mOut 的 mData 地址。读的部分相应的取 mIn 的，这里给接收的大小也是 256，后面可以看到 Bn 端发过来也是也是 `binder_transatcion_data` 结构，所以 256 也够了。然后在 ioctl 前把 consumed 都设置成0。


然后就 ioctl 到 kernel 的 binder 驱动里面去了，我在 binder 驱动中看看，parcel 是怎么从 Bp 端传递到 Bn 端（或者从 Bn 返回到 Bp）的。首先是上面的 Bp 向 binder 发送 `BC_TRANSACTION` 把 `binder_transtion_data` 的地址保存到 ioctl 的参数 `binder_write_read` 中：

```cpp
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg) 
{
    int ret;
    struct binder_proc *proc = filp->private_data;
    struct binder_thread *thread;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;
... ...
    switch (cmd) {
    case BINDER_WRITE_READ: {
        struct binder_write_read bwr; 
        if (size != sizeof(struct binder_write_read)) {
            ret = -EINVAL;
            goto err; 
        }    
        if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
            ret = -EFAULT;
            goto err; 
        }  
... ...
        if (bwr.write_size > 0) { 
            ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
            if (ret < 0) { 
                bwr.read_consumed = 0; 
                if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                    ret = -EFAULT;
                goto err; 
            }    
        }    
        if (bwr.read_size > 0) { 
            ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags & O_NONBLOCK);
            if (!list_empty(&proc->todo))
                wake_up_interruptible(&proc->wait);
            if (ret < 0) {
                if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                    ret = -EFAULT;
                goto err;
            }
        }
... ...
        if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
            ret = -EFAULT;
            goto err;
        }
        break;
    }
... ...
    default:
        ret = -EINVAL;
        goto err;
    }
... ...
}
```

这里有个 kernel 函数调用： `copy_from_user`，是从用户空间 copy 指定的一段内存数据到 kernel 空间（用户态（空间），kernel态（空间）有啥区别网上查吧，我也不是很清除），这样 IPCThreadState talkWithDriver 那些填写的那个 `binder_write_read` 就传递到 kernel binder 驱动中了。这里可以看得到，如果 IPCThreadState 把 wirte 或是 read 的 size 设置为 0 的话就不会处理（前面也说过）。我们先看 write， `write_buffer` 里面的数据是 IPCThreadState 用 mOut 打包的内存数据块。根据前面的分析，应该是这样的格式:

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Binder-parcel/1.png)

然后去 `binder_thread_write` 里面去看看：

```cpp
int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
            void __user *buffer, int size, signed long *consumed)
{
    uint32_t cmd;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;

    while (ptr < end && thread->return_error == BR_OK) {
        if (get_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;   
        ptr += sizeof(uint32_t); 
        switch (cmd) {
... ...
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;

            if (copy_from_user(&tr, ptr, sizeof(tr)))
                return -EFAULT;
            ptr += sizeof(tr);
            binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
            break;
        }
... ...
        default:
            printk(KERN_ERR "binder: %d:%d unknown command %d\n",
                   proc->pid, thread->pid, cmd);
            return -EINVAL;
        }
        *consumed = ptr - buffer;
    }
    return 0;
}
```

这里有个 while 循环，结束条件是 ptr 指针移动 end 处，就是处理完 IPCThreadState write 进来的数据为止。循环一开始就用 `get_user` 从 ptr 指向的用户空间出一个 int32 的数据到 kernel 空间（`get_user` 和 `copy_from_user` 的区别是，一个是 copy 一个简单的变量，一个是 copy 一块内存块）。然后接着把 ptr 指针移动一个 int32 大小。这里注意下，前面 ioctl 那 `copy_from_user` 是从用户空间得到 `binder_write_read` 结构（地址在 ioctl 的参数里面），而这里从用户空间 copy 的是保存在 `binder_write_read` `write_buffer` 中的地址，也就是前面 mOut 的 mData 的地址。所以要根据前面打包的格式来读（看上面的图）。前面说了 parcel 的读和写对应。所以这里先取 cmd（是  `BC_TRANSACTION`），然后 parcel 调用 read 接口会自动移动 mDataPos ，binder 驱动里面要自己手动移动指针位置（这里再次看出，parcel 提供简单的内存读写，很灵活，也比较简单，但是同时也比较容易出错）。然后后面继续 `copy_from_user` 从用户态的 mOut 地址把 `binder_transation_data` copy 过来（顺带移动指针），然后交由 `binder_transation` 函数处理（这篇的流程其实和前面通信模型是一样的，但是本篇主要讲数据的流动）。这里先看到后面，`binder_transation` 处理完后， consumed 就被设置为相应读取的数据大小（这个 consumed 是个指针，其实就设置 `binder_write_read` 这个结构 `write_consumed` 这个变量的， `binder_write_read` 这个结构最后又会被传回用户空间去的，后面能看到）。

至于 `binder_transation` 中怎么传递到另一个进程中的去，去看我下一篇 binder 的内存管理篇吧，那里有详细的说明，这里不多说这些。反正最后通过 `binder_thread_read` 传递用 Bn 端的用户空间，然后借着上一篇 Bn 端的 getAndExecuteCommand 从 talkWithDriver 那的 ioctl 返回得到 Bp 端通过 kernel 发送的数据：

```cpp
status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd; 

    result = talkWithDriver();
    if (result >= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN < sizeof(int32_t)) return result;
        // 这里先读 cmd
        cmd = mIn.readInt32();
        IF_LOG_COMMANDS() {
            alog << "Processing top-level Command: "
                 << getReturnString(cmd) << endl;
        }    

        result = executeCommand(cmd);
... ...

    return result;
}
```

Bn 端等待 Bp 端的时候，把自己 mIn 的 parcel 的 buffer 传递到 kernel 里面去了，所以 Bp 端发送过来的 parcel 通过 kernel 传递到 Bn 端的 mIn 中去了。
内存管理篇那里 `binder_thread_read` 会把 cmd 写入 mIn buffer 的第一个 int32 的地址，所以这里先读 int32 的 cmd，然后送给 executeCommand 处理：

```cpp
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch (cmd) {
... ...
    case BR_TRANSACTION:
        {
            binder_transaction_data tr;
            result = mIn.read(&tr, sizeof(tr));
            ALOG_ASSERT(result == NO_ERROR,
                "Not enough command data for brTRANSACTION");
            if (result != NO_ERROR) break;

            Parcel buffer;
            buffer.ipcSetDataReference(
                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                tr.data_size,
                reinterpret_cast<const size_t*>(tr.data.ptr.offsets),
                tr.offsets_size/sizeof(size_t), freeBuffer, this);
... ...

            Parcel reply;
            if (tr.target.ptr) {
                sp<BBinder> b((BBinder*)tr.cookie);
                const status_t error = b->transact(tr.code, buffer, &reply, tr.flags);
                if (error < NO_ERROR) reply.setError(error);

            } else {
                const status_t error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
                if (error < NO_ERROR) reply.setError(error);
            }

            if ((tr.flags & TF_ONE_WAY) == 0) {
                LOG_ONEWAY("Sending reply to %d!", mCallingPid);
                sendReply(reply, 0);
            } else {
                LOG_ONEWAY("NOT sending reply to %d!", mCallingPid);
            }

            mCallingPid = origPid;
            mCallingUid = origUid;

        }
        break;

    default:
        printf("*** BAD COMMAND %d received from Binder driver\n", cmd);
        result = UNKNOWN_ERROR;
        break;
    }

    if (result != NO_ERROR) {
        mLastError = result;
    }

    return result;
}

//======================================

status_t Parcel::read(void* outData, size_t len) const
{
    if ((mDataPos+PAD_SIZE(len)) >= mDataPos && (mDataPos+PAD_SIZE(len)) <= mDataSize) {
        memcpy(outData, mData+mDataPos, len);
        mDataPos += PAD_SIZE(len);     
        ALOGV("read Setting data pos of %p to %d\n", this, mDataPos);
        return NO_ERROR;
    }
    return NOT_ENOUGH_DATA;
}
```

继续看前一篇的那张通信模型的图， Bn 这里是接到的 cmd 是 kernel 发过来的 `BR_TRANSACTION`， 然后前面 Bp 把 `binder_transaction_data` 通过 Parcel 写入，这里就要通过 read 来读出来了。内存流的，直接一个强制转化就行了，read 也很简单，就是 memcpy （可以好好看看内存管理篇，kernel 里面传递 parcel data 的 buffer 的技巧很牛x）。然后这里的 Parcel buffer 是临时变量， ipcSetDataReference 设置 freeBuffer 函数怎么回事，内存管理篇都有讲，这里就不多说了。然后最后 

<pre config="brush:bash;toolbar:false;">
b->transact(tr.code, buffer, &reply, tr.flags)
</pre>

就把 Bp 传递过来的 parcel 传递子类实现 binder 业务的 transact 函数去处理的，顺带，把放返回值的 reply 给传过去了。

我们拿 AM 中的一个简单的接口来看一下（frameworks/base/core/java/android/app/ActivityManagerNative.java）：

```java
    // ActivityManagerNativeProxy: Bp 端
    public boolean finishActivity(IBinder token, int resultCode, Intent resultData)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        // 写入本服务接口的标志，一个字符串，一般是完整的类名 android.xx.xx
        data.writeInterfaceToken(IActivityManager.descriptor);
        // 参数1： flat_binder_object 特殊类型
        data.writeStrongBinder(token);
        // 参数2: int 类型的
        data.writeInt(resultCode);
        if (resultData != null) {
            // 参数3： 一个自定义类型的对象，支持 Parcelable 的
            // 在开始写一个 1，标记一下
            data.writeInt(1);
            resultData.writeToParcel(data, 0);
        } else {
            // 如果传递的参数非法，标记写0
            data.writeInt(0);
        }    
        mRemote.transact(FINISH_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        boolean res = reply.readInt() != 0;
        data.recycle();
        reply.recycle();
        return res; 
    }

    //============================================

    // ActivityManagerNative： Bn 端
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags) 
            throws RemoteException {       
        switch (code) {
... ...
        case FINISH_ACTIVITY_TRANSACTION: {
            // 先读接口标志，对比下是不是本服务的 Bp 端发过来的请求
            data.enforceInterface(IActivityManager.descriptor);
            // 参数1： 特殊 flat_binder_object
            IBinder token = data.readStrongBinder();
            Intent resultData = null;
            // 参数2： int 类型
            int resultCode = data.readInt();
            // 参数3： Parcelable 类型自定义对象
            // 前面如果传递的参数2合法，这里读出的就是 1
            if (data.readInt() != 0) {
                // 把内存流的 parcel 传递给实现了 Parcelable 结构的自定义类
                // 通过数据在 Bn 端重新创建一个对象
                resultData = Intent.CREATOR.createFromParcel(data);
            }
            // 参数解析完毕，调用真正的业务函数实现功能
            boolean res = finishActivity(token, resultCode, resultData);
            reply.writeNoException();
            reply.writeInt(res ? 1 : 0);
            return true;
        }
... ...
        }

        return super.onTransact(code, data, reply, flags);
    }
```

上面看代码中的注释就差不多了，顺序都是一一对应的。然后说说那个 Parcelable 这个接口。这个接口最主要就是2个函数：writeToParcel、createFromParcel 这2个，一个相当于是序列化，一个是反序列化。就是自己类自己实现了的，再复杂的对象都可以通过 Parcel 前面的那些基本类型来存储。

## 大型数据
大型数据主要是通过 Parcel 的匿名共享内存（Ashmem）接口来使用的（writeBlob、readBlob），当然你也可以使用 writeInPlace 使用普通内存来传递，效率么，呵呵（还有不能超过 binder 的 1MB 大小的限制哦）。 这个话去看我的那篇专门说 ashmem 的吧。

## binder 对象
Parcel 有一个特殊的结构叫 `flat_binder_object`。这个是专门用来传递 binder 对象的（其实这个在 ashmem 篇里发现这个还可以传递文件描述 fd，咋这只说 binder 句柄）。

这里我略过 Parcel 的 java 接口 和 jni 的马甲了，直接拿 native 的代码说，稍微简洁一些。

我还是以上面 AM 里面那个例子说。里面在 IPC 有个要传递的对象是 IBinder 类型的，通过原型篇的分析，这个就是 binder 对象了，这里说的 binder 句柄传递，就是针对这一类型的数据。这里就是调用了 Parcel 的2个接口，我们先来看看写的那个：

```cpp
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}

status_t flatten_binder(const sp<ProcessState>& proc,
    const sp<IBinder>& binder, Parcel* out)
{
    flat_binder_object obj;

    obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    if (binder != NULL) {
        // 区分是 Bn（localBinder） 还是 Bp（remoteBinder）
        IBinder *local = binder->localBinder();
        if (!local) {
            BpBinder *proxy = binder->remoteBinder();
            if (proxy == NULL) {
                ALOGE("null proxy");
            }
            // 如果是 Bp 的话，则保存 Bp 的 handle 值
            // Bp 的 type 是 BINDER_TYPE_HANDLE
            const int32_t handle = proxy ? proxy->handle() : 0;
            obj.type = BINDER_TYPE_HANDLE;
            obj.handle = handle;
            obj.cookie = NULL;
        } else {
            // 如果是 Bn 的话，直接保存 binder 对象本身
            // Bn 的 type 是 BINDER_TYPE_BINDER
            obj.type = BINDER_TYPE_BINDER;
            obj.binder = local->getWeakRefs();
            obj.cookie = local;
        }
    } else {
        obj.type = BINDER_TYPE_BINDER;
        obj.binder = NULL;
        obj.cookie = NULL;
    }

    // 前面只是设置 flat_binder_object，这个函数才是真正写入 parcel 数据
    return finish_flatten_binder(binder, obj, out);
}
```

Parcel 其实还有 writeWeakBinder，但是这里只管 writeStrongBinder，而且一般也是 strong 用的多。先说说 `flat_binder_object` 这个东西，它是 kernel binder 驱动里面的一个结构：

```cpp
/*
 * This is the flattened representation of a Binder object for transfer
 * between processes.  The 'offsets' supplied as part of a binder transaction
 * contains offsets into the data where these structures occur.  The Binder
 * driver takes care of re-writing the structure type and data as it moves
 * between processes.
 */
struct flat_binder_object {
    /* 8 bytes for large_flat_header. */
    unsigned long       type;
    unsigned long       flags;

    /* 8 bytes of data. */
    union {
        void        *binder;    /* local object */
        signed long handle;     /* remote object */
    };

    /* extra data associated with local object */
    void            *cookie;
};
```

这个注释就已经真相了，这个玩意就是专门拿来传 binder 对象的（前面说了还有 fd），而且 offsets 就是这个东西的在传递数据中的位置，就是前面说的 parcel 中那个 mObjects 其实是个偏移来的啦。然后 binder 和 handle 是一个 union，就是说这个 binder 对象要么是 Bn（local），要么是 Bp（remote）。

那么我接下去看 `finish_flatten_binder`：

```cpp
// 内联函数 -_-||
inline static status_t finish_flatten_binder(
    const sp<IBinder>& binder, const flat_binder_object& flat, Parcel* out)
{
    return out->writeObject(flat, false);
}

status_t Parcel::writeObject(const flat_binder_object& val, bool nullMetaData)
{
    // 这里得同时判断 mData 和 mObjects 够不够咧
    const bool enoughData = (mDataPos+sizeof(val)) <= mDataCapacity;
    const bool enoughObjects = mObjectsSize < mObjectsCapacity;
    // 空间够，就可以写
    if (enoughData && enoughObjects) {
restart_write:
        // 注意这里，flat_binder_object 是保存在 mData 里的
        *reinterpret_cast<flat_binder_object*>(mData+mDataPos) = val;
        
        // Need to write meta-data?    
        if (nullMetaData || val.binder != NULL) {
            // 再注意这里，保存刚刚写 flat_binder_object 的开始的地址
            // 这里就能说明 mObjects 保存的是偏移了。
            mObjects[mObjectsSize] = mDataPos;
            // 给 flat_binder_object 保存的 binder 对象增加引用计数
            acquire_object(ProcessState::self(), val, this);
            // 数据中保存的 object 数量 +1
            mObjectsSize++;
        }
        
        // remember if it's a file descriptor
        if (val.type == BINDER_TYPE_FD) { 
            if (!mAllowFds) {
                return FDS_NOT_ALLOWED;        
            }
            mHasFds = mFdsKnown = true;    
        }

        return finishWrite(sizeof(flat_binder_object));
    }

    // 空间不够就和前面一样，调整内存大小
    if (!enoughData) {
        const status_t err = growData(sizeof(val));
        if (err != NO_ERROR) return err;
    }
    // 这里还有可能保存 offset 的空间不够了
    if (!enoughObjects) {
        size_t newSize = ((mObjectsSize+2)*3)/2;
        size_t* objects = (size_t*)realloc(mObjects, newSize*sizeof(size_t));
        if (objects == NULL) return NO_MEMORY;
        mObjects = objects;
        mObjectsCapacity = newSize;    
    }
       
    // 调整完内存后，重新回去写
    goto restart_write;
}
```

最后是调用 writeObject（object 只能是 `flat_binder_object`） 来写到 parcel 中去。这里和前面的差不多，都得先判断空间够不够，但是这里还得多判断一个 mObjects 的空间够不够。不够的和前面一样调用 growData 去调整大小。这里同样多处理一个 mObjects 空间调整，这里很简单了就是 realloc 一下就行了。这里看代码就知道 `flat_binder_object` 是保存在 mData 的区域的，而且后面几句代码彻底说明了 mObjects 保存的是偏移地址。最后 finishWrite 和前面一样，把 mDataPos 移动一下。

这里多嘴说下 `acquire_object`，我就说我烦 android 的那个啥智能指针，也不是太智能的样子，这里还得显示的增加下引用计数：

```cpp
void Parcel::acquireObjects()
{
    const sp<ProcessState> proc(ProcessState::self());
    size_t i = mObjectsSize;
    uint8_t* const data = mData;
    size_t* const objects = mObjects;
    while (i > 0) {
        i--;
        // 看到 flat_binder_object 怎么取了的没，首地址 + 偏移地址
        const flat_binder_object* flat 
            = reinterpret_cast<flat_binder_object*>(data+objects[i]);
        acquire_object(proc, *flat, this);
    }
}

void acquire_object(const sp<ProcessState>& proc,
    const flat_binder_object& obj, const void* who)
{
    switch (obj.type) {
        case BINDER_TYPE_BINDER:
            if (obj.binder) {
                LOG_REFS("Parcel %p acquiring reference on local %p", who, obj.cookie);
                static_cast<IBinder*>(obj.cookie)->incStrong(who);
            }
            return;
        case BINDER_TYPE_WEAK_BINDER:
            if (obj.binder)
                static_cast<RefBase::weakref_type*>(obj.binder)->incWeak(who);
            return;
        case BINDER_TYPE_HANDLE: {
            const sp<IBinder> b = proc->getStrongProxyForHandle(obj.handle);
            if (b != NULL) {
                LOG_REFS("Parcel %p acquiring reference on remote %p", who, b.get());
                b->incStrong(who);
            }
            return;
        }
        case BINDER_TYPE_WEAK_HANDLE: {
            const wp<IBinder> b = proc->getWeakProxyForHandle(obj.handle);
            if (b != NULL) b.get_refs()->incWeak(who);
            return;
        }
        case BINDER_TYPE_FD: {
            // intentionally blank -- nothing to do to acquire this, but we do
            // recognize it as a legitimate object type.
            return;
        }
    }

    ALOGD("Invalid object type 0x%08lx", obj.type);
}
```

然后 parcel 中内存分配应该是这样的：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Binder-parcel/2.png)

然后打包了 `flat_binder_object` 的 parcel 就传到 kernel 的 binder 驱动里面去了。驱动里面有做特殊处理的，驱动里的处理放到后面一篇说 ServiceManager 那里细说，这里只要知道驱动里面倒腾了一下就到目标进程了，然后目标进程可以使用 parcel 的读接口读到之前写的 binder 对象：

```cpp
sp<IBinder> Parcel::readStrongBinder() const
{
    sp<IBinder> val;
    unflatten_binder(ProcessState::self(), *this, &val);
    return val;
}

// 这个 inline 函数是耍存在感的么 -_-||
inline static status_t finish_unflatten_binder(
    BpBinder* proxy, const flat_binder_object& flat, const Parcel& in)
{
    return NO_ERROR;
}
    
status_t unflatten_binder(const sp<ProcessState>& proc,
    const Parcel& in, sp<IBinder>* out)
{
    const flat_binder_object* flat = in.readObject(false);
    
    if (flat) {
        switch (flat->type) {
            case BINDER_TYPE_BINDER:
                // Bn 本地的直接强转一下
                *out = static_cast<IBinder*>(flat->cookie);
                return finish_unflatten_binder(NULL, *flat, in);
            case BINDER_TYPE_HANDLE:
                // Bp 的话要通过 handle 构造一个远程的代理对象（Bp 对象）
                *out = proc->getStrongProxyForHandle(flat->handle);
                return finish_unflatten_binder(
                    static_cast<BpBinder*>(out->get()), *flat, in);
        }        
    }
    return BAD_TYPE;
} 
```

readStrongBinder 其实挺简单的，是本地的可以直接用，远程的那个 getStrongProxyForHandle 也是放到后面 ServiceManager 再细说。到这里目标进程就收到原始进程传递过来的 binder 对象了，然后可以转化为 binder 的 interface 调用对应的 IPC 接口。

然后最后看下清理的情况，由于 binder 实现的接口中，Parcel 基本都是局部变量，所以 IPC 调用一结束，就会调用析构函数：

```cpp
Parcel::~Parcel()
{
    freeDataNoInit();
}

void Parcel::freeDataNoInit()
{
    if (mOwner) {
        //ALOGI("Freeing data ref of %p (pid=%d)\n", this, getpid());
        mOwner(this, mData, mDataSize, mObjects, mObjectsSize, mOwnerCookie);
    } else {
        // 这里看下面这个分支，mOwner 是 IPC 传递参数的 parcel 专用的
        // 减少引用计数，还得手动减少 -_-||
        releaseObjects();
        // 直接 free mData 和 mObjects 
        // 申请的是 malloc，释放就是 free 咯
        if (mData) free(mData);
        if (mObjects) free(mObjects);
    }
}

void Parcel::releaseObjects()
{
    const sp<ProcessState> proc(ProcessState::self());
    size_t i = mObjectsSize;
    uint8_t* const data = mData;
    size_t* const objects = mObjects;
    // 这里和 acquireObjects 很像呐
    while (i > 0) {
        i--;
        const flat_binder_object* flat
            = reinterpret_cast<flat_binder_object*>(data+objects[i]);
        release_object(proc, *flat, this);
    }
}

void release_object(const sp<ProcessState>& proc,
    const flat_binder_object& obj, const void* who)
{
    switch (obj.type) {
        case BINDER_TYPE_BINDER:
            if (obj.binder) {
                LOG_REFS("Parcel %p releasing reference on local %p", who, obj.cookie);
                static_cast<IBinder*>(obj.cookie)->decStrong(who);
            }
            return;
        case BINDER_TYPE_WEAK_BINDER:
            if (obj.binder)
                static_cast<RefBase::weakref_type*>(obj.binder)->decWeak(who);
            return;
        case BINDER_TYPE_HANDLE: {
            const sp<IBinder> b = proc->getStrongProxyForHandle(obj.handle);
            if (b != NULL) {
                LOG_REFS("Parcel %p releasing reference on remote %p", who, b.get());
                b->decStrong(who);
            }
            return;
        }
        case BINDER_TYPE_WEAK_HANDLE: {
            const wp<IBinder> b = proc->getWeakProxyForHandle(obj.handle);
            if (b != NULL) b.get_refs()->decWeak(who);
            return;
        }
        case BINDER_TYPE_FD: {
            if (obj.cookie != (void*)0) close(obj.handle);
            return;
        }
    }

    ALOGE("Invalid object type 0x%08lx", obj.type);
}
```

这里的函数都不复杂，我直接贴到底了。其实就是 free 掉之前 malloc 的 mData 和 mObjects，还有就是前面手动增加了的引用计数，这里得再手动减少（这玩意就是麻烦）。

其实传递 binder 对象，最关键的地方其实在 kernel 的 binder 驱动里面，但是鉴于这篇已经够长了，而且这个和 ServiceManager 关系也挺密切的，所以决定把这块地方放到 ServiceManager 那篇去。


Parcel 是 android binder 通信中扮演着数据打包、解包的角色，是比较重要的一个东西。它的内存结构其实很简单，以后自己用的时候要注意下遵守规则（读、写顺序一致）。然后它可以传递一些小数据、也还可以传一些二进制流，对于大型数据提供匿名共享内存（Ashmem）的支持，它还有一个很特殊的功能，就是传递 binder 对象，保证了 binder IPC 通信的正常使用。


