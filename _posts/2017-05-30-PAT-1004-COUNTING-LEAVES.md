---
published: false
---
## 1004. Counting Leaves (30)
A family hierarchy is usually presented by a pedigree tree. Your job is to count those family members who have no child.

**Input**

Each input file contains one test case. Each case starts with a line containing 0 < N < 100, the number of nodes in a tree, and M (< N), the number of non-leaf nodes. Then M lines follow, each in the format:

ID K ID[1] ID[2] ... ID[K]
where ID is a two-digit number representing a given non-leaf node, K is the number of its children, followed by a sequence of two-digit ID's of its children. For the sake of simplicity, let us fix the root ID to be 01.

**Output**

For each test case, you are supposed to count those family members who have no child for every seniority level starting from the root. The numbers must be printed in a line, separated by a space, and there must be no extra space at the end of each line.

The sample case represents a tree with only 2 nodes, where 01 is the root and 02 is its only child. Hence on the root 01 level, there is 0 leaf node; and on the next level, there is 1 leaf node. Then we should output "0 1" in a line.

**Sample Input**

2 1

01 1 02

**Sample Output**

0 1

这道题意思是给定树的结点数和非叶子结点和其子结点的关系，输出每一层的叶子结点树。需要注意的一个坑就是输入样例并不是按每一层的顺序来输入的，导致count层数的时候需要第一步先记下父节点，后再计算层数
```c
#include <stdio.h>

int main(){
    int a[101]={0},b[101]={-1},c[101]={0},d[101]={0};
    int N,M;
    scanf("%d %d",&N,&M);
    for(int i=0;i<M;i++){
        int father,sc,son;
        scanf("%d %d",&father,&sc);
        a[father-1] = 1; //非叶子结点赋值为1
        for(int j=0;j<sc;j++){
            scanf("%d",&son);
            //b[son-1] = (b[father-1]+1); 
            d[son-1] = (father-1);   //不能用计算层数,因为m行并不一定按顺序给出;改为标示父节点是谁
        }
    }
    for(int i=0;i<N;i++){
        b[i] = b[d[i]] + 1; //在这里计算level
    }
    for(int i=0;i<N;i++){
        if(a[i]==0){
            c[b[i]] ++;
        }
    }
    int k = 0;
    for(int i=0;i<N;i++){
        if(k<b[i]){
            k=b[i];
        }
    }
    k++;
    printf("%d",c[0]);
    for(int i=1;i<k;i++){
        printf(" %d",c[i]);
    }
    return 0;
}
```