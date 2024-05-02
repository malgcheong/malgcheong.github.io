---
title:  "OS Project 1 WIL"
date: 2024-05-02
excerpt: ""

categories: [PINTOS]
tags: [SW 사관학교 정글]

---
# 1. Alarm Clock
### timer interrupt
1. timer는 하드웨어를 통해 초당 PIT_FREQ 회 interrupt가 발생함.
2. 4tic마다 time slicing하여 멀티 태스킹을 할 스레드들이 골고루 cpu 연산 작업을 하기 위해 timer interrupt를 사용함.
3. interrupt는 지금 실행중인 thread를 잠깐 멈추고 그 interrupt에 해당하는 핸들러를 수행하도록 함. interrupt 핸들러가 수행되면 기존에 작업하던 스레드로 돌아감.

### Problem
1. pintos는 busy waitng 방식으로, ready queue에서 깨어날지 안할지를 thread_yield ()를 매번 발생시켜 cpu에서 확인함. cpu에서 확인하기 때문에 그만큼 쓸데없는 자원을 많이 소모함.

### Solve
1. sleep list를 만들어 thread가 wait해야할 시간동안 sleep list에 저장했다가 ready queue로 옮기도록 구현함.
```c
// 스레드를 sleep 시켜서 sleep_list에 넣습니다.
void thread_sleep(int64_t ticks) {
    struct thread *curr = thread_current();
    enum intr_level old_level;

    old_level = intr_disable();

    if (curr != idle_thread) {
        curr->wakeup_ticks = ticks;
        list_push_front(&sleep_list, &curr->elem);
        next_awake_ticks(curr->wakeup_ticks);
        thread_block();
    }

    intr_set_level(old_level);
}
```
2. thread 구조체에 자신의 thread가 일어날 시간인 wakeup_ticks를 설정함. thread가 일어날 단위가 tick이라서 timer interrupt 핸들러에서 sleep list를 확인하고 wakeup_ticks을 통해 wake up할 스레드를 찾으면 sleep list에서 삭제하고 ready queue에 추가함.
```c
// 일어날 시간이 되면 대기 큐에 업데이트 후 sleep_list에서 remove
void thread_wakeup(int64_t ticks) {
    struct list_elem *e = list_begin(&sleep_list);  // sleep_list 시작부터 순회
    
    if (ticks < next_tick_to_awake)
        return;

    while (e != list_end(&sleep_list))  // sleep_list에 마지막까지 봄
    {
        struct thread *t = list_entry(e, struct thread, elem);

        if (t->wakeup_ticks <= ticks) {
            e = list_remove(e);
                threaunblock(t);
        }
        else {
            e = list_next(e);
        }
    }
}
```
3. sleep list 조회의 효율성을 위해 가장 빨리 깨어날 thread의 시간을 전역변수로 설정함. 왜냐하면 지금 os의 tick이 sleep list에서 가장 빨리 일어나야할 thread보다 작다면 sleep list를 조회할 필요 없다. 깨울 thread가 있는 경우에만 sleep list를 조회했다.
```c
static int64_t next_tick_to_awake = NULL; // 전역변수로 선언

// 일어날 시간이 되면 대기 큐에 업데이트 후 sleep_list에서 remove
void thread_wakeup(int64_t ticks) {
        ... 
    if (ticks < next_tick_to_awake)
        return;
        ...
}
```

# 2. Priority Scheduling
### Problem
1. 기존에 구현되어 있던 것은 round-robin 방식임.
2. 파일을 다운하면서 게임을 하듯이 우선순위가 높은 게임이 먼저 실행되기 위해서 thread마다 우선순위를 부여해 스케줄링할 필요가 있음.

### Solve
1. ready queue에  priority를 내림차순으로 정렬해 우선순위가 높은 thread를 CPU에 thread_yield ()될 수 있도록 구현함.    
```c
/* CPU를 양보합니다. 현재 스레드는 슬립 상태로 전환되지 않으며,
    스케줄러의 변덕에 따라 즉시 다시 스케줄될 수 있습니다. */
void thread_yield(void) {
    struct thread *curr = thread_current();
    enum intr_level old_level;

    ASSERT(!intr_context());
    
    old_level = intr_disable();
    if (curr != idle_thread) {
        // list_push_back(&ready_list, &curr->elem);
        list_insert_desc_ordered(&ready_list, &curr->elem, compare_priority, PRIORITY);
    }
    do_schedule(THREAD_READY);
    intr_set_level(old_level);
}
```
```c
/* 값 비교. t1의 우선순위가 낮은 경우 true*/
bool compare_priority(struct list_elem *e1, struct list_elem *e2, void *aux) {
    struct thread *t1 = list_entry(e1, struct thread, elem);
    struct thread *t2 = list_entry(e2, struct thread, elem);
    int t1_priority = t1->priority;
    int t2_priority = t2->priority;

        // list_insert_desc_ordered에서 e1, e2 순서 반대여서 첫번째 요소가 작을 경우가 참임
    if (t1_priority < t2_priority) { 
        return true;
    }
    return false;
}
```
2.  thread가 생성 될 때와 cpu의 priority가 변경 될 때 ready queue의 첫 thread의 priority와 cpu에 올라간 thread의 priority를 비교해 thread_yield () 할 수 있도록 구현함. 높은 우선순위의 스레드가 생기자마자 해당 스레드를 cpu에서 사용하기 위해서임.
```c
tid_t thread_create(const char *name, int priority, thread_func *function, void *aux) {
        ...
    /* 준비 큐에 추가. */
    thread_unblock(t);

    // 우선순위 스케줄링
    test_max_priority();
        ...
}
```
```c
/* 현재 스레드의 우선순위를 NEW_PRIORITY로 설정합니다. */
void thread_set_priority(int new_priority) {
    thread_current()->priority = new_priority;
    test_max_priority();
}
``` 
```c
// 우선순위 스케줄링 하는 함수
void test_max_priority(void) {
    struct thread *curr = thread_current();
    struct list_elem *highest_elem = list_begin(&ready_list);
    if (list_end(&ready_list) == (highest_elem)) {
        return;
    }

    if (compare_priority(&curr->elem, highest_elem, NULL)) {
        thread_yield();
    }
}
```

# 3. Priority Scheduling - priority inversion
### Problem
1. 낮은 우선순위의 thread가 공유자원에 lock을 건 채로 cpu에서 연산작업을 하다가 
높은 우선순위의 thread가 생성되면 thread_yield ()가 발생해 context switching이 발생한다. 
높은 우선순위의 thread가 cpu에 올라가 동일한 공유자원에 lock을 얻고 싶어도 cpu를 반환하게 되고 
우선순위가 낮은 thread가 먼저 실행되는 우선순위 역전의 문제가 발생한다.
2. 혹은 낮은 우선순위의 thread가 lock을 쥔채로 대기 상태로 들어가고 
해당 lock을 갖고 싶은 높은 우선순위의 thread가 계속 cpu를 선점한다면 dead lock이 발생한다. 

### Solve (donation)
1. 위의 문제를 해결하기 위해선  priority donation이 필요하다.
2. lock 잠금시 이미 lock을 소유한 thread에게 높은 우선순위의 thread의 priority를 상속한다.
```c
void donate_priority(void) {

    int depth;
    struct thread *curr = thread_current();

    for (depth = 0; depth < 8 ; depth++) {
        if(!curr->wait_on_lock) 
            break;
        struct thread *holder = curr->wait_on_lock->holder;
        holder->priority = curr->priority;
        curr = holder;
    }
}
```
3. lock 해제시 priority를 반환한다. 하지만 그 lock을 원하는 우선순위의 thread로 바뀌거나 없다면 자기 자신의 priority로 바뀐다.
```c
void lock_release (struct lock *lock) {
    ASSERT (lock != NULL);
    ASSERT (lock_held_by_current_thread (lock));

    remove_with_lock(lock);
    refresh_priority();

    lock->holder = NULL;
    sema_up (&lock->semaphore);
}
``` 
```c
void remove_with_lock(struct lock *lock) {
    struct list_elem *e;
    struct thread *curr = thread_current();

    for (e = list_begin(&curr->donations); e != list_end(&curr->donations); e = list_next(e)) {
        struct thread *t = list_entry(e, struct thread, donation_elem);
        if (t->wait_on_lock == lock) 
            list_remove(&t->donation_elem);
    }
}
```
```c
void refresh_priority(void) {
    struct thread *curr = thread_current();
    curr->priority = curr->init_priority;
    
    if (!list_empty(&curr->donations)) {
        list_sort(&curr->donations, thread_compare_donate_priority, NULL);
        struct thread *front = list_entry(list_front(&curr->donations), struct thread, donation_elem);
        if (front->priority > curr->priority)
            curr->priority = front->priority;
    }
}
```