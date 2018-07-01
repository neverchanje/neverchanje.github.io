---
title: MySQL UTF-8 导致性能下降
date: 2017-1-28
type: "post"
comments: false
tags: mysql
---

## 问题发现

有段时间公司云 MySQL 性能降低了很多，我们跟了一下上线记录，发现是 MySQL 的配置在一个月之前有改动（可见每次上线前的改动都要严格地记录下来，并且需要便于查看）。我们回退每个配置项修改，每次回退都做一次性能测试，最终发现是我们把字符集从 latin 改为 utf8，造成的这次性能下降。

## 问题调研

MySQL utf-8 会导致字符串体积增加，会对所有内部 buffer 机制造成影响，例如只有更少的索引更够被放入内存等，所以会导致性能下降。

虽然 utf-8 默认是 4 bytes，但是 MySQL 只会用 3 bytes 存储，这是因为大部分字符不需要 4 bytes。所以理论上：

- 同一个 CHAR 声明的字符串用 latin 存储和用 utf-8 存储空间上会差 3 倍
- 对于 VARCHAR 声明的 utf8 存储，相对会节省空间，如果只需要 1 byte 便可表示的字符，那就不会用 3 bytes（是类似 protobuf 的 varint 机制？）
- 但是不管是对于 CHAR 和 VARCHAR，在许多任务上还是按照最大可能使用内存来分配的。在存储时，VARCHAR(255) 确实会优于 CHAR(255)，前者理论上会花费更少空间。但是在其他地方（比如排序）时，MySQL 依然会直接分配 255 bytes 的空间给 VARCHAR。所以特别是在使用 utf-8 时，需要更加注意 VARCHAR 的使用。

所以将 MySQL 的默认编码改成 utf-8 后造成性能下降应该是正常现象。

## 额外的调研

- MySQL 会将字符长度分为 CHAR_LENGTH() 和 LENGTH()，对于 utf8 的字符串，后者是前者的 3 倍。

- MySQL 限制索引列不能超过 999 bytes，相当于不能有超过 333 个 utf8 字符。

- 字符集的意义在于进行字符串比较（e.g. LIKE），排序，字符串操作（e.g. SUBSTRING），而如果完全不需要这些操作，可以直接使用 binary，对于复杂情况，可以多加一列，表示字符集。

## 参考

- https://adayinthelifeof.nl//2010/12/04/about-using-utf-8-fields-in-mysql/
- High Performance MySQL, p298 ch7 — Character Sets and Collations

