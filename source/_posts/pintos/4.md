---
title: "OS Project 3 WIL"
date: 2024-05-21 02:53:18
excerpt: ""

categories: [PINTOS]
tags: [SW 사관학교 정글]
cover: /images/pintos/part3 virtual memory.jpg

---

추후에 정리
# 반효경 운영체제 가상메모리
http://www.kocw.net/home/search/kemView.do?kemId=1046323
https://asfirstalways.tistory.com/130
https://product.kyobobook.co.kr/detail/S000001772604
- Memory Management 1~4: 하드웨어가 하는 일(주소 변환, 물리메모리 관리)
- Virtual Memory 1~2: 운영체제가 하는 일(page table, io 작업)


## 주소 바인딩
**logical address (=virtual address)**
- 프로세스마다 독립적으로 가지는 주소 공간
- 각 프로세스마다 0번지부터 시작
- **CPU가 보는 주소는 logical address임**(mmu가 변환해서 실제 메모리에서 값 가져옴)

**Physical address**
- 메모리에 실제 올라가는 위치

**Symbolic address**
- 프로그래머들이 특정 이름을 통해 변수를 지정하고 값을 저장할 때, 그 변수를 하드웨어의 몇 번지에 저장할지 정하지 않는다. 또한 그 변수를 CPU가 어떻게  인식할지도 정하지 않는다. 그저 변수의 이름을 통해서 그 값에 접근할 수 있는 것이다. 그 변수를 Symbolic Address라고 한다. 이 주소가 컴파일 되어 숫자 주소가 만들어지고 이것이 물리적인 메모리와 매핑되는 것이다. 이를 주소 바인딩이라고 한다.

## Dynamic loading, overlay
overlay가 완전 초창기 개념,
dynamic loading은 그 후에 나옴(라이브러리로 구현해서 더 편함)
실제 paing 기법의 dynamic loading과는 달른 의미에서 쓰임

## Dynamic Linking
실행파일과는 별도로 라이브러리를 두고 유저프로그램이 실행하다가 라이브러리 호출 부분에 라이브러리 루틴의 위치를 찾기 위한 stub이라는 작은 코드를 둔대 (해당 라이브러리가 있는 포인터)

별도의 프로그램에 속하기 때문에 100개의 copy가 메모리에 올라감 (static)

실행할 수 있는 라이브러리 위치코드만 두면 100명이 유저프로그램을 실행할때 메모리를 하나만 보게됨 그래서 이를 shared memory(shared object, dll)라고 함

## inner page table은 2^10으로 표현해야함
2^32 프로세서에서 2단계로 페이지테이블을 표시하면 아래와 같다.
이때 왜 페이지 넘버를 2^10 두번으로 했는지 그 이유도 알아야 한다.
inner page table이나 ouuter page table이나 모두 하나의 페이지에 해당한다.(메모리에 있어야 하니) 
pte는 4byte니 page table이 표현가능한 주소는 1024개다.
그러니 p2는 2^10으로 1024개를 표현해야 페이지를 전부 사용가능하다. 그래서 p2가 2^10이고 그러한 inner page table의 개수가 또 2^10(outter page table)만큼 있다.


그렇다면 2^64 프로세서에선?
그때는 한 페이지에 8바이트의 pte가 512개가 들어가니 inner page table을 표현하기 위해선 2^9이 필요하다.

## 다단계 페이지 테이블 메모리 효율 안 좋아도 쓰는 이유
다단계 페이지 테이블은 5번에 메모리 접근 필요함. 메모리 변환에 4번, 실제 메모리 접근에 1번에서 오버해드가 크다. 근데 TLB가 있으니 이러한 과정을 생략 가능. 메모리는 멀티레벨로 줄이고 시간은 TLB로 줄이고!
effective memory access time으로 설명 가능함

## PTE가 추가로 가지는 비트 정보
protection이 다른 프로세스가 내 메모리 보는 그런 거 막아준다는 의미가 아님(가상메모리 그 자체가 protection을 이미 해줌)
valid는 pte가 꼭 1024개 혹은 512개 전부 존재하기 때문에 그 중에 frame에 존재하지 않는 주소는 표시해줘야함 (code, data, stack 은 다 전부 다른 page에 있음 stack은 맨 위 페이지, code는 맨 아래 페이지, 그중에 빈 페이지도 존재)

## 페이지 폴트 처리 루틴이 운영체제에 정의되어 있다.
대부분의 경우 페이지 폴트가 안 난대 (0.98???? 교수님이 잘못 말하신듯) ~0.01래
- 많은 데스크톱 및 서버 시스템에서 페이지 폴트율은 일반적으로 **0.1%에서 1%** 범위에 있습니다.
- 이 정도의 페이지 폴트율은 대부분의 작업에서 수용 가능하며, 성능에 큰 영향을 미치지 않습니다.
페이지폴트가 한번 나면 오버헤드가 엄청 크다.
하드웨어에서 os로 제어를 넘기는 시간(page fault handler 하기) + 필요하다면 swap page out하고 swap page in하고 다시 cpu한테 제어 넘기기

## swap out은 page replacement로 정한 page를 swap out함
가능하면 페이지 폴트 안 나게, 잘 쓰는 페이지는 남겨둬야지.
페이지가 메모리에 올라오고 나서 write된적이 없다면 victim을 swap out할 필요 없이 그냥 삭제만 하면 되고 write한 적이 있다면 swap out을 해야함(백킹 storage에 저장해야 하니깐)

swap out 및 destory되면 page table에 invalid로 표현하고 새로 swap in하는 페이지는 valid로 해야함

## page replacement
- optimal algorithm은 upper bound로서 다른 알고리즘의 참고용으로 쓰임
- fifo (미래를 모르면 우선 과거부터 보자, 먼저 들어온걸 지움)
- lru (가장 오래전에 참조된건 지움)
- lfu (참조횟수가 가장 적은 페이지를 지움)

> 캐슁에서도 이런 replacement 전략이 당연히 쓰임

주소변환은 하드웨어에서 하는일이므로 page table 읽어서 물리메모리 참조하는 건 cpu가 알아서 하는 일이라 운영체제가 중간에 낄 수가 없다.
page fault가 나면 disk 접근이 필요해서 트랩이 발생하고 제어권이 cpu에서 운영체제로 넘어가게된다. 그때 운영체제가 필요없는 페이지를 replacement해야하는데(ram이 가득차면) LRU, LFU를 할 수 있을까?
답은 알수가없다. 왜냐하면 페이지 폴트가 안 나면 운영체제가 하는일이 없기때문에 물리메모리 접근횟수를 운영체제가 알 수가 없다.
그러니 참조할때마다 list를 변경해줘야하는데 그걸 운영체제가 할 수 없다 (제어권이 안 넘어오므로) 
buffer, web caching에서는 lru, lfu를 쓸 수 있다. 나중에 설명

## 그래서 사용하는 알고리즘이 Clock Algorithm
Reference bit는 하드웨어가  (이미 메모리에 존재하는 페이지를 참조하면 cpu가 해당 페이지를 읽어들이면서 1로 바꿔줌) 그걸 운영체제가 페이지 폴트시 replace해야될 상황이면 clear해주는 거지

read가 아니라 write로 참조도 할 수 있지. modified bit(dirty)가 0이면 replace시 그냥 destory하면 되고 1이면 disk에 반영하고 지워야함.

## Thrashing
프로세스 하나면 IO작업할때 다른 프로세스 일 못하니 CPU 이용량이 좋지 않음
프로세스를 늘릴수록 cpu가 쉴시간이 없이 프로세스들이 작업됨
근데 어느 순간부터 cpu 이용량이 줄기시작하는데, 그 부분이 thrashing이 시작하는 부분임
프로세스마다 각자 할당된 페이지가 작아서 프로세스 실행할려면 자꾸 페이지 폴트가 발생함.
그러면 io 작업이 많아지고 cpu가 쉬는 시간이 많아짐

그래서 동시에 올라가는 프로세스의 수를 조절해줘야함. 즉 적어도 프로그램이 어느정도는 메모리 확보할 수 있도록 보장해줘야함 그걸 해주는 알고리즘이 working-set algorithm, PFF(page-fault frequency) scheme이다.


# Memory Management
# Anonymous page
# Stack Growth
# Memory Mapped file
# Swap In/Out
# Copy-on-Write(Extra)