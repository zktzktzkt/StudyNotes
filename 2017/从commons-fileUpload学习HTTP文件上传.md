# 从commons-fileUpload学习HTTP文件上传

如题，由于前一阵子开始学习一些关于Node.js的东西,然后也做一个简单的文件服务器之类的程序来练手,但是关于服务端解析HTTP获取文件上传的部分代码处理不好从网上搜索了一份代码用上了,但是这份代码有个缺陷就是它解析的过程将上传文件的数据全部放在内存中解析,由于我的服务器内存是2G的所以开始想办法优化一下,出于希望优化后的处理方式是非常好的,所以先从commons-fileUpload框架内先学习一下成熟的框架的处理方式,但是此次只是一个较为粗略的分析,结论也比较粗糙(commons-fileUpload采用先缓存临时文件的方式进行处理),对于commons-fileUpload中关于HTTP中的协议的详细处理都没有仔细去看,这一部分希望是抽空系统的看下HTTP的上传部分的协议.


## 1.整体逻辑

由于commons-fileUpload整个处理的逻辑很长,所以首先简单的的梳理一下整体逻辑如下

step 1:

服务器端使用commons-fileUpload框架内的解析代码如下

```
//1、创建一个磁盘文件相关的factory
DiskFileItemFactory factory = new DiskFileItemFactory();
//2、创建一个文件上传解析器
ServletFileUpload upload = new ServletFileUpload(factory);
upload.setHeaderEncoding("UTF-8");
try {
    //此处获取解析到的数据item,即commons-io解析HTTP报文的入口方法
	List<FileItem> list = upload.parseRequest(request);
    //之后根据业务场景对得到的上传文件进行处理
    }
} catch (FileUploadException e) {
	e.printStackTrace();
}
```
step 2:

ServletFileUpload实例的解析方法parseRequest方法首先将new ServletRequestContext(request)包装一层然后调用到如下方法开始解析

```
public List<FileItem> parseRequest(RequestContext ctx) throws FileUploadException {
    List<FileItem> items = new ArrayList<FileItem>();
    boolean successful = false;
    try {
        //根据RFC 1867的规范处理multipart/form-data 数据流并通过迭代器的形式返回得到的数据项
        FileItemIterator iter = getItemIterator(ctx);//调用链尾部调用了new FileItemIteratorImpl(ctx);
        //获取step 1中的设置的DiskFileItemFactory实例
        FileItemFactory fac = getFileItemFactory();
        //...
        while (iter.hasNext()) {//判断是否还有数据item没有解析完毕
            final FileItemStream item = iter.next();
            final String fileName = ((FileItemIteratorImpl.FileItemStreamImpl) item).name;
            //通过factory创建FileItem来管理临时文件,该方法执行完毕之后一个临时文件则生成
            FileItem fileItem = fac.createItem(item.getFieldName(), item.getContentType(), item.isFormField(), fileName);
            items.add(fileItem);
            try {
                //将item中的数据写到fileItem的输出流中,完成临时文件的数据写入,个人认为此处函数名有误导,
                Streams.copy(item.openStream(), fileItem.getOutputStream(), true);
            } catch (FileUploadIOException e) {
                throw (FileUploadException) e.getCause();
            } catch (IOException e) {
                throw new IOFileUploadException(format("Processing of %s request failed. %s", MULTIPART_FORM_DATA, e.getMessage()), e);
            }
            final FileItemHeaders fih = item.getHeaders();
            fileItem.setHeaders(fih);
        }
        successful = true;
        return items;
    }
    //错误处理略....
}
```

由上的逻辑可以对commons-io对于上传的文件的处理方式有一个简单的了解,看到其中copy方法我认为能够隐隐约约感觉到common-io的文件处理方式是,先将解析的一段数据项存到临时文件中封装为一个个的FileItem并返回一个集合,这样用户只需操作写好的临时文件即可.


## 2、二进制流的解析

从getItemIterator(ctx)开始看起，顺着getItemIterator的调用链可以追到该方法最终调用到的有效方法是FileItemIteratorImpl

step 1: 依据rfc1867文档进行处理数据流

```
FileItemIteratorImpl(RequestContext ctx) throws FileUploadException, IOException {
    if (ctx == null) {
        throw new NullPointerException("ctx parameter");
    }
    String contentType = ctx.getContentType();//获取HTTP请求头中的content-type值
    InputStream input = ctx.getInputStream();//获取输入流
    @SuppressWarnings("deprecation") // still has to be backward compatible
    final int contentLengthInt = ctx.getContentLength();
    final long requestSize = UploadContext.class.isAssignableFrom(ctx.getClass()) ? ((UploadContext) ctx).contentLength() : contentLengthInt;
    //此处用于限制上传文件的大小,默认是不进入此处逻辑,sizeMax=-1
    if (sizeMax >= 0) {
        //....
    }
    String charEncoding = headerEncoding;
    if (charEncoding == null) {
        charEncoding = ctx.getCharacterEncoding();
    }
    //获取content-type中的Content-Type:multipart/form-data; boundary=----WebKitFormBoundaryQDPv1LHuBVgxrNc4 此处解析方法见3.1
    boundary = getBoundary(contentType);
    if (boundary == null) {
        throw new FileUploadException("the request was rejected because no multipart boundary was found");
    }
    //用于给监听者提供进度通知,listener默认为null
    notifier = new MultipartStream.ProgressNotifier(listener, requestSize);
    try {
        multi = new MultipartStream(input, boundary, notifier);
    } catch (IllegalArgumentException iae) {
        throw new InvalidContentTypeException(format("The boundary specified in the %s header is too long", CONTENT_TYPE), iae);
    }
    multi.setHeaderEncoding(charEncoding);
    skipPreamble = true;//第一次preamble(不知道这个该如何翻译)
    //HTTP请求头解析完毕,开始进行传输数据body的解析
    findNextItem();
}
```

step 2:数据体的解析,如果找到下一项则返回true

```
//这个方法是数据解析主要逻辑,此处只关注整体的流程和一些流程上的功能的具体实现,而不去细究这里针对HTTP协议的具体处理逻辑.
private boolean findNextItem() throws IOException {
    if (eof) {
        return false;
    }
    //关闭上轮解析的数据实体bean
    if (currentItem != null) {
        currentItem.close();
        currentItem = null;
    }
    for (;;) {
        boolean nextPart;
        //skipPreamble标识是否需要跳过boundary,此处两种方法均为寻找剩余数据流是否还有数据体需要解析
        if (skipPreamble) {
            nextPart = multi.skipPreamble();
        } else {
            nextPart = multi.readBoundary();
        }
        if (!nextPart) {
            if (currentFieldName == null) {
                eof = true;
                return false;
            }
            multi.setBoundary(boundary);
            currentFieldName = null;
            continue;
        }
        FileItemHeaders headers = getParsedHeaders(multi.readHeaders());//读上传数据的实体的描述
        //此处currentFieldName一般为null
        if (currentFieldName == null) {
            String fieldName = getFieldName(headers);
            if (fieldName != null) {
                String subContentType = headers.getHeader(CONTENT_TYPE);
                if (subContentType != null &&  subContentType.toLowerCase(Locale.ENGLISH).startsWith(MULTIPART_MIXED)) {
                    //只在type=multipart/mixed会进入
                    currentFieldName = fieldName;
                    // Multiple files associated with this field name
                    byte[] subBoundary = getBoundary(subContentType);
                    multi.setBoundary(subBoundary);
                    skipPreamble = true;
                    continue;
                }
                //正常逻辑，构造信息
                String fileName = getFileName(headers);
                //构造了一个ItemInputStream,用于读取HTTP数据流为上传文件生成临时文件
                currentItem = new FileItemStreamImpl(fileName, fieldName, headers.getHeader(CONTENT_TYPE), fileName == null, getContentLength(headers));
                currentItem.setHeaders(headers);
                notifier.noteItem();//通知进度
                itemValid = true;
                return true;
            }
        } else {
            //略
        }
        multi.discardBodyData();//丢弃无用数据
    }
}
```

至此虽然还有很多具体的功能实现没有看,但是整个文件上传的流程已经看完了即解析HTTP头->解析数据体->生成临时文件->用户处理,接着进一步看一下具体一点的功能实现,比如分割符boundary的解析、上传文件的文件名解析、数据body的获取、临时文件的创建于管理等.

### 3 具体功能实现

### 3.1 文件分割单元boundary的解析

step 1:创建解析器准备解析

```
//HTTP请求头中的Content-Type内容,例:multipart/form-data; boundary=----WebKitFormBoundary7TWrKdXZZkI9Vp6s
protected byte[] getBoundary(String contentType) {
    ParameterParser parser = new ParameterParser();//一个简单的key=value解析器
    parser.setLowerCaseNames(true);
    ////第一个参数是要解析的内容，第二个则key=value之间的分隔符,返回一个hashmap
    Map<String, String> params = parser.parse(contentType, new char[] {';', ','});
    String boundaryStr = params.get("boundary");
    if (boundaryStr == null) {
        return null;
    }
    byte[] boundary;
    try {
        boundary = boundaryStr.getBytes("ISO-8859-1");
    } catch (UnsupportedEncodingException e) {
        boundary = boundaryStr.getBytes();
    }
    return boundary;
}
```

step 2:确定key-value间的分割符

```
public Map<String, String> parse(final String str, char[] separators) {
    if (separators == null || separators.length == 0) {
        return new HashMap<String, String>();
    }
    char separator = separators[0];
    if (str != null) {
        int idx = str.length();
        //寻找分割符位置,应该针对的是分割符只有一种的情况,否则这里获取到的最终的分割符为最后一个找到的
        for (char separator2 : separators) {
            int tmp = str.indexOf(separator2);
            if (tmp != -1 && tmp < idx) {
                idx = tmp;
                separator = separator2;
            }
        }
    }
    return parse(str, separator);
}
```

step 3：分割字符,返回map

```
public Map<String, String> parse(final char[] charArray, int offset, int length, char separator) {
    if (charArray == null) {
        return new HashMap<String, String>();
    }
    HashMap<String, String> params = new HashMap<String, String>();
    this.chars = charArray;
    this.pos = offset;
    this.len = length;
    String paramName = null;
    String paramValue = null;
    while (hasChar()) {//根据当前解析的数据下标和数组长度判断是否还有数据需要解析
        paramName = parseToken(new char[] {'=', separator });//根据=、,提取key,简单的字符比较获取数据其实下标这里的separator是无用的,pos最终停在=所在的位置
        paramValue = null;
        if (hasChar() && (charArray[pos] == '=')) {
            pos++; // skip '='
            paramValue = parseQuotedToken(new char[] { separator });//可以解析" "引号中的数据
            if (paramValue != null) {
                try {
                    paramValue = MimeUtility.decodeText(paramValue);
                } catch (UnsupportedEncodingException e) {
                }
            }
        }
        if (hasChar() && (charArray[pos] == separator)) {
            pos++;
        }
        if ((paramName != null) && (paramName.length() > 0)) {
            if (this.lowerCaseNames) {
                paramName = paramName.toLowerCase(Locale.ENGLISH);
            }
            params.put(paramName, paramValue);
        }
    }
    return params;
}
```

这里我本以为只会通过简单的字符匹配来解析,但是事实证明虽然解析的部分差不多是通过字符匹配来完成,但是要做的其他的事情要复杂的多,这可能是工业级的代码和个人的代码的区别吧.

### 3.2 字节数据流的解析及FileItem的创建

由于数据流的解析主要由MultipartStream完成，MultipartStream类用于处理符合RFC1867文档中关于MIME type为multipart的数据流,而且能够处理任意大小的数据.这里主要按照上面的findNextItem方法中的逻辑顺序进行方法的分析.

step 1:首先看一下其构造方法,主要它的缓存的大小,以及boundary的处理

```
public MultipartStream(InputStream input, byte[] boundary, int bufSize, ProgressNotifier pNotifier) {
    if (boundary == null) {
        throw new IllegalArgumentException("boundary may not be null");
    }
    this.boundaryLength = boundary.length + BOUNDARY_PREFIX.length;//BOUNDARY_PREFIX=\r\n--
    //bufSize 默认为4096个字节
    if (bufSize < this.boundaryLength + 1) {
        throw new IllegalArgumentException("The buffer size specified for the MultipartStream is too small");
    }
    this.input = input;
    this.bufSize = Math.max(bufSize, boundaryLength * 2);
    this.buffer = new byte[this.bufSize];
    this.notifier = pNotifier;
    this.boundary = new byte[this.boundaryLength];
    this.keepRegion = this.boundary.length;
    System.arraycopy(BOUNDARY_PREFIX, 0, this.boundary, 0, BOUNDARY_PREFIX.length);
    //将boundary设置为\r\n-- + boundary来作为数据体的分割边界
    System.arraycopy(boundary, 0, this.boundary, BOUNDARY_PREFIX.length, boundary.length);
    head = 0;//标志当前处理的字符串在buffer中起始位置
    tail = 0;
}
```

step 2.1:

```
public boolean skipPreamble() throws IOException {
    // First delimiter may be not preceeded with a CRLF.
    System.arraycopy(boundary, 2, boundary, 0, boundary.length - 2);
    boundaryLength = boundary.length - 2;
    try {
        //抛弃无效数据,实际上是抛弃直到下一个boundary之前的数据,discardBody的实现和后面FileItem创建、临时文件生成的逻辑相似此处不具体分析
        discardBodyData();
        return readBoundary();//寻找的新的数据项
    } catch (MalformedStreamException e) {
        return false;
    } finally {
        System.arraycopy(boundary, 0, boundary, 2, boundary.length - 2);
        boundaryLength = boundary.length;
        boundary[0] = CR;
        boundary[1] = LF;
    }
}
```

step 2.2:

```
public boolean readBoundary() throws FileUploadIOException, MalformedStreamException {
    byte[] marker = new byte[2];
    boolean nextChunk = false;
    head += boundaryLength;
    try {
        //这里通过匹配\r\n来判断是否已经开始新的数据项
        marker[0] = readByte();
        if (marker[0] == LF) {
            return true;
        }
        marker[1] = readByte();//如果当前的buffer已经处理完毕则再次读入一个bufSize字节大小的数据到buffer数组中,并返回第一个字节
        if (arrayequals(marker, STREAM_TERMINATOR, 2)) {//STREAM_TERMINATOR等于--
            nextChunk = false;
        } else if (arrayequals(marker, FIELD_SEPARATOR, 2)) {//FIELD_SEPARATOR等于\r\n
            nextChunk = true;
        } else {
            throw new MalformedStreamException("Unexpected characters follow a boundary");
        }
    } catch (FileUploadIOException e) {
        throw e;
    } catch (IOException e) {
        throw new MalformedStreamException("Stream ended unexpectedly");
    }
    return nextChunk;
}
```

step 3:解析header

```
public String readHeaders() throws FileUploadIOException, MalformedStreamException {
    int i = 0;
    byte b;
    // to support multi-byte characters
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    int size = 0;
    //读取数据流,直到发现header的\r\n结尾
    while (i < HEADER_SEPARATOR.length) {//HEADER_SEPARATOR等于\r\n
        try {
            b = readByte();//读取一个字节
        } catch (FileUploadIOException e) {
            throw e;
        } catch (IOException e) {
            throw new MalformedStreamException("Stream ended unexpectedly");
        }
        if (++size > HEADER_PART_SIZE_MAX) {
            throw new MalformedStreamException(format("Header section has more than %s bytes (maybe it is not properly terminated)",
                           Integer.valueOf(HEADER_PART_SIZE_MAX)));
        }
        if (b == HEADER_SEPARATOR[i]) {
            i++;
        } else {
            i = 0;
        }
        baos.write(b);
    }
    String headers = null;
    if (headerEncoding != null) {
        try {
            headers = baos.toString(headerEncoding);
        } catch (UnsupportedEncodingException e) {
            headers = baos.toString();
        }
    } else {
        headers = baos.toString();
    }
    return headers;
}
```

step 4:数据体的解析与FileItem的创建

```
FileItemStreamImpl(String pName, String pFieldName, String pContentType, boolean pFormField, long pContentLength) throws IOException {
    name = pName;
    fieldName = pFieldName;
    contentType = pContentType;
    formField = pFormField;
    final ItemInputStream itemStream = multi.newInputStream();//拿到本次buffer中分割符之前的所有数据
    InputStream istream = itemStream;
    if (fileSizeMax != -1) {
        //略,一般不设置最大文件限制
    }
    stream = istream;
}
```

step 5:FileItem核心-输入流的创建

```
ItemInputStream newInputStream() {
    return new ItemInputStream();
}
ItemInputStream() {
    findSeparator();
}

//在buffer中查找boundary并作出读取限制
private void findSeparator() {
    pos = MultipartStream.this.findSeparator();
    //如果找到boundary,则限制输入流的读取字节数,其中pad为结束下标
    if (pos == -1) {
        if (tail - head > keepRegion) {
            pad = keepRegion;
        } else {
            pad = tail - head;
        }
    }
}

//MultipartStream.this.findSeparator();
protected int findSeparator() {
    int first;
    int match = 0;
    int maxpos = tail - boundaryLength;
    for (first = head; first <= maxpos && match != boundaryLength; first++) {
        first = findByte(boundary[0], first);
        if (first == -1 || first > maxpos) {
            return -1;
        }
        //匹配boundary
        for (match = 1; match < boundaryLength; match++) {
            if (buffer[first + match] != boundary[match]) {
                break;
            }
        }
    }
    if (match == boundaryLength) {
        return first - 1;
    }
    return -1;
}
```

简单的说来这个输入流的核心创建逻辑是从当前处理中buffer中找到boundary出现的位置,这样boundary之前的字节数据都是属于上传文件的一部分

### 3.3 临时文件的数据填充

step 1：数据体的写入到临时文件

此处实现由Streams.copy(item.openStream(), fileItem.getOutputStream(), true);实现,该方法将解析得到的数据写入到临时文件中,开始我第一遍看到这里的时候有一点困惑，因为从前面的方法来看,coommons-io在解析数据时只解析到header就结束了,然后只是在创建FileItem的时候创建了一个输入流而且输入流的数据也只不过一个buffer左右的数据,没有解析获取数据体的过程,后来仔细看了下面这段才发现数据体的读取实在最终生成临时文件时操作的

```
public static long copy(InputStream inputStream, OutputStream outputStream, boolean closeOutputStream, byte[] buffer)throws IOException {
    OutputStream out = outputStream;
    InputStream in = inputStream;
    try {
        long total = 0;
        for (;;) {
            int res = in.read(buffer);//核心方法,从HTTP数据流中读取数据
            if (res == -1) {
                break;
            }
            if (res > 0) {
                total += res;
                if (out != null) {
                    out.write(buffer, 0, res);
                }
            }
        }
        if (out != null) {
            if (closeOutputStream) {
                out.close();
            } else {
                out.flush();
            }
            out = null;
        }
        in.close();
        in = null;
        return total;
    } finally {
        IOUtils.closeQuietly(in);
        if (closeOutputStream) {
            IOUtils.closeQuietly(out);
        }
    }
}
```

step 2:

InputStream的read(buffer)方法是通过read方法来完成的如下

```
public int read() throws IOException {
    if (closed) {
        throw new FileItemStream.ItemSkippedException();
    }
    //available()方法仅仅根据当前buffer数组中是否还有数据可读作为依据,makeAvailable()则会尝试从HTTP的数据流中来扩展新的数据
    if (available() == 0 && makeAvailable() == 0) {
        return -1;
    }
    ++total;
    int b = buffer[head++];//一次只读一个字节
    if (b >= 0) {
        return b;
    }
    return b + BYTE_POSITIVE_OFFSET;
}
```

step 3：扩容

```

private int makeAvailable() throws IOException {
    if (pos != -1) {
        return 0;
    }
    total += tail - head - pad;
    System.arraycopy(buffer, tail - pad, buffer, 0, pad);
    head = 0;
    tail = pad;
    for (;;) {
        //从HTTP的数据里在读出一个buffer的数据
        int bytesRead = input.read(buffer, tail, bufSize - tail);
        if (bytesRead == -1) {
            // The last pad amount is left in the buffer.
            // Boundary can't be in there so signal an error
            // condition.
            final String msg = "Stream ended unexpectedly";
            throw new MalformedStreamException(msg);
        }
        if (notifier != null) {
            notifier.noteBytesRead(bytesRead);
        }
        tail += bytesRead;
        findSeparator();//查找boundary位置
        int av = available();
        //用于判断是否已经读到边界boundary
        if (av > 0 || pos != -1) {
            return av;
        }
    }
}
```

相信上面的3个步骤看下来应该就是解答上面的疑惑,就是虽然FileItem创建时初始化的数据流的数据量只不过是一个buffer左右的数据,但是在最终读取解析body时,如果数据读完,这个数据流会自动的从HTTP数据流中再读出部分数据,知道读取boundary为止.

### 3.4 临时文件的管理

step 1:准备通过factory创建临时文件

```
public FileItem createItem(String fieldName, String contentType, boolean isFormField, String fileName) {
    DiskFileItem result = new DiskFileItem(fieldName, contentType,isFormField, fileName, sizeThreshold, repository);
    FileCleaningTracker tracker = getFileCleaningTracker();
    if (tracker != null) {
        tracker.track(result.getTempFile(), result);
    }
    return result;
}
```

step 2 创建临时文件

```
protected File getTempFile() {
    if (tempFile == null) {
        File tempDir = repository;
        if (tempDir == null) {
            tempDir = new File(System.getProperty("java.io.tmpdir"));
        }
        //创建临时文件
        String tempFileName = format("upload_%s_%s.tmp", UID, getUniqueId());
        tempFile = new File(tempDir, tempFileName);
    }
    return tempFile;
}
```

step 3: 将临时文件添加到跟踪列表中

```
private synchronized void addTracker(final String path, final Object marker, final FileDeleteStrategy deleteStrategy) {
    if (exitWhenFinished) {
        throw new IllegalStateException("No new trackers can be added once exitWhenFinished() is called");
    }
    if (reaper == null) {
        reaper = new Reaper();
        reaper.start();//创建并启动处理临时文件的线程
    }
    //创建文件信息并添加到集合中,注意Tracker集成自PhantomReference是java种最弱的引用虚引用,这里将DiskFileItem实例与q 引用队列建立了联系关系
    trackers.add(new Tracker(path, deleteStrategy, marker, q));//将文件信息添加到几个
}
```

step 4:处理线程检查引用可达性

```
public void run() {
    // thread exits when exitWhenFinished is true and there are no more tracked objects
    while (exitWhenFinished == false || trackers.size() > 0) {
        try {
            final Tracker tracker = (Tracker) q.remove(); // cannot return null
            trackers.remove(tracker);
            if (!tracker.delete()) {
                deleteFailures.add(tracker.getPath());
            }
            tracker.clear();
        } catch (final InterruptedException e) {
            continue;
        }
    }
}
```

上述分析可以知道commons-fileUpload通过利用临时文件实例DiskFileItem创建虚引用并和引用队列建立联系,通过引用队列来判断临时文件是否还需要使用巧妙地完成了临时文件的监控处理


## 4.总结

综上commons-fileUpload的处理逻辑为事先解析所有数据并生成临时文件,然后提供接口给用户使用,当用户准备获取文件时实质上实在读取临时文件而不是从HTTP数据流中读取(但是这个事实多增加了一次文件I/O降低了处理速度),此外临时文件的清理则是通过弱引用的实现.