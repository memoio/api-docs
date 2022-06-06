# 一、 启动节点和网关



如果是在windows下可以直接使用windows安装包完成安装，运行start图标以后程序会自动启动节点以及网关。并在节点的根目录下的account.txt文件中记录网关的账号和密码，默认账号为节点钱包地址，默认密码为memoriae，默认网关api端口为5080



备注：

如果想手动部署启动节点，在memo的官方网站中给出了多个部署启动节点的方法文档可供使用。

如果想在命令行下手动启动网关，则使用以下命令启动gateway，将使用默认参数：

mefs-user gateway run 

也可以在启动时可以手动指定账号，密码，endpoint等参数，启动参数说明：

```
C:\memouser\batch>mefs-user.exe gateway run -h
NAME:
   mefs-user gateway run - run a memo gateway

USAGE:
   mefs-user gateway run [command options] [arguments...]

OPTIONS:
   --username value, -n value  input your user name (default: "memo")
   --password value, -p value  input your password (default: "memoriae")
   --endpoint value, -e value  input your endpoint (default: "0.0.0.0:5080")
   --console value, -c value   input your console for browser (default: "8080")
   --help, -h                  show help (default: false)
```



# 二、 网关支持的S3兼容接口列表

MakeBucket

ListBuckets

BucketExists

RemoveBucket

GetObject

FGetObject

PutObject

FPutObject

ListObjects

RemoveObject

StatObject

# 三、 minio go SDK说明

## 构造函数

New(endpoint, accessKeyID, secretAccessKey string, ssl bool) (*Client, error)

构造函数用于初始化minio client对象

| 参数              | 类型     | 描述                       |
| :---------------- | :------- | :------------------------- |
| `endpoint`        | *string* | S3兼容对象存储服务endpoint |
| `accessKeyID`     | *string* | 对象存储的Access key       |
| `secretAccessKey` | *string* | 对象存储的Secret key       |
| `ssl`             | *bool*   | true代表使用HTTPS          |

初始化minio client对象的示例，memo网关默认不适用ssl

```go
package main

import (
    "fmt"

    "github.com/minio/minio-go/v6"
)

func main() {
        // 不使用ssl
        ssl := false

        // 初使化minio client对象。
        minioClient, err := minio.New("127.0.0.1:5080", "0x9c784fD443Faf95D25F1598F7D2ece581CeA2DEe", "memoriae", ssl)
        if err != nil {
                fmt.Println(err)
                return
        }
}
```

## 操作桶的接口


### 创建一个桶

MakeBucket(bucketName, location string) error

| 参数         | 类型     | 描述                                                         |
| :----------- | :------- | :----------------------------------------------------------- |
| `bucketName` | *string* | 存储桶名称                                                   |
| `location`   | *string* | 存储桶被创建的region(地区)，默认是us-east-1(美国东一区)，在 memo中具体使用时可以直接用默认值。 |

示例

```go
err = minioClient.MakeBucket("mybucket", "us-east-1")
if err != nil {
    fmt.Println(err)
    return
}
fmt.Println("Successfully created mybucket.")
```

### 列出user的所有桶

ListBuckets() ([]BucketInfo, error)

| 返回值       | 类型                 | 描述               |
| :----------- | :------------------- | :----------------- |
| `bucketList` | *[]minio.BucketInfo* | 所有存储桶的list。 |

示例

```go
buckets, err := minioClient.ListBuckets()
if err != nil {
    fmt.Println(err)
    return
}
for _, bucket := range buckets {
    fmt.Println(bucket)
}
```

### 检查存储桶是否存在

BucketExists(bucketName string) (found bool, err error)

| 参数         | 类型     | 描述       |
| :----------- | :------- | :--------- |
| `bucketName` | *string* | 存储桶名称 |

示例

```go
found, err := minioClient.BucketExists("mybucket")
if err != nil {
    fmt.Println(err)
    return
}
if found {
    fmt.Println("Bucket found")
}
```

### 删除一个存储桶

RemoveBucket(bucketName string) error

unimplement

### 列举存储桶里的对象

ListObjects(bucketName, prefix string, recursive bool, doneCh chan struct{}) <-chan ObjectInfo

| 参数           | 类型            | 描述                                                         |
| :------------- | :-------------- | :----------------------------------------------------------- |
| `bucketName`   | *string*        | 存储桶名称                                                   |
| `objectPrefix` | *string*        | 要列举的对象前缀                                             |
| `recursive`    | *bool*          | `true`代表递归查找，`false`代表类似文件夹查找，以'/'分隔，不查子文件夹。 |
| `doneCh`       | *chan struct{}* | 在该channel上结束ListObjects iterator的一个message。         |

示例

```go

// Create a done channel to control 'ListObjects' go routine.
doneCh := make(chan struct{})

// Indicate to our routine to exit cleanly upon return.
defer close(doneCh)

isRecursive := true
objectCh := minioClient.ListObjects("mybucket", "myprefix", isRecursive, doneCh)
for object := range objectCh {
    if object.Err != nil {
        fmt.Println(object.Err)
        return
    }
    fmt.Println(object)
}
```

## 操作对象的接口



### 获取对象

GetObject(bucketName, objectName string, opts GetObjectOptions) (*Object, error)

返回对象数据的流，error是读流时经常抛的那些错。

**参数**

| 参数         | 类型                     | 描述                                          |
| :----------- | :----------------------- | :-------------------------------------------- |
| `bucketName` | *string*                 | 存储桶名称                                    |
| `objectName` | *string*                 | 对象的名称                                    |
| `opts`       | *minio.GetObjectOptions* | GET请求的一些额外参数，像encryption，If-Match |



### 获取对象到文件

FGetObject(bucketName, objectName, filePath string, opts GetObjectOptions) error

下载并将文件保存到本地文件系统

| 参数         | 类型                     | 描述                                          |
| :----------- | :----------------------- | :-------------------------------------------- |
| `bucketName` | *string*                 | 存储桶名称                                    |
| `objectName` | *string*                 | 对象的名称                                    |
| `filePath`   | *string*                 | 下载后保存的路径                              |
| `opts`       | *minio.GetObjectOptions* | GET请求的一些额外参数，像encryption，If-Match |

示例

```go
err = minioClient.FGetObject("mybucket", "myobject", "/tmp/myobject", minio.GetObjectOptions{})
if err != nil {
    fmt.Println(err)
    return
}
```



### 上传对象

PutObject(bucketName, objectName string, reader io.Reader, objectSize int64,opts PutObjectOptions) (n int, err error)

当对象小于128MiB时，直接在一次PUT请求里进行上传。当大于128MiB时，根据文件的实际大小，PutObject会自动地将对象进行拆分成128MiB一块或更大一些进行上传。对象的最大大小是5TB。

| 参数         | 类型                     | 描述                                                         |
| :----------- | :----------------------- | :----------------------------------------------------------- |
| `bucketName` | *string*                 | 存储桶名称                                                   |
| `objectName` | *string*                 | 对象的名称                                                   |
| `reader`     | *io.Reader*              | 任意实现了io.Reader的GO类型                                  |
| `objectSize` | *int64*                  | 上传的对象的大小，-1代表未知。                               |
| `opts`       | *minio.PutObjectOptions* | 允许用户设置可选的自定义元数据，内容标题，加密密钥和用于分段上传操作的线程数量。 |

示例

```go
file, err := os.Open("my-testfile")
if err != nil {
    fmt.Println(err)
    return
}
defer file.Close()

fileStat, err := file.Stat()
if err != nil {
    fmt.Println(err)
    return
}

n, err := minioClient.PutObject("mybucket", "myobject", file, fileStat.Size(), minio.PutObjectOptions{ContentType:"application/octet-stream"})
if err != nil {
    fmt.Println(err)
    return
}
fmt.Println("Successfully uploaded bytes: ", n)
```



### 上传对象文件

FPutObject(bucketName, objectName, filePath, opts PutObjectOptions) (length int64, err error)

将filePath对应的文件内容上传到一个对象中。

当对象小于128MiB时，FPutObject直接在一次PUT请求里进行上传。当大于128MiB时，根据文件的实际大小，FPutObject会自动地将对象进行拆分成128MiB一块或更大一些进行上传。对象的最大大小是5TB。

| 参数         | 类型                     | 描述                                                         |
| :----------- | :----------------------- | :----------------------------------------------------------- |
| `bucketName` | *string*                 | 存储桶名称                                                   |
| `objectName` | *string*                 | 对象的名称                                                   |
| `filePath`   | *string*                 | 要上传的文件的路径                                           |
| `opts`       | *minio.PutObjectOptions* | 允许用户设置可选的自定义元数据，content-type，content-encoding，content-disposition以及cache-control headers，传递加密模块以加密对象，并可选地设置multipart put操作的线程数量。 |

示例

```go
n, err := minioClient.FPutObject("my-bucketname", "my-objectname", "my-filename.csv", minio.PutObjectOptions{
    ContentType: "application/csv",
});
if err != nil {
    fmt.Println(err)
    return
}
fmt.Println("Successfully uploaded bytes: ", n)
```

### 删除一个对象

RemoveObject(bucketName, objectName string) error

unimplement

### 获取对象的元数据

StatObject(bucketName, objectName string, opts StatObjectOptions) (ObjectInfo, error)

unimplement



### 示例代码

此代码演示了创建bucket，上传文件，显示bucket列表的功能。

```go
package main

import (
	"fmt"
	"log"

	"github.com/minio/minio-go/v6"
)

func main() {
	fmt.Println("connecting gateway..")
	endpoint := "127.0.0.1:5080"
	accessKeyID := "0x9c784fD443Faf95D25F1598F7D2ece581CeA2DEe"
	secretAccessKey := "memoriae"
	//useSSL := true

	fmt.Println("initial minio client..")
	// 初使化minio client对象。
	minioClient, err := minio.New(endpoint, accessKeyID, secretAccessKey, false)
	if err != nil {
		log.Fatalln(err)
	}

	//  创建bucket
	bucketName := "mymusic4"

	fmt.Println("creating bucket..")
	location := "us-east-1"
	err = minioClient.MakeBucket(bucketName, location)
	if err != nil {
		exists, err := minioClient.BucketExists(bucketName)
		if err == nil && exists {
			log.Printf("We already own %s\n", bucketName)
			return
		} else {
			log.Fatalln(err)
		}
	}
	log.Printf("Successfully created %s\n", bucketName)

	// 上传一个文件。
	objectName := "Recovery.txt"
	filePath := "c:/Recovery.txt"

	fmt.Println("put object..")
	n, err := minioClient.FPutObject(bucketName, objectName, filePath, minio.PutObjectOptions{})
	if err != nil {
		log.Fatalln(err)
	}

	log.Printf("Successfully uploaded %s of size %d\n", objectName, n)

	// 显示bucket列表
	fmt.Println("list buckets..")
	buckets, err := minioClient.ListBuckets()
	if err != nil {
		fmt.Println(err)
		return
	}
	for _, bucket := range buckets {
		fmt.Println(bucket)
	}
}

```

执行输出为：

```
C:\Users\wy\Desktop\exe>test.exe
connecting gateway..
initial minio client..
creating bucket..
2022/05/31 18:25:12 Successfully created mymusic5
sleep 30 for bucket confirm
put object..
2022/05/31 18:25:47 Successfully uploaded Recovery.txt of size 0
list buckets..
{test1 2022-05-30 08:53:15 +0000 UTC}
{mymusic 2022-05-30 09:54:31 +0000 UTC}
...
{mymusic5 2022-05-31 10:25:12 +0000 UTC}
```

#  四、 minio js SDK说明

## 准备工作

下载依赖包：

npm install --save minio  

## 构造函数

new Minio.Client ({endPoint, port, useSSL, accessKey, secretKey})

初使化一个新的client对象。

| 参数        | 类型     | 描述                                                         |
| :---------- | :------- | :----------------------------------------------------------- |
| `endPoint`  | *string* | endPoint是一个主机名或者IP地址。                             |
| `port`      | *number* | TCP/IP端口号。可选，默认值是，如果是http,则默认80端口，如果是https,则默认是443端口。 |
| `accessKey` | *string* | accessKey类似于用户ID，用于唯一标识你的账户。                |
| `secretKey` | *string* | secretKey是你账户的密码。                                    |
| `useSSL`    | *bool*   | 如果是true，则用的是https而不是http,默认值是true。 

示例

```go
var Minio = require('minio')

var minioClient = new Minio.Client({
    endPoint: '127.0.0.1',
    port: 5080,
    useSSL: false,
    accessKey: '0x9c784fD443Faf95D25F1598F7D2ece581CeA2DEe',
    secretKey: 'memoriae'
});
```

## 操作桶的接口

### 创建一个新的存储桶

makeBucket(bucketName, region[, callback])

| 参数            | 类型       | 描述                                                         |
| :-------------- | :--------- | :----------------------------------------------------------- |
| `bucketName`    | *string*   | 存储桶名称。                                                 |
| `region`        | *string*   | 存储桶被创建的region(地区)，默认是us-east-1(美国东一区)，访问网关时直接使用默认值即可 |
| `callback(err)` | *function* | 回调函数，`err`做为错误信息参数。如果创建存储桶成功则`err`为null。如果没有传callback的话，则返回一个`Promise`对象。 |

示例

```go
minioClient.makeBucket('mybucket', 'us-east-1', function(err) {
  if (err) return console.log('Error creating bucket.', err)
  console.log('Bucket created successfully in "us-east-1".')
})
```

### 列出所有存储桶

listBuckets([callback])

| 参数                          | 类型       | 描述                                                         |
| :---------------------------- | :--------- | :----------------------------------------------------------- |
| `callback(err, bucketStream)` | *function* | 回调函数，第一个参数是错误信息。bucketStream是带有存储桶信息的流。如果没有传callback的话，则返回一个Promise对象。 |



示例

```go
minioClient.listBuckets(function(err, buckets) {
  if (err) return console.log(err)
  console.log('buckets :', buckets)
})
```

### 删除存储桶

removeBucket(bucketName[, callback])

unimplement

### 列出存储桶中所有对象

listObjects(bucketName, prefix, recursive)

| 参数         | 类型     | 描述                                                         |
| :----------- | :------- | :----------------------------------------------------------- |
| `bucketName` | *string* | 存储桶名称。                                                 |
| `prefix`     | *string* | 要列出的对象的前缀 (可选，默认值是`''`)。                    |
| `recursive`  | *bool*   | `true`代表递归查找，`false`代表类似文件夹查找，以'/'分隔，不查子文件夹。（可选，默认值是`false`） |

示例

```go
var stream = minioClient.listObjects('mybucket','', true)
stream.on('data', function(obj) { console.log(obj) } )
stream.on('error', function(err) { console.log(err) } )
```

## 操作对象的接口

### 下载对象

getObject(bucketName, objectName[, callback])

| 参数                    | 类型       | 描述                                                         |
| :---------------------- | :--------- | :----------------------------------------------------------- |
| `bucketName`            | *string*   | 存储桶名称。                                                 |
| `objectName`            | *string*   | 对象名称。                                                   |
| `callback(err, stream)` | *function* | 回调函数，第一个参数是错误信息。`stream`是对象的内容。如果没有传callback的话，则返回一个`Promise`对象。 |



示例

```go
var size = 0
minioClient.getObject('mybucket', 'photo.jpg', function(err, dataStream) {
  if (err) {
    return console.log(err)
  }
  dataStream.on('data', function(chunk) {
    size += chunk.length
  })
  dataStream.on('end', function() {
    console.log('End. Total size = ' + size)
  })
  dataStream.on('error', function(err) {
    console.log(err)
  })
})
```



### 下载对象到文件

fGetObject(bucketName, objectName, filePath[, callback])

下载并将对象保存成本地文件

| 参数            | 类型       | 描述                                                         |
| :-------------- | :--------- | :----------------------------------------------------------- |
| `bucketName`    | *string*   | 存储桶名称。                                                 |
| `objectName`    | *string*   | 对象名称。                                                   |
| `filePath`      | *string*   | 要写入的本地文件路径。                                       |
| `callback(err)` | *function* | 如果报错的话，则会调用回调函数，传入`err`参数。 如果没有传callback的话，则返回一个`Promise`对象。 |

示例

```go
var size = 0
minioClient.fGetObject('mybucket', 'photo.jpg', '/tmp/photo.jpg', function(err) {
  if (err) {
    return console.log(err)
  }
  console.log('success')
})
```



### 上传对象

putObject(bucketName, objectName, stream, size, contentType[, callback])

从一个stream/Buffer中上传一个对象。

| 参数                  | 类型       | 描述                                                         |
| :-------------------- | :--------- | :----------------------------------------------------------- |
| `bucketName`          | *string*   | 存储桶名称。                                                 |
| `objectName`          | *string*   | 对象名称。                                                   |
| `stream`              | *Stream*   | 可以读的流。                                                 |
| `size`                | *number*   | 对象的大小（可选）。                                         |
| `contentType`         | *string*   | 对象的Content-Type（可选，默认是`application/octet-stream`）。 |
| `callback(err, etag)` | *function* | 如果`err`不是null则代表有错误，`etag` _string_是上传的对象的etag值。如果没有传callback的话，则返回一个`Promise`对象。 |

示例

```go
var Fs = require('fs')
var file = '/tmp/40mbfile'
var fileStream = Fs.createReadStream(file)
var fileStat = Fs.stat(file, function(err, stats) {
  if (err) {
    return console.log(err)
  }
  minioClient.putObject('mybucket', '40mbfile', fileStream, stats.size, function(err, etag) {
    return console.log(err, etag) // err should be null
  })
})
```



### 上传文件对象

fPutObject(bucketName, objectName, filePath, contentType[, callback])

| 参数                  | 类型       | 描述                                                         |
| :-------------------- | :--------- | :----------------------------------------------------------- |
| `bucketName`          | *string*   | 存储桶名称。                                                 |
| `objectName`          | *string*   | 对象名称。                                                   |
| `filePath`            | *string*   | 要上传的文件路径。                                           |
| `contentType`         | *string*   | 对象的Content-Type。                                         |
| `callback(err, etag)` | *function* | 如果`err`不是null则代表有错误，`etag` _string_是上传的对象的etag值。如果没有传callback的话，则返回一个`Promise`对象。 |

示例

```go
var file = '/tmp/40mbfile'
minioClient.fPutObject('mybucket', '40mbfile', file, 'application/octet-stream', function(err, etag) {
  return console.log(err, etag) // err should be null
})
```

### 查看对象信息

unimplement



### 示例代码

本范例使用js先创建一个bucket，然后上传一个文件

```js
function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

async function demo() {
	var Minio = require('minio')
	
    console.log("Instantiate the minio client")
	var minioClient = new Minio.Client({
        endPoint: '127.0.0.1',
        port: 5080,
        useSSL: false,
        accessKey: '0x9c784fD443Faf95D25F1598F7D2ece581CeA2DEe',
        secretKey: 'memoriae'
	});


	// File that needs to be uploaded.
	var file = 'c:/Recovery.txt'

	console.log("create bucket.")

	var bucketName='bucketjs13'
	minioClient.makeBucket(bucketName, 'us-east-1', async function(err) {
		if (err) return console.log(err)
		console.log('Bucket created successfully in "us-east-1".')

		var metaData = {
			'Content-Type': 'application/octet-stream',
			'X-Amz-Meta-Testing': 1234,
			'example': 5678
		}
		
		console.log("wait 30 sec for bucket confirm")
		await sleep(30000);
		
		console.log("put object")
		minioClient.fPutObject(bucketName, 'obj1', file, metaData, function(err, etag) {
		  if (err) return console.log(err)
		  console.log('File uploaded successfully.')
		});
	});
}

demo();
```

输出

```
C:\Users\wy\Desktop\testminio>node test.js
Instantiate the minio client
create bucket.
Bucket created successfully in "us-east-1".
wait 30 sec for bucket confirm
put object
File uploaded successfully.
```


# 五、 minio官方文档链接

minio go SDK官方文档

http://docs.minio.org.cn/docs/master/golang-client-api-reference

minio js SDK官方文档

http://docs.minio.org.cn/docs/master/javascript-client-api-reference

