---
published: false
---
##1002. A+B for Polynomials (25)

A family hierarchy is usually presented by a pedigree tree. Your job is to count those family members who have no child.
Input

Each input file contains one test case. Each case starts with a line containing 0 < N < 100, the number of nodes in a tree, and M (< N), the number of non-leaf nodes. Then M lines follow, each in the format:

ID K ID[1] ID[2] ... ID[K]
where ID is a two-digit number representing a given non-leaf node, K is the number of its children, followed by a sequence of two-digit ID's of its children. For the sake of simplicity, let us fix the root ID to be 01.
Output

For each test case, you are supposed to count those family members who have no child for every seniority level starting from the root. The numbers must be printed in a line, separated by a space, and there must be no extra space at the end of each line.

The sample case represents a tree with only 2 nodes, where 01 is the root and 02 is its only child. Hence on the root 01 level, there is 0 leaf node; and on the next level, there is 1 leaf node. Then we should output "0 1" in a line.

**Sample Input**

2 1

01 1 02

**Sample Output**

0 1

题意简单，就是简单的多项式加法，这里用的是暴力求解，开辟一个给定上限的数组，在输入过程中就完成加和，最后输出
```c
#include <stdio.h>

int main(){
    float a[1001]={0},coef;
    int K,exp;
    scanf("%d",&K);
    for(int i=0;i<K;i++){
        scanf("%d %f",&exp,&coef);
        a[exp] += coef;
    }
    scanf("%d",&K);
    for(int i=0;i<K;i++){
        scanf("%d %f",&exp,&coef);
        a[exp] += coef;
    }
    K = 0;
    for(int i=0;i<1001;i++){
        if(a[i]!=0){
            K++;
        }
    }
    printf("%d",K);
    for(int i=1000;i>=0;i--){
        if(a[i]!=0){
            printf(" %d %.1f",i,a[i]);
        }
    }
}
```