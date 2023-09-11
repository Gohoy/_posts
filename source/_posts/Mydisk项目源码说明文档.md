---
title: Mydisk项目说明
category: 设计文档
tag: 
 - 设计文档
 - Mydisk
---

# Mydisk项目说明

## 前台方法（controller）

### minioController：文件的上传、下载、获取等

* 上传文件：uploadFile: public ResultType uploadFile(@RequestParam String fileName, @RequestParam("file") MultipartFile file)
  
  * fileName 文件名称
  
  * MultipartFile file 文件本身
  
  * 思路：根据文件的分类存在不同的bucket和不同路径下。 

* 获取全部文件：getFiles：public ResultType getFiles(@PathVariable(value = "prefix", required = false) String prefix)
  
  * prefix：前缀，可以使用这个来获取某一文件夹下的文件，但是目前这个功能是由前端直接实现的。
  
  * 思路：提供给前端所有存在的文件，支持根据prefix来分类获取。且使用 redis进行缓存。
  
  * getBucketFiles(MinioClient minioClient, String bucketName, String prefix)
    
    * 这是工具函数，用于把文件名进行递归拼接，可以直接返回给前端构建好的列表

* 获取预览/下载链接：public ResultType getFileUrl(@PathVariable("fileName")String fileName)
  
  * fileName：文件名称，需要包含其路径。
  
  * 思路：在后端返回一个可以直接被get的链接，前端模拟点击，可以被预览的文件，如图片视频将在浏览器上直接预览，其他文件将下载。

### TypeController：文件分类的管理

