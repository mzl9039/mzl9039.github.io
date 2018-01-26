---
layout:     post
title:      "Spark Utils两个函数的体会"
subtitle:   ""
date:       2018-1-25 23:00:00
author:     mzl
catalog:    true
tags:
    - java.io
    - File
    - Canonical
    - Absolute
---

{:toc}
# 在看Spark DiskBlockManager的时候，涉及了Utils里的两个方法 chmod700 和 createDirectory，由于以前对 java 不太熟悉，故写一下体会
## chmod700
```scala
  /**
    * JDK equivalent of `chmod 700 file`.
    * 利用 JDK 的功能给文件赋权限为 700，即仅当前用户具有：读写执行权限，其它用户没有任何权限
    * @param file the file whose permissions will be modified
    * @return true if the permissions were successfully changed, false otherwise.
    */
  def chmod700(file: File): Boolean = {
    // java.io 包的 File 类的方法 setReadable/setWritable/setExecutable 三个方法各有同名方法，以 setReadable 为例说明：
    // 1. public boolean setReadable(boolean readable)
    // 2. public boolean setReadable(boolean readable, boolean owneronly)
    // readable 为 true, 即设置为具有读权限；为 false, 则设置为不具有读权限
    // owneronly 为 true, 即权限只适用于所有者；为 false, 则权限适用于所有用户。但当文件系统不能区分时所有者读权限，读权限将适用于所有用户
    // 返回值：true 表示赋权限成功；否则赋权限失败。
    // 异常：SecurityException: 当安全管理器存在且其securityManager.checkwrite 方法执行对文件进行写访问时抛出
    file.setReadable(false, false) &&
    file.setReadable(true, true) &&
    file.setWritable(false, false) &&
    file.setWritable(true, true) &&
    file.setExecutable(false, false) &&
    file.setExecutable(true, true)
  }
```

## createDirectory
```scala
  /**
    * Create a directory inside the given parent directory. The directory is guaranteed to be
    * newly created, and is not marked for automatic deletion.
    * 这个方法特别的地方在于：创建目录的时候，添加了重试机制，应该是防止硬盘无响应的情况下进行，容错性更好一些。
    * 另外，注意如果要创建的目录 dir 已经存在（可能性很低，因为使用了UUID）, 则不再创建目录，多次重试都存在，则返回
    * 目录的标准路径
    * 这里需要注意 File 类的 Canonical 与 Absolute 这两个词的区别：
    * Canonical 用解析定义 File 时用的 . 或 .. ，解析为正常的路径返回；
    * Absolute 则不解析定义 File 时的 . 或 .. ，而是带着 . 或 .. 返回。
    */
  def createDirectory(root: String, namePrefix: String = "spark"): File = {
    var attempts = 0
    val maxAttempts = MAX_DIR_CREATION_ATTEMPTS
    var dir: File = null
    while (dir == null) {
      attempts += 1
      if (attempts > maxAttempts) {
        throw new IOException("Failed to create a temp directory (under " + root + ") after " +
          maxAttempts + " attempts!")
      }
      try {
        dir = new File(root, namePrefix + "-" + UUID.randomUUID().toString)
        if (dir.exists() || !dir.mkdirs()) {
          dir = null
        }
      } catch { case e: SecurityException => dir = null; }
    }

    dir.getCanonicalFile
  }
```
