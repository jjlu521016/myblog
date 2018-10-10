---
title: 使用Spring Boot、Multipartfile上传文件路径错误问题
toc: true
date: 2018-07-17 10:52:32
tags:
    - Multipartfile
---
## 报错信息： 
java.io.IOException: java.io.FileNotFoundException: /tmp/tomcat/.../tmp/files/xxx.jpg (No such file or directory)
问题源码： transferTo方法报错
```java
File file = new File("/tmp/files/");
try {
    multipartFile.transferTo(file);
    ...
}
```
<!-- more -->
## 问题分析
源码中文件定义的是相对路径，预期路径应该是项目路径/tmp/source/，但是报错确是一个系统临时文件路径（tomcat的）。
由于是transferTo方法报错，因此应该是该方法写入文件时报错，因此，我们跟入方法源码。
```java
public class StandardMultipartHttpServletRequest extends AbstractMultipartHttpServletRequest {
    private static final String CONTENT_DISPOSITION = "content-disposition";
    private static final String FILENAME_KEY = "filename=";
    private static final String FILENAME_WITH_CHARSET_KEY = "filename*=";
    private static final Charset US_ASCII = Charset.forName("us-ascii");
    private Set<String> multipartParameterNames;
    // ....
    private static class StandardMultipartFile implements MultipartFile, Serializable {
        private final Part part;
        private final String filename;

        // ...
        public void transferTo(File dest) throws IOException, IllegalStateException {
            this.part.write(dest.getPath());
        }
    }
}
```
```java
public class ApplicationPart implements Part {
   // ....

    public void write(String fileName) throws IOException {
        File file = new File(fileName);
        // 问题在这里：如果文件不是绝对路径就重新创建！！
        if (!file.isAbsolute()) {
            file = new File(this.location, fileName);
        }

        try {
            this.fileItem.write(file);
        } catch (Exception var4) {
            throw new IOException(var4);
        }
    }

}

```
使用Servlet3.0的支持的上传文件功能时，如果我们没有使用绝对路径的话，transferTo方法会在相对路径前添加一个location路径，即：file = new File(location, fileName)，由于创建的File在项目路径/tmp/files/，而transferTo方法预期写入的文件路径为/tmp/tomcat/.../tmp/files/xxx.jpg，我们并没有创建该目录，因此会抛出异常。

## 问题解决方案
+ 1 使用绝对路径
+ 2 修改location的值
这个location可以理解为临时文件目录，我们可以通过配置location的值，使其指向我们的项目路径，这样就解决了我们遇到的问题。
在Spring Boot下配置location，可以在main()方法所在文件中添加如下代码：
```java
 @Bean
 MultipartConfigElement multipartConfigElement() {
    MultipartConfigFactory factory = new MultipartConfigFactory();
    //设置路径xxx
    factory.setLocation("/xxx");
    return factory.createMultipartConfig();
}
```