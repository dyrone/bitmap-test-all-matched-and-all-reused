# Overview

we build a repo which have 3 objects : 1 commit 1 blob 1 tree.

Then, we pack them into packfile and create corresponding IDX and bitmap.

We run a git-pack-objects command to observe the behaviour of indexing with bitmap.

# Preparation

[all matched, all reused]

* 3 objects
* 1 pack (1 idx)
* 1 bitmap
* 0 loose objects

# `static void get_object_list(struct rev_info *revs, int ac, const char **av)`


// wants 为 t(111),  haves 为 0, 他们存储 压缩前的数据格式

```
(gdb) print/t bitmap_git.result.word_alloc 
$25 = 1

(gdb) print/t bitmap_git.result.words[0]
$24 = 111
```

// 打印他们的rlw 值

```
(gdb) x /g  bitmap_git.commits.rlw
0x100904ae0:    0000000000000000000000000000001000000000000000000000000000000000
(gdb) x /g  bitmap_git.trees.rlw 
0x100904be0:    0000000000000000000000000000001000000000000000000000000000000000
(gdb) x /g  bitmap_git.blobs.rlw 
0x100904ce0:    0000000000000000000000000000001000000000000000000000000000000000
(gdb) x /g  bitmap_git.tags.rlw 
0x100904de0:    0000000000000000000000000000000000000000000000000000000000000000
```

# In `reuse_partial_packfile_from_bitmap()`

全部的对象都在reuse bimap中， result中已经不存在任何want objects

```
(gdb) print bitmap_git->result.words[0]
$8 = 0
(gdb) print bitmap_git->result.word_alloc 
$9 = 1
```

# In `write_pack_file()`

```
#0  show_objects_for_type (bitmap_git=0x100904370, object_type=OBJ_COMMIT, 
    show_reach=0x1000bdcc0 <add_object_entry_from_bitmap>) at pack-bitmap.c:1395
#1  0x000000010027403d in traverse_bitmap_commit_list (bitmap_git=0x100904370, revs=0x7ffeefbfda20, 
    show_reachable=0x1000bdcc0 <add_object_entry_from_bitmap>) at pack-bitmap.c:2011
```

所以这里 traverse_bitmap_commit_list 遍历不到任何值， 遍历result， 遍历个寂寞

nr_remaining  = nr_result = 3: 待打包的对象数 : 代表要打包的对象的数量， 
result的命名不准确，因为result中可能已经 没有值了（比如当前的测试数据下，
所有的对象都在reuse中），所以确切的说应该叫nr_to_pack.

nr_result 有两个地方累加， 一个是 create_object_entry API中， 另一个是reuse计算后的对象，

to_pack: Objects we are going to pack are collected in the `to_pack` structure. 在
create_object_entry API中， 会将对象添加到 to_pack中。 在本case下， 因为所有的要打包的
对象都在reuse中，所以不会走到create_object_entry API中。故to_pack中没有任何对象。


write_order 为空, 因为to_pack中没有值， to_pack的值
是通过traverse过程中的show_* API写入



```
➜  pack git:(master) git verify-pack --verbose pack-bbd2e3d4ca6e892f66c32418a00dd15a96f4c037.idx 
5bdcd2a157df7615f2cf9dbb4bee06a6ca9398ac commit 249 135 12
b4de3947675361a7770d29b8982c407b0ec6b2a0 blob   3 12 147
03ded7f5353cffe560d27afb1107e71e1faa8f73 tree   33 44 159
non delta: 3 objects
pack-bbd2e3d4ca6e892f66c32418a00dd15a96f4c037.pack: ok
```

// 调试命令

```
sudo gdb --args git  pack-objects --revs --use-bitmap-index --stdout
```


// wants 为 t(111),  haves 为 0, 他们存储 压缩前的数据格式

(gdb) print/t bitmap_git.result.word_alloc 
$25 = 1

(gdb) print/t bitmap_git.result.words[0]
$24 = 111

// 打印他们的rlw 值
(gdb) x /g  bitmap_git.commits.rlw 
0x100904ae0:    0000000000000000000000000000001000000000000000000000000000000000
(gdb) x /g  bitmap_git.trees.rlw 
0x100904be0:    0000000000000000000000000000001000000000000000000000000000000000
(gdb) x /g  bitmap_git.blobs.rlw 
0x100904ce0:    0000000000000000000000000000001000000000000000000000000000000000
(gdb) x /g  bitmap_git.tags.rlw 
0x100904de0:    0000000000000000000000000000000000000000000000000000000000000000



全部的对象都在reuse bimap中， result中已经不存在任何want objects

(gdb) print bitmap_git->result.words[0]
$8 = 0
(gdb) print bitmap_git->result.word_alloc 
$9 = 1


#0  show_objects_for_type (bitmap_git=0x100904370, object_type=OBJ_COMMIT, 
    show_reach=0x1000bdcc0 <add_object_entry_from_bitmap>) at pack-bitmap.c:1395
#1  0x000000010027403d in traverse_bitmap_commit_list (bitmap_git=0x100904370, revs=0x7ffeefbfda20, 
    show_reachable=0x1000bdcc0 <add_object_entry_from_bitmap>) at pack-bitmap.c:2011


所以这里 traverse_bitmap_commit_list 遍历不到任何值， 遍历result， 遍历个寂寞


nr_remaining  = nr_result = 3: 待打包的对象数 : 代表要打包的对象的数量， 
result的命名不准确，因为result中可能已经 没有值了（比如当前的测试数据下，
所有的对象都在reuse中），所以确切的说应该叫nr_to_pack.

nr_result 有两个地方累加， 一个是 create_object_entry API中， 另一个是reuse计算后的对象，


to_pack: Objects we are going to pack are collected in the `to_pack` structure. 在
create_object_entry API中， 会将对象添加到 to_pack中。 在本case下， 因为所有的要打包的
对象都在reuse中，所以不会走到create_object_entry API中。故to_pack中没有任何对象。


write_order 为空, 因为to_pack中没有值， to_pack的值
是通过traverse过程中的show_* API写入的





