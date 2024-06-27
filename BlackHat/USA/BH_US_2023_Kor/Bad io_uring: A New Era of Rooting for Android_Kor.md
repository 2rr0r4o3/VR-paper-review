# Bad io_uring: New Attack Surface and New Exploit Technique to Rooting Android

1. 서론 (Introduction)
io_uring은 Linux 커널 5.1 버전에서 도입된 고성능 비동기 I/O 프레임워크이다. 이 프레임워크는 디스크 및 네트워크 I/O 작업의 효율성과 확장성을 높이고 지연 시간을 줄이며, 문맥 전환과 CPU 사용을 최소화한다. io_uring은 큰 공격 표면을 제공하는데, 이는 더 많은 코드가 더 많은 버그를 의미하기 때문이다. Syzkaller를 통해 발견된 161개의 버그가 있으며, io_uring은 AOSP(Android Open Source Project)에서 기본적으로 활성화되어 있어 안드로이드에서도 새로운 공격 표면을 제공한다.

2. CVE-2022-20409
CVE-2022-20409는 io_uring에서 발생하는 Use-After-Free(UAF) 버그이다. 이 섹션에서는 버그의 원인과 메모리 손상 가능성에 대해 설명한다.

2.1 버그의 원인 (The root cause)
io_uring_task 구조체는 각 커널 스레드에 대한 작업별 데이터를 저장하는 객체이다. 일반적으로 identity 필드는 중첩 구조체 __identity를 가리킨다.

```
struct io_uring_task {
    struct xarray xa;
    struct wait_queue_head wait;
    struct file *last;
    struct percpu_counter inflight;
    struct io_identity __identity;
    struct io_identity *identity;
    atomic_t in_idle;
    bool sqpoll;
};
```

io_req_init_async 함수는 비동기 요청을 초기화한다.

```
static inline void io_req_init_async(struct io_kiocb *req) {
    struct io_uring_task *tctx = current->io_uring;
    req->work.identity = tctx->identity; // [1]
    if (tctx->identity != &tctx->__identity) // [2]
        refcount_inc(&req->work.identity->count); // [3]
}
```
io_put_identity 함수는 identity를 해제한다.

```
static void io_put_identity(struct io_uring_task *tctx, struct io_kiocb *req) {
    if (req->work.identity == &tctx->__identity)
        return;
    if (refcount_dec_and_test(&req->work.identity->count)) // [4]
        kfree(req->work.identity); // [5]
}
```

두 함수 호출 간의 불일치로 인해 버그가 발생한다. 두 작업(task_a와 task_b)을 가정했을 때, io_req_init_async에서 req(task_a)->work.identity가 task_b-io_uring-identity로 초기화되지만 참조 카운트가 증가하지 않는다. 이후 io_put_identity에서 참조 카운트가 감소하고 identity가 해제된다. 이는 잘못된 해제를 유발한다.

2.2 메모리 손상 (The memory corruption)
Figure 1은 io_uring_task 객체를 해제하기 전과 후의 메모리 레이아웃을 보여준다. 메모리 손상으로 인해 identity가 댕글링 포인터가 된다. 공격자는 해제된 메모리에 가짜 identity 객체를 만들거나 identity 객체를 해제하여 커널 힙에서 잘못된 해제를 일으킬 수 있다.

3. 안드로이드에서의 익스플로잇 난해성 (The Exploitation Challenges on Android)
안드로이드 커널을 익스플로잇하는 것은 여러 가지 이유로 높은 난이도를 지닌다:

제한된 기능: 안드로이드 커널에는 Linux 배포판에서 사용 가능한 많은 기능이 없으며, 추가적인 보안 조치가 적용되어 있다.
강화된 보안: 안드로이드 커널에는 커널 실행 흐름을 검증하는 Kernel Control-Flow Integrity(KCFI)와 같은 추가적인 보안 기능이 있다.
엄격한 접근 제어: 안드로이드 운영 체제는 시스템 자원에 대한 접근을 엄격히 제어하여 커널 접근이 제한된다.

4. 슬랩을 벗어나기 – 페이지 스프레이 (Jump out of the slab – Page Spray)
대부분의 커널 익스플로잇 전략은 스프레이 객체를 사용하여 커널 메모리를 덮어쓰는 것이다. 그러나 안드로이드에서는 적절한 스프레이 객체를 찾기 어렵다. 대신 페이지를 스프레이하여 목표 메모리를 덮어쓰는 방식을 사용한다.

4.1 페이지 스프레이 기술 (The Page Spray Technique)
슬랩의 모든 객체가 해제되면 메모리 페이지가 buddy allocator로 반환된다. 이를 재획득하여 해제된 페이지에 가짜 객체를 만들 수 있다.

4.2 페이지 후보 (Page Candidates)
페이지 스프레이 기술은 슬랩 할당기와 공유되는 페이지를 할당해야 한다. io_uring과 pipe 서브시스템에서 이러한 페이지를 찾을 수 있다.

```
static void *io_mem_alloc(size_t size) {
    gfp_t gfp_flags = GFP_KERNEL | __GFP_ZERO | __GFP_NOWARN | __GFP_COMP | __GFP_NORETRY;
    return (void *) __get_free_pages(gfp_flags, get_order(size));
}
```

io_uring 서브시스템에서는 buddy allocator를 통해 커널 페이지를 할당하여 사용자 공간에서 페이지 내용을 직접 수정할 수 있다.

파이프 서브시스템에서도 페이지를 할당하여 파이프에 작성된 데이터를 저장할 수 있다. 그러나 여기서는 페이지 크기를 제어할 수 없고, 사용자 공간에서 직접 데이터를 수정할 수 없다.

```
static ssize_t pipe_write(struct kiocb *iocb, struct iov_iter *from) {
    if (!page) {
        page = alloc_page(GFP_HIGHUSER | __GFP_ACCOUNT);
        if (unlikely(!page)) {
            ret = ret ? : -ENOMEM;
            break;
        }
        pipe->tmp_page = page;
    }
}
```

5. 임의 읽기/쓰기 달성 (Achieving Arbitrary Read/Write)
페이지 스프레이 기술을 사용하여 UAF 객체를 조작할 수 있다. UAF 객체 자체는 사용자 공간으로 전송될 수 있는 정보를 포함하지 않으며, 정보 유출 없이 메모리 손상을 활용할 수 없다.

이를 해결하기 위해 취약점 능력을 피벗팅한다. Figure 3은 이러한 피벗팅 과정을 보여준다. UAF 객체와 같은 슬랩에 pipe_buffer 객체를 할당하고, UAF 객체를 해제한 후 나머지 객체를 해제하여 페이지를 자유롭게 만든다. 페이지 스프레이 기술을 사용하여 해제된 슬랩 페이지를 재획득하고 pipe_buffer 객체를 덮어쓴다.

```
struct pipe_buffer {
    struct page *page;
    unsigned int offset, len;
    const struct pipe_buf_operations *ops;
    unsigned int flags;
    unsigned long private;
};
```

pipe_buffer 객체는 페이지 필드와 함수 테이블 포인터 ops를 포함한다. 이 필드를 활용하여 커널 정보를 유출하고 임의 읽기/쓰기를 달성할 수 있다.

6. 삼성 KNOX 우회 (Bypassing Samsung’s KNOX RKP)
삼성 KNOX는 안드로이드 커널에 추가적인 보호를 제공한다. KNOX는 보안 모니터를 활용하여 커널 속성을 보호하며, cred 객체를 보호한다. 하지만, KNOX는 슬랩 캐시 메타데이터를 변조하여 우회할 수 있는 취약점을 가지고 있다.

6.1 삼성 KNOX RKP 설계 (Samsung’s KNOX RKP Design)
KNOX는 중요한 속성을 읽기 전용 메모리에 보호하며, 커널이 메모리를 수정할 때 하이퍼바이저와 통신하여 메모리 권한을 변경한다. 이를 통해 공격자가 cred를 직접 수정할 수 없다. 그러나 KNOX는 cred를 검증할 때 슬랩 캐시 메타데이터를 확인하지 않아, 이를 변조하여 우회할 수 있다.

```
int security_integrity_current(void) {
    const struct cred *cur_cred = current_cred();
    rcu_read_lock();
    if (kdp_enable &&
        (is_kdp_invalid_cred_sp((u64)cur_cred, (u64)cur_cred->security) ||
         cmp_sec_integrity(cur_cred, current->mm))) {
        rcu_read_unlock();
        panic("KDP CRED PROTECTION VIOLATION\n");
    }
    rcu_read_unlock();
    return 0;
}

static inline bool is_kdp_invalid_cred_sp(u64 cred, u64 sec_ptr) {
    struct task_security_struct *tsec = (struct task_security_struct *)sec_ptr;
    if (!is_kdp_protect_addr(cred) ||
        !is_kdp_protect_addr(cred + cred_size) ||
        !is_kdp_protect_addr(sec_ptr) ||
        !is_kdp_protect_addr(sec_ptr + tsec_size)) {
        return true;
    }
    if ((u64)tsec->bp_cred != cred) {
        return true;
    }
    return false;
}
```

KNOX는 cred가 보호된 메모리 영역에 있는지 확인하지만, 페이지 메타데이터는 보호하지 않는다. 공격자는 페이지 메타데이터를 변조하여 cred 객체를 위조할 수 있다.

7. 결론 (Conclusion)
io_uring은 데스크탑뿐만 아니라 안드로이드 AOSP에도 큰 공격 표면을 제공한다. 안드로이드에서 io_uring을 제한하는 것만으로는 충분하지 않으며, DirtyPage(page spray)와 같은 새로운 익스플로잇 기법이 필요하다. 논문은 io_uring의 취약점을 악용하여 안드로이드 디바이스에서 루팅을 수행하는 방법을 설명하고, 이를 방어하기 위한 다양한 방법을 제안한다.

이 논문은 io_uring의 기술적 세부 사항과 이를 악용한 익스플로잇 기법을 심도 있게 다루고 있으며, 기존의 보안 메커니즘이 가진 한계와 보완점을 논의한다.
