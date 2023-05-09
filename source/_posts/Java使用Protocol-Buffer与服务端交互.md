---
title: Java使用Protocol Buffer与服务端交互
tags: [Java]
date: 2023-05-09 14:14:26
keywords: Protocol Buffer, .proto
---

最近和三方对接时，对方给出的接口文档是使用protol buffer进行交互的，并非是我们常见的json、xml这种格式，了解了一下这种格式或者说交协议的特点。
首先，Protocol Buffer序列化之后是二进制流，不进行反序列化基本不可读。
其次，序列化之后的体积很小，适合网络传输或者设备之间传输
最后，可以跨平台、跨语言使用
不过这些特点既是优点也是缺点：序列化之后的数据不可读，还原序列化之后的数据需要事先定义好的数据格式

<!--more-->

#### 安装Protocol Buffer的编译器

我们需要使用相应的编译器将`.proto`文件转化为对应的编程语言的代码。
编译器可以在这里下载[https://github.com/protocolbuffers/protobuf](https://github.com/protocolbuffers/protobuf)
这里我下载的版本是22.3。下载完成后解压、添加环境变量，命令行执行 `protoc --version`能够输出版本号就可以了

#### 编写 .proto文件

文件内容及格式可以参考这里[https://protobuf.dev/](https://protobuf.dev/)
下面是一个示例

``` protocol
syntax = "proto2";

package tutorial;

option java_multiple_files = true;
option java_package = "com.example.tutorial.protos";
option java_outer_classname = "AddressBookProtos";

message Person {
  optional string name = 1;
  optional int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    optional string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }

  repeated PhoneNumber phones = 4;
}

message AddressBook {
  repeated Person people = 1;
}

```
然后我们需要使用上面安装好的编译工具将文件编译转化为对应编程语言的文件，这里使用的是java
``` shell

 protocol --java_out=src/main/java src/main/protobuf/AddressBookProtos.proto

```
`src/main/java`是输出文件的位置，`src/main/protobuf/tgssp.proto`是数据格式文件的位置

没有报错的话，我们就可以在输出文件的位置看到生成的java文件了

#### 如何使用

想要使用该文件，我们需要在工程中引入相应的依赖库，这里还是用java举例
``` groovy
implementation group: 'com.google.protobuf', name: 'protobuf-java', version: '3.22.3'
```

因为上面的`.proto`文件中定义的`java_multiple_files`为true，所以这里是分开生成的文件。
然后我们就可以使用了
``` java
        Person person = Person.newBuilder()
                .setEmail("123@123.com")
                .setId(1)
                .setName("null")
                .build();
        AddressBook addressBook = AddressBook.newBuilder()
                .addPeople(person)
                .build();
        System.out.println(addressBook);
```
当然我们也可以将`addressBook`对象调用`toByteArray()`方法序列化为二进制数据流;也可以调用`AddressBook.parseFrom(byte[] bytes)`从二进制数据中反序列化

#### 与服务器交互

这里为了方便，直接使用的apache的网络请求库，使用其他库原理是一样的
依赖
``` groovy
    implementation group: 'org.apache.httpcomponents', name: 'httpcore', version: '4.4.14'
    implementation group: 'org.apache.httpcomponents', name: 'httpclient', version: '4.5.13'
```
代码

``` java
HttpPost request = new HttpPost("https://a.b.com");
request.setEntity(new ByteArrayEntity(tgrequest.toByteArray()));
CloseableHttpClient client = HttpClients.createDefault();
CloseableHttpResponse response = client.execute(request);
// 处理 HTTP 响应
HttpEntity entity = response.getEntity();
if (entity != null) {
    // 将响应实体转换为字节数组
    byte[] data = toByteArray(entity.getContent());
    AddressBook addressBook = AddressBook.parseFrom(data);
    System.out.println(addressBook);
}

//读取响应
private static byte[] toByteArray(InputStream in) throws Exception {
    ByteArrayOutputStream out = new ByteArrayOutputStream();
    byte[] buffer = new byte[4096];
    int len;
    while ((len = in.read(buffer)) != -1) {
        out.write(buffer, 0, len);
    }
    return out.toByteArray();
}

```

到这里就算是完成了一次使用protocol buffer的交互

#### 其他方式

我们可以使用`protostuff`这个库，从而不借助`.proto`文件就可以直接对POJO进行序列化和反序列化。
详情可以查看这个仓库 [https://github.com/protostuff/protostuff](https://github.com/protostuff/protostuff)

