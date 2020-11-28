# memcpy和memmove的区别

其实很早就知道两个函数其中有一个在面临内存覆盖时行为有点特别, 但是工作中很少用到此场景, 也就没有深究. 现在居然面试遇到了, 那就把研究清楚吧.

- memcpy 简单粗暴, 不考虑内存重叠问题. 后果程序员自负
- memmove 比memcpy多了层检查内存重叠的考虑,如果发现重叠, 则反向拷贝, 性能和memcpy基本一样. 就是多了个检查是否重叠的代码.

综上所述, 以后干脆就用memmove吧. 省的那么多事. 反正性能几乎没有损失.

##### 测试代码：
```c++
int main(int argc, char **args)
{
    char c1[20] =  {0};
    snprintf(c1, sizeof(c1), "hello world");
    printf("original str   : %s",c1);
    printf("\n");

    memcpy(c1+3, c1, 9); 
    printf("memcpy result  : %s",c1);
    printf("\n");
 
    snprintf(c1, sizeof(c1), "hello world");
    memmove(c1+3, c1, 9); 
    printf("memmove result : %s",c1);
    printf("\n");
 
    return 0;
}
```

###### 输出结果如下:
```txt
original str   : hello world
memcpy result  : helhellellol      // 完全乱了
memmove result : helhello wor      // 正确处理了内存重叠问题
```

#### 扩展阅读, memmove是如何实现的?

由于该函数比memcpy安全的一点在于要考虑内存是否重叠,如果重叠则反向copy避免还没有copy的地方被覆盖.
实现要点:
1) 检查是否内存重叠,如果没有重叠,则直接调用memcpy.
2) 如果重叠,则从src+len,到dst+len倒序copy.
   此时不能利用memcpy,因为memcpy是正向copy的. 需要按照类似memcpy的实现,在汇编指令中把cld替换为std,把方向标记位改下即可.
 
内存重叠的含义是什么呢?
1)如果dst < src 则无论dst和src距离多远都没问题.
2)如果dst > src,并且两者的差值>=len, 则虽然dst在后面,因距离很远, 也没有重叠
3)如果dst > src 并且差值<len, 那么copy时则会重叠导致不可预测的结果.
 
========================================================
glibc 2.17 的memmove源码如下
 
```c++
rettype
MEMMOVE (a1, a2, len)
     a1const void *a1;
     a2const void *a2;
     size_t len;
{
  unsigned long int dstp = (long int) dest;
  unsigned long int srcp = (long int) src;
 
  // 根据上面的三种情况, 这里由于dstp,srcp都是无符号的即使dstp<srcp, 负值对应的无符号数也是很大的值.
  /* This test makes the forward copying code be used whenever possible.
     Reduces the working set.  */
  if (dstp - srcp >= len) /* *Unsigned* compare!  */
    {
      /* Copy from the beginning to the end.  */
 
 
#if MEMCPY_OK_FOR_FWD_MEMMOVE
      dest = memcpy (dest, src, len);
#else
      ... 这里略过, 其实就是memcpy的实现
#endif /* MEMCPY_OK_FOR_FWD_MEMMOVE */
    }
  else
    {
      /* Copy from the end to the beginning.  */
      srcp += len;
      dstp += len;
 
 
      /* If there not too few bytes to copy, use word copy.  */
      if (len >= OP_T_THRES)
 {
   /* Copy just a few bytes to make DSTP aligned.  */
   len -= dstp % OPSIZ;
   // 这里利用了反向copy的宏. BWD means backward​
   BYTE_COPY_BWD (dstp, srcp, dstp % OPSIZ);
 
 
   /* Copy from SRCP to DSTP taking advantage of the known
      alignment of DSTP.  Number of bytes remaining is put
      in the third argument, i.e. in LEN.  This number may
      vary from machine to machine.  */
 
 
   WORD_COPY_BWD (dstp, srcp, len, len);
 
 
   /* Fall out and copy the tail.  */
 }
 
 
      /* There are just a few bytes to copy.  Use byte memory operations.  */
      BYTE_COPY_BWD (dstp, srcp, len);
    }
 
 
  RETURN (dest);
}
```