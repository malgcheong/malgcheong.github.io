---
title: C프로그램에서의 공통된 메모리 관련 버그
date: 2024-04-17 11:24:40
categories: [C]
tags: [SW 사관학교 정글]
---

9.11 C프로그램에서의 공통된 메모리 관련 버그 참고

### 1. 잘못된 포인터의 역참조
```c
scanf("%d", val);
```
scanf는 **val의 값을 주소로 해석**하고 **이 위치에 워드를 쓰려고** 한다.
최상의 경우에 프로그램은 즉시 예외를 발생시키고 종료한다. 최악의 경우는 val의 값이 유효한 가상메모리의 읽기/쓰기 영역에 대응되어 메모리를 덮어쓰고, 대개 치명적이고 혼란을 일으키는 결과를 훨씬 뒤에 갖게 된다.

### 2. 초기화되지 않은 메모리를 읽는 경우
- bss메모리 위치(초기값 없는 전역변수, static으로 선언된 변수, 배열)들은 항상 로더에 의해 0으로 초기화된다.
- heap메모리 위치는 calloc으로 할당해야 0으로 초기화됨. malloc으로 초기화하면 그렇지 않다.

### 3. 스택 버퍼 오버플로우 허용하기

```c
#include "stdio.h"

MAX_LENGTH = 4;

int main()
{
char str1[MAX_LENGTH];
char str2[MAX_LENGTH];

// gets 함수를 사용하여 문자열 입력 받기
printf("Enter a string using gets: ");
gets(str1);
printf("String entered using gets: %s\n", str1);

// fgets 함수를 사용하여 문자열 입력 받기
printf("Enter a string using fgets: ");
fgets(str2, sizeof(str2), stdin);
// fgets는 개행 문자도 읽어들이므로 개행 문자 제거
str2[strcspn(str2, "\n")] = '\0';
printf("String entered using fgets: %s\n", str2);

return 0;
}
```
![[img/Pasted image 20240417104700.png]]

### 4. 포인터와 이들이 가리키는 객체들이 같은 크기라고 가정하기
```c
/* Create an nxm array */
int **makeArray1(int n, int m){
	int i;
	int **A = (int **)malloc(n * sizeof(int));

	for (int i = 0; i < n; i++)
		A[i] = (int *)malloc(m * sizeof(int));
}
```

### 5. Off-by-One 에러 만들기
```c
/* Create an nxm array */
int **makeArray2(int n, int m){
	int i;
	int **A = (int **)malloc(n * sizeof(int *));

	for (int i = 0; i <= n; i++)
		A[i] = (int *)malloc(m * sizeof(int));
}
```

### 6. 포인터가 가리키는 객체 대신에 포인터 참조하기
```c
int *binheapDelete(int **binheap, int *size)
{
	int *packet = binheap[0];

	binheap[0] = binheap[*size - 1];
	*size--; /* This should be (*size)--*/
	heapify(binheap, *size, 0);
	return(packet);
}
```
6번 줄에서, **--와 * 연산자들은 동일한 우선순위**를 가지고 우에서 좌로 결합하기 때문에 이 코드는 실제로는 가리키는 정수 값 대신에 **포인터 자신을 감소**시킨다. 

### 7. 포인터 연산에 대한 오해
```c
int *search(int *p, int val)
{
	while(*p && *p != val)
		p += sizeof(int); /* Should be p++ */
	return p;
}
```

### 8. 존재하지 않는 변수 참조하기
```c
int *stackref()
{
	int val;

	return &val;
}
```
`stackref` 함수가 호출될 때마다 `val`이라는 지역 변수가 생성되고 해당 변수의 주소를 반환합니다. 그러나 `val`은 함수가 종료되면 스택 프레임과 함께 사라지는 지역 변수이므로, **`stackref` 함수가 종료되면 비록 포인터가 여전히 유효한 메모리 주소를 가리키지만, 더 이상 유효한 변수를 가리키지는 않는다.**

따라서 `stackref` 함수를 호출할 때마다 새로운 지역 변수 `val`이 생성되고 그 주소를 반환하지만, 이러한 주소는 함수가 종료된 후에는 더 이상 유효하지 않습니다. 다시 호출될 때마다 새로운 지역 변수가 생성되므로, 이전에 반환된 주소는 다른 지역 변수를 가리킬 것입니다.

따라서 이러한 함수의 반환 값인 포인터는 재사용될 수 없으며, 사용하려면 함수가 반환하는 포인터를 별도의 변수에 저장해야 합니다.

### 9. 가용 힙 블록 내 데이터 참조하기
```c
int *healpref(int n, int m)
{
int i;
int *x, *y;

x = (int *)malloc(n * sizeof(int));
free(x);
y = (int *)malloc(n * sizeof(int));

for (int i = 0; i < m; i++)
	y[i] = x[i]++; /* Oops! x[i] is a word in a free block */

return y;
}
```
### 10. 메모리 누수 leak 유발
```c
void leak(int n)
{
	int *x = (int *)malloc(n * sizeof(int));
	
	return; /* x is garbage at this pointer */
}
```
메모리 누수는 엔지니어가 **할당된 블록들을 반환하는 것을 잊어서** 힙에 가비지를 부주의하게 만들 때 일어나는 조용한 살인자다.
만일 leak이 자주 호출되면, 힙은 점차적으로 가비지로 가득 차게 되고, 최악의 경우 전체 가상 주소공간을 소모하게 된다. 메모리 누수는 특히 데몬 daemon, 서버와 같이 종료되지 않는 프로그램들의 경우에 심각하다.