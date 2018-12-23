---
layout: post
title: CSAPP:malloclab（一）
category: CSAPP_LAB
description: CSAPP:malloclab
published: true
---

## malloclab介绍
malloclab顾名思义就是要求自己实现c语言中的malloc函数，此外还有free和realloc函数
这个实验是csapp系列实验目前做到现在觉得最棘手的了=_=|||
可能是因为这部分上课没有认真听，参考了几个博客之后，终于理解了，先尝试完成一个最简单的隐式空闲链表+首次适配+原始realloc的版本，这个版本书上基本都给出了完整的代码，我们只需要实现find_fit()和place()即可

## 隐式空闲链表+首次适配+原始realloc
首先，先在mm.h中把需要的宏定义和函数声明加上

```c
#define WSIZE 4
#define DSIZE 8
#define CHUNKSIZE (1<<12)

#define MAX(x,y) ((x)>(y)?(x):(y))

#define PACK(size,alloc) ((size)|(alloc))

#define GET(p)     (*(unsigned int *)(p))
#define PUT(p,val) (*(unsigned int *)(p) = (val))

#define GET_SIZE(p)  (GET(p) & ~0x7)
#define GET_ALLOC(p) (GET(p) & 0x1)

#define HDRP(bp) ((char *)(bp) - WSIZE)
#define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)

#define NEXT_BLKP(bp) ((char *)(bp) + GET_SIZE(((char *)(bp) - WSIZE)))
#define PREV_BLKP(bp) ((char *)(bp) - GET_SIZE(((char *)(bp) - DSIZE)))

extern void *extend_heap(size_t words);
extern void *coalesce(void *bp);
extern void *find_fit(size_t size);
extern void place(void *bp,size_t asize);
extern int mm_init (void);
extern void *mm_malloc (size_t size);
extern void mm_free (void *ptr);
extern void *mm_realloc(void *ptr, size_t size);
```

然后是mm.c里面各个函数的实现

不要忘记新建一个指向块首的指针，在遍历整个堆寻找合适块的时候要用到
```c
static char *heap_listp = 0;
```

### mm_init()
![image.png-95.8kB][1]
以堆底为起始位置，新建四个WSIZE（4bytes or 32bits）大小的块
第一块什么都不放作为填充字
第二、三块分别作为序言块的头和脚
第四块作为结尾块
然后将指针后移两块，指向序言块脚部起始位置
随后调用extend_heap()，申请CHUNKSIZE大小的空间，以备用

```c
int mm_init(void)
{
    if((heap_listp = mem_sbrk(4*WSIZE))==(void *)-1){
        return -1;
    }
    PUT(heap_listp,0);
    PUT(heap_listp+(1*WSIZE),PACK(DSIZE,1));
    PUT(heap_listp+(2*WSIZE),PACK(DSIZE,1));
    PUT(heap_listp+(3*WSIZE),PACK(0,1));
    heap_listp += (2*WSIZE);
    if(extend_heap(CHUNKSIZE/WSIZE)==NULL){
        return -1;
    }
    return 0;
}
```

### extend_heap()
此函数将堆扩容指定byte大小
如果指定words大小不为8的倍数则向上取整使得每次扩容都是八字节对齐
最后将头、脚内容补齐
并将下一个块置为结尾块
最后调用coalesce()函数堆bp进行合并操作后返回

```c
static void *extend_heap(size_t words){
    char *bp;
    size_t size;

    size = (words%2) ? (words+1)*WSIZE : words*WSIZE;
    if((long)(bp=mem_sbrk(size))==(void *)-1)
        return NULL;

    PUT(HDRP(bp),PACK(size,0));
    PUT(FTRP(bp),PACK(size,0));
    PUT(HDRP(NEXT_BLKP(bp)),PACK(0,1));

    return coalesce(bp);
}
```

### mm_malloc()
此函数申请大小为size的空间
asize为对size进行8字节对齐检查后的大小
extendsize为取CHUNKSIZE和asize中较大者
先用find_fit()在现有的块中进行搜索，如果搜索到了，即用place()函数将asize大小的空间放到bp中（有可能全部分也可能分割成一个占用块和一个空闲块，所以不能粗暴的完全占用，要写一个place()函数来分配）
如果找不到合适块，就向堆申请新空间，新空间的大小为extendsize，然后再用place()函数放入

```c
void *mm_malloc(size_t size)
{
    size_t asize;
    size_t extendsize;
    char *bp;
    if(size ==0) return NULL;

    if(size <= DSIZE){
        asize = 2*(DSIZE);
    }else{
        asize = (DSIZE)*((size+(DSIZE)+(DSIZE-1)) / (DSIZE));
    }
    if((bp = find_fit(asize))!= NULL){
        place(bp,asize);
        return bp;
    }
    extendsize = MAX(asize,CHUNKSIZE);
    if((bp = extend_heap(extendsize/WSIZE))==NULL){
        return NULL;
    }
    place(bp,asize);
    return bp;
}
```

### mm_free()
此函数释放指针指向的占用块
很简单，将块的头尾块的alloc信息改为0
然后进行合并

```c
void mm_free(void *bp)
{
    if(bp == 0)
	return;

    size_t size = GET_SIZE(HDRP(bp));

    PUT(HDRP(bp), PACK(size, 0));
    PUT(FTRP(bp), PACK(size, 0));
    coalesce(bp);
}
```

### coalesce（）
此函数将对传入的块指针进行前后检查
当然，传进来的是空闲块
如果前面块或者后面块同样为空闲块，就进行合并
说起来很简单，代码还是比较长，需要认真看看
首先获取前后块的空闲状态，然后进行条件判断
四种情况
1.前后都忙碌。2.前忙碌，后空闲。3.前空闲，后忙碌。4.前后都空闲
![image.png-214.9kB][2]

```c
static void *coalesce(void *bp){
    size_t  prev_alloc = GET_ALLOC(FTRP(PREV_BLKP(bp)));
    size_t  next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp)));
    size_t size = GET_SIZE(HDRP(bp));

    if(prev_alloc && next_alloc) {
        return bp;
    }else if(prev_alloc && !next_alloc){
        	size += GET_SIZE(HDRP(NEXT_BLKP(bp)));
            PUT(HDRP(bp), PACK(size,0));
            PUT(FTRP(bp), PACK(size,0));
    }else if(!prev_alloc && next_alloc){
        size += GET_SIZE(HDRP(PREV_BLKP(bp)));
        PUT(FTRP(bp),PACK(size,0));
        PUT(HDRP(PREV_BLKP(bp)),PACK(size,0));
        bp = PREV_BLKP(bp);
    }else {
        size +=GET_SIZE(FTRP(NEXT_BLKP(bp)))+ GET_SIZE(HDRP(PREV_BLKP(bp)));
        PUT(FTRP(NEXT_BLKP(bp)),PACK(size,0));
        PUT(HDRP(PREV_BLKP(bp)),PACK(size,0));
        bp = PREV_BLKP(bp);
    }
    return bp;
}
```

### find_fit（）
此函数从头至尾遍历堆，找到合适大小的块则返回指向该块的指针（即头部块末尾）
找不到则返回NULL

```c
static void *find_fit(size_t size){
    void *bp;
    for(bp = heap_listp; GET_SIZE(HDRP(bp))>0; bp = NEXT_BLKP(bp)){
        if(!GET_ALLOC(HDRP(bp)) && (size <= GET_SIZE(HDRP(bp)))){
            return bp;
        }
    }
    return NULL;
}
```

### place()
此函数将经过8字节对齐大小的空间放到指定的块中
如果分割后原块剩余大小大于2*DSIZE即16bytes
则进行分块
否则将整个块都占用

```c
static void place(void *bp,size_t asize){
    size_t csize = GET_SIZE(HDRP(bp));
    if((csize-asize)>=(2*DSIZE)){
        PUT(HDRP(bp),PACK(asize,1));
        PUT(FTRP(bp),PACK(asize,1));
        bp = NEXT_BLKP(bp);
        PUT(HDRP(bp),PACK(csize-asize,0));
        PUT(FTRP(bp),PACK(csize-asize,0));
    }else{
        PUT(HDRP(bp),PACK(csize,1));
        PUT(FTRP(bp),PACK(csize,1));
    }
}
```

### mm_realloc()
此函数返回指向一个大小为size的区域指针，满足以下条件：
if ptr is NULL, the call is equivalent to mm_malloc(size); 
if size is equal to zero, the call is equivalent to mm_free(ptr); 
if ptr is not NULL：先按照size指定的大小分配空间，将原有数据从头到尾拷贝到新分配的内存区域，而后释放原来ptr所指内存区域


```c
void *mm_realloc(void *ptr, size_t size)
{
    size_t oldsize;
    void *newptr;

    /* If size == 0 then this is just free, and we return NULL. */
    if(size == 0) {
	mm_free(ptr);
	return 0;
    }

    /* If oldptr is NULL, then this is just malloc. */
    if(ptr == NULL) {
	return mm_malloc(size);
    }

    newptr = mm_malloc(size);

    /* If realloc() fails the original block is left untouched  */
    if(!newptr) {
	return 0;
    }

    /* Copy the old data. */
    oldsize = GET_SIZE(HDRP(ptr));
    if(size < oldsize) oldsize = size;
    memcpy(newptr, ptr, oldsize);

    /* Free the old block. */
    mm_free(ptr);

    return newptr;
}
```

至此，一个简单的隐式空闲链表+首次适配+原始realloc版本的malloclab就完成了

  [1]: http://static.zybuluo.com/windmelon/r6ozbceh4znts6pmzy9chncy/image.png
  [2]: http://static.zybuluo.com/windmelon/1de32aexe4mvbt8ljqzeybce/image.png