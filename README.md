## 一、主要内容
- 演示如何将blob 对象上传到服务器端。主要应用场景是在无法选择本地文件，无法使用上传控件，将内存或网络上的数据直接上传到服务器上情况。


>Blob 对象表示一个不可变、原始数据的类文件对象。它的数据可以按文本或二进制的格式进行读取，也可以转换成 ReadableStream 来用于数据操作。 
Blob 表示的不一定是JavaScript原生格式的数据。File 接口基于Blob，继承了 blob 的功能并将其扩展使其支持用户系统上的文件。

关于更多 Blob 的介绍可参考：https://developer.mozilla.org/zh-CN/docs/Web/API/Blob

## 二、关于演示程序的引用

需要引用 CryptoJS v3.1.2  的md5方法计算文件的hashcode,用于唯一标识需要上传的大文件。

```
<script src="md5.js"></script>
```

## 三、为什么例子中采用文件上传开始那？

例子是用来演示Blob的上传，而不是文件上传，但是为了方便起见，将文件对象(File)转换为Blob。文件转化的Blob和其他途径来的Blob是没有什么区别

```
    const maxChunkSize = 10000000;
    const endpoint = "http://127.0.0.1:8080/FileUpload/Upload";
    const input = document.querySelector('input[type="file"]');

    input.addEventListener('change', function (e) {
        
        let file = input.files[0];
        let filename = file.name;
        
        let reader = new FileReader();
        reader.onload = function(e) {
            let blob = new Blob([new Uint8Array(e.target.result)], {type: file.type });
            console.log(blob);

            if (blob.size <= maxChunkSize)
                uploadWhole(endpoint, blob,filename);
            else
                uploadBig(endpoint, blob, maxChunkSize,filename);
        };
        reader.readAsArrayBuffer(file);
        
    }, false);
```

上面的代码借用了FileReader 将 File对象读取，进而转换成 Blob

## 四、小文件的上传比较简单

```
function uploadWhole(url, blob,filename) {
    const formData = new FormData();
    var file = new File([blob], filename);
    formData.append("files", file);
    fetch(endpoint, {
        method: "post",
        body: formData
    }).then((response) => {

        return response.json();
    }).then((json) => {
        
        console.log("小文件上传成功",json);
    });
}
```
在blob上传前转换为File，主要是增加文件的名字，后台需要文件名字。
采用原生的fetch方法上传，可以采用Promise的形式接收返回结果，代码上更加简练。



## 五、大文件上传需要三步走

```
function uploadBig(url, blob, maxsize,filename) {
    // 1、切片
    let chunks = sliceFile(blob, maxChunkSize);
    console.log(chunks);
    // 2、计算文件 hashcode 
    blob.arrayBuffer().then(buffer => {
        var wordArray = CryptoJS.lib.WordArray.create(buffer);
        var md5 = CryptoJS.MD5(wordArray).toString()
        return md5;
    }).then(fileid => {
        //计算出文件唯一ID，再启动切片上传
        console.log("fileid", fileid);
        let chunkSize = chunks.length;
        // 3、开始分别上传切片
        uploadPart(url, chunks, chunkSize,fileid, filename);
    });
}
```
1、切片
2、计算文件 hashcode
3、开始分别上传切片

## 六、分片上传

```
function uploadPart(url, chunks,chunkSize,fileid,filename) {
    let part = chunks.shift();
    if (part) {
        const formData = new FormData();
        let blob = part.blob;
        
        var filePart = new File([blob], filename);
        
        let range = part.range;

        formData.append("files", filePart);
        fetch(url, {
            headers: {
                'X-File-Identifier': fileid,
                "Content-Range": range
            },
            method: "post",
            body: formData
        }).then((response) => {
            return response.json();
        }).then((json) => {
            //console.log(json);

            let ratio = (chunkSize - chunks.length) / chunkSize * 100;
            console.log("已完成",ratio);

            if (chunks.length > 0) {
                setTimeout(uploadPart(url, chunks,chunkSize,fileid, filename), 500);
            } else {
                return json;
            }
        }).then((result) => {
            if (result) {
                //文件上传成功
                console.log(result);
            }

        });
    }
}
```
1、上传切片时需要在头信息中携带文件 hashcode，和 当前切片的序列信息

```
Content-Range: bytes 0-9999999/18296832
X-File-Identifier: 58bc266b18fc1c8bd240fc674c4325d0
```

2、每次上传完都可以重新计算上传的进度,如果没有上传完则继续调用上传

```
  let ratio = (chunkSize - chunks.length) / chunkSize * 100;
            console.log("已完成",ratio);
if (chunks.length > 0) {
    setTimeout(uploadPart(url, chunks,chunkSize,fileid, filename), 500);
} else {
    return json;
}
```
3、最有一个切片上传成功后提示成功

```
if (result) {
    //文件上传成功
    console.log(result);
}
```
