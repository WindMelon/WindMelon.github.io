---
published: true
---
## 1001. A+B Format (20)
	
 Calculate a + b and output the sum in standard format -- that is, the digits must be separated into groups of three by commas (unless there are less than four digits).

**Input**

Each input file contains one test case. Each case contains a pair of integers a and b where -1000000 <= a, b <= 1000000. The numbers are separated by a space.

**Output**

For each test case, you should output the sum of a and b in one line. The sum must be written in the standard format.

**Sample Input**

-1000000 9

**Sample Output**

-999,991

这题的意思就是让你输入两个数字，然后以每千位以逗号分隔的形式输出，这道题我用了两种做法，如下

第一种比较简单暴力，因为题目给定了a，b的上下限，所以可以使用这个上下限写个if语句进行每千位的筛选就行了
```c
#include <stdio.h>

//print a integer in stantard format 1,000,000 
void fun(int a){
  if(a == 0 ){
    printf("%d",a);
  }else
  if(a > 0){
    if(a < 1000){
      printf("%d",a);
    }else
    if(a < 1000000){
      printf("%d,%03d",a/1000,a%1000);
    }else
      printf("%d,%03d,%03d",a/1000000,(a/1000)%1000,a%1000);
  }else
  if(a < 0){
    if(a > -1000){
      printf("%d",a);
    }else
    if(a > -1000000){
      printf("%d,%03d",a/1000,-a%1000);
    }else
      printf("%d,%03d,%03d",a/1000000,(-a/1000)%1000,-a%1000);
  }
}

int main(){
  int a,b,c;
  scanf("%d %d",&a,&b);
  c = a+b;
  fun(c);
  printf("\n");
  return 0;
}
```