---
title: Couchbase client 2.x 自定义压缩支持
date: 2018-03-20 21:55:27
tags: 
    - NoSQL
    - Java
---

Couchbase client 2.x存储对象主要以Json格式的Document为主。
为了支持N1QL查询特性，除了LegacyDocument，client内部其它定义的Document均都未支持数据压缩功能。

```Java
/**
 * This document is fully compatible with Java SDK 1.* stored documents.
 *
 * It is not compatible with other SDKs. It should be used to interact with legacy documents and code, but it is
 * recommended to switch to the unifying document types (Json* and String) if possible to guarantee better
 * interoperability in the future.
 *
 * @author Michael Nitschinger
 * @since 2.0
 */
public class LegacyDocument extends AbstractDocument<Object> {
    ...
}

```

如上面Java doc说明，LegacyDocument是兼容了1.x版本的Java SDK。
所以LegacyDocument势必要支持数据压缩功能，它的压缩机制是通过LegacyTranscoder实现的。

LegacyTranscoder中有两个方法：doEncode和doDecode。
顾名思义，doEncode实现了序列化编码功能，而doDecode实现了序列化解码功能。
```Java
public class LegacyTranscoder extends AbstractTranscoder<LegacyDocument, Object> {

    public static final int DEFAULT_COMPRESSION_THRESHOLD = 16384;
    
    ...

    private final int compressionThreshold;
    public LegacyTranscoder(int compressionThreshold) {
            this.compressionThreshold = compressionThreshold;
    }
    ...

    @Override
    protected Tuple2<ByteBuf, Integer> doEncode(LegacyDocument document)
        throws Exception {

        int flags = 0;
        Object content = document.content();

        boolean isJson = false;
        ByteBuf encoded;
        if (content instanceof String) {
            String c = (String) content;
            isJson = isJsonObject(c);
            encoded = TranscoderUtils.encodeStringAsUtf8(c);
        } else {
            encoded = Unpooled.buffer();

            if (content instanceof Long) {
                flags |= SPECIAL_LONG;
                encoded.writeBytes(encodeNum((Long) content, 8));
            } else if (content instanceof Integer) {
                flags |= SPECIAL_INT;
                encoded.writeBytes(encodeNum((Integer) content, 4));
            } else if (content instanceof Boolean) {
                flags |= SPECIAL_BOOLEAN;
                boolean b = (Boolean) content;
                encoded = Unpooled.buffer().writeByte(b ? '1' : '0');
            } else if (content instanceof Date) {
                flags |= SPECIAL_DATE;
                encoded.writeBytes(encodeNum(((Date) content).getTime(), 8));
            } else if (content instanceof Byte) {
                flags |= SPECIAL_BYTE;
                encoded.writeByte((Byte) content);
            } else if (content instanceof Float) {
                flags |= SPECIAL_FLOAT;
                encoded.writeBytes(encodeNum(Float.floatToRawIntBits((Float) content), 4));
            } else if (content instanceof Double) {
                flags |= SPECIAL_DOUBLE;
                encoded.writeBytes(encodeNum(Double.doubleToRawLongBits((Double) content), 8));
            } else if (content instanceof byte[]) {
                flags |= SPECIAL_BYTEARRAY;
                encoded.writeBytes((byte[]) content);
            } else {
                flags |= SERIALIZED;
                encoded.writeBytes(serialize(content));
            }
        }

        if (!isJson && encoded.readableBytes() >= compressionThreshold) {
            byte[] compressed = compress(encoded.copy().array());
            if (compressed.length < encoded.array().length) {
                encoded.clear().writeBytes(compressed);
                flags |= COMPRESSED;
            }
        }

        return Tuple.create(encoded, flags);
    }

    ...
}
```

这里有一个注意点:
doEncode默认是不支持JSON格式的字符串进行压缩的。
如上述代码描述的，若存储内容是一个字符串，它会优先判断是不是JSON格式的字符串，若是，则设置isJson为true，后续流程就跳过了压缩逻辑。

** 因此，若要支持JSON格式的字符串压缩，一种可选的方案是，使用LegacyDocument，重写LegacyTranscoder，覆盖doEncode逻辑，去掉对JSON字符串的判断处理。**

此外，在调用CLUSTER.openBucket方法时，使用类似如下包含transcoders签名参数的方法，将自定义的transcoder传入。
```Java
Bucket openBucket(String name, List<Transcoder<? extends Document, ?>> transcoders);
```

另外一点:
LegacyTranscoder默认设置了压缩阈值16k，即存储内容大小达到16k以后才会压缩。这对有些使用场景来说，阈值设置太大了。
由于compressionThreshold字段是私有的，因此，若需要调整阈值，可选的办法:
1. 继承LegacyTranscoder，在构造方法中重新给compressionThreshold赋值。
2. 如处理压缩逻辑一样，直接继承AbstractTranscoder，重写LegacyTranscoder。

关于非JSON Document的存储，详情可以进一步参考[Couchbase文档-Non-JSON Documents](https://developer.couchbase.com/documentation/server/5.1/sdk/nonjson.html)。
