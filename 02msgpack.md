## msgpack

**业务需要：**

将一个数据塞进队列由另一端接收，这时要考虑到数据的大小。

因为队列的效率和稳定性正相关。

### 使用

#### msgpack

可以方便地将大部分数据格式转换，python独有的数据格式无法转换。

#### msgpack.packb	序列化

可以将数据转换为 bytes类型的数据。

####  msgpack.unpackb	反序列化

可以将转换后的bytes数据转换为Python使用的数据格式 