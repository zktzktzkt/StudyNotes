# Node.js File System模块API介绍

## 1.File System 模块概述
 
Node js的文件I/O 是基于标准的POSIX系统的API的一个封装,所有的方法都提供了异步和同步API.其中异步的API都带有一个callback方法回调,且回调方法的第一个参数是关于error的信息,
如果没有error信息,则第一个参数的值为null或者undefined.

## 2.文件访问常用常量说明

**2.1文件访问常量**

文件访问常量主要用于函数fs.access()测试进程对于文件访问权使用

+ F_OK:表示当前文件对于当前进程可见,可用于测试文件是否存在
+ R_OK:表示进程对文件有读权限
+ W_OK：表示进程对于文件有写权限
+ X_OK:表示进程可执行当前文件

**2.2文件打开部分常量**

文件打开常量主要用于函数fs.open系列用于指示文件打开模式

+ 'r':读文件,如果不存在则失败

'r+':读写文件，不存在则失败 

'rs+':以同步模式读写文件,同时指示操作系统绕开本地缓存,对于性能有较大影响.

'w':写文件,不存在则创建，否则覆盖

'wx':类似于w，但是如果路径存在则失败

'w+':类似于w,读写文件

'wx+':类似于w+但是如果路径存在则失败

'a':追加形式打开文件,如果不存在则创建

'ax':类似于a,但是存在则失败

'a+':读、追加模式打开,不存在则自动创建

'ax+':类似于a+,但是文件存在则失败

## 3.FS模块API模块介绍

### 3.1读文件API

+ fs.createReadStream(path[, options])

创建文件流,以流的形式读取,第一个参数可以是文件路径也可以一个buffer,options是json格式的参数,是可选的.示例如下

```
let stream=fs.createReadStream("fs.txt",{flags:'r',encoding:"utf-8",start:10,end:23000});
stream.on("data",function(chunk){//如果设置了encoding，此处chunk为一个字符串,否则为buffer
    console.log(`Received ${chunk.length} bytes of data.`);
    console.log(chunk);
}); 
```

+ fs.open(path, flags[, mode], callback)
+ fs.read(fd, buffer, offset, length, position, callback)

前者API用于打开文件获取文件fd,参数分别表示文件路径、打开模式、文件模式(默认可读可写),callback包括两个参数error、fd，后者则可以使用该fd进行文件的读取，从position位置读取length
长度的字节数数据,后者的callback参数包括(err, bytesRead, buffer),示例如下

```
fs.open("path.txt",'r',function(err,fd){
    var readBuffer = new Buffer(1024);
    //读取从0开始的1024个字节
    fs.read(fd,readBuffer,0,readBuffer.length,0,(err,bytesRead,readBuffer)=>{
        console.log(readBuffer.slice(0, bytesRead));
    });
    }
})
```

此外对应的同步API为fs.openSync(path, flags[, mode]),fs.readSync(fd, buffer, offset, length, position)
+ fs.readFile(file[, options], callback)

读取文件内容,第一个参数可以是路径、buffer、文件fd,可选的options包括编码以及读取模式,callback包含两个参数(err, data),此外如果encoding没有指定则返回的是原始buffer.示例如下

```
fs.readFile("."+path,"utf-8",function(err,data){
        if(err) {
            res.end("error occur, error is "+err);
        }
        else{
            res.end(data);
        }
    });
```

对应的同步API-fs.readFileSync(file[, options])

### 3.2 写文件API

+ fs.createWriteStream(path[, options])

类似于createReadStream,该方法的第一个参数可以是文件路径String也可以一个buffer,options是json格式的参数,是可选的.使用方法如下

```
const fs = require("fs");
let write = fs.createWriteStream("D:\\out.txt",{flags:'w',encoding:'utf-8',autoClose:true});
write.write('some data');
write.write('some more data');
write.end('done writing data');
```

+ fs.write(fd, buffer, offset, length[, position], callback)

第一个参数为文件描述符、第二个参数为要写入到文件的String 或者 Buffer,第三个参数表示写入位置,第四个参数表示写入数据字节长度,postion表示写入数据的偏移或者说起始位置,callback的参数为(err, written, buffer),其中written表示写入多少数据到buffer

```
fs.open("out.txt",'w',function(err,fd){
    fs.write(fd,"I am write data", 0, 0, function(err,written,buffer){
        console.log("written="+written+"buffer="+buffer);
    })
})
```

测试表示,length不起作用,最终会写入所有数据.同步API为fs.writeSync(fd, buffer, offset, length[, position])

+ fs.write(fd, data[, position[, encoding]], callback)

类似于上一个API,第三个参数position表示写入位置,第四个参数表示写入的编码,callback的参数为(err, written, string),第二个参数表示写入的字节数.用法同上,同步API为fs.writeSync(fd, data[, position[, encoding]])

+ fs.writeFile(file, data[, options], callback)

第一个参数为文件路径、Buffer、文件描述符,第二个参数为String或者Buffer,第三个参数是写入文件的配置json文本,第四个参数callback只有一个参数err

```
fs.writeFile("out.txt","Test Write File",{encoding:'utf-8',flag:'a+'},(err)=>{})
```

相应的同步APIfs.writeFileSync(file, data[, options])

### 3.3目录相关API

+ fs.readdir(path[, options], callback)

用于读取指定目录下的文件列表,options用于指定编码,callback参数为(err, files),相应的同步API为fs.readdirSync(path[, options])

### 3.4文件属性相关API

+ fs.access(path[, mode], callback)

用于测试文件权限,其中fs.constants.F_OK可用于检测文件是否存在

```
fs.access('/etc/passwd', fs.constants.R_OK | fs.constants.W_OK, (err) => {
  console.log(err ? 'no access!' : 'can read/write');
});
```

+ fs.stat(path, callback)

callback参数为(err, stats),stats的类型为fs.Stats对象.可以用获取文件创建时间、是否为目录等等一系列属性