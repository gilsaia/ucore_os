同理需要添加部分 proc.c...哦对不起这个是后面练习
# 练习一
添加的内容大概是这个意思
```c
if (offset % SFS_BLKSIZE != 0 || endpos / SFS_BLKSIZE == offset / SFS_BLKSIZE) { // 判断被需要读/写的区域所覆盖的数据块中的第一块是否是完全被覆盖的，如果不是，则需要调用非整块数据块进行读或写的函数来完成相应操作
        blkoff = offset % SFS_BLKSIZE; // 计算出在第一块数据块中进行读或写操作的偏移量
        size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset); // 计算出在第一块数据块中进行读或写操作需要的数据长度
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) goto out; // 获取当前这个数据块对应到的磁盘上的数据块的编号
        if ((ret = sfs_buf_op(sfs, buf, size, ino, blkoff)) != 0) goto out; // 将数据读/写到磁盘中 函数在上面指定
        alen += size; // 维护已经读写成功的数据长度信息
        buf += size;
}
uint32_t my_nblks = nblks;
if (offset % SFS_BLKSIZE != 0 && my_nblks > 0) my_nblks --;
if (my_nblks > 0) { // 判断是否存在被需要读写的区域完全覆盖的数据块
        if ((ret = sfs_bmap_load_nolock(sfs, sin, (offset % SFS_BLKSIZE == 0) ? blkno: blkno + 1, &ino)) != 0) goto out; // 如果存在，首先获取这些数据块对应到磁盘上的数据块的编号
        if ((ret = sfs_block_op(sfs, buf, ino, my_nblks)) != 0) goto out; // 将这些磁盘上的这些数据块进行读或写操作
        size = SFS_BLKSIZE * my_nblks;
        alen += size; // 维护已经成功读写的数据长度
        buf += size; // 维护缓冲区的偏移量
}
if (endpos % SFS_BLKSIZE != 0 && endpos / SFS_BLKSIZE != offset / SFS_BLKSIZE) { // 判断需要读写的最后一个数据块是否被完全覆盖（这里还需要确保这个数据块不是第一块数据块，因为第一块数据块已经操作过了）
        size = endpos % SFS_BLKSIZE; // 确定在这数据块中需要读写的长度
        if ((ret = sfs_bmap_load_nolock(sfs, sin, endpos / SFS_BLKSIZE, &ino) == 0) != 0) goto out; // 获取该数据块对应到磁盘上的数据块的编号
        if ((ret = sfs_buf_op(sfs, buf, size, ino, 0)) != 0) goto out; // 进行非整块的读或者写操作
        alen += size;
        buf += size;
}
```
实现管道机制是将一个进程的stdout和另一个进程的stdin链接起来，在sfs里其实有对应的文件类型，实际上只需要一部分的缓冲区而后做好两边进程对应文件结构的指向即可，为了效率可以直接是主存中的缓冲
# 练习二
alloc_proc 多初始化一个
do_fork 多拷一个
load_icode 说清楚要参考之前的写啊...吓我一跳 打开文件以及关闭文件拷贝文件的部分肯定是要改的 然后要考虑的一段就是加载参数的一段，找了参考来写的...

硬链接类似与文件别名，有相同的inode号并且引用计数，不能对目录创建硬链接，实际上..和.就是自带的硬链接，为了防止出现环

软链接是一个文件，只是文件是指向另一个文件的，有自己的inode号等等，没有引用计数，导致并不能实时发现链接文件删除的问题

创建时候分别设置分好即可。