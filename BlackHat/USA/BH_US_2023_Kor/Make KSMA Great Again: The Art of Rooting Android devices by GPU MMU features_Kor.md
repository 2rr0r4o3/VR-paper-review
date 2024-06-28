# Make KSMA Great Again: The Art of Rooting Android devices by GPU MMU features

1. 소개 (Introduction)
이 발표자료는 Alibaba Cloud Pandora Lab의 Wang Yong이 작성한 것이다. 발표자는 Android와 Chrome의 취약성 연구를 전문으로 하며, KSMA (Kernel Space Mirroring Attack) 기술의 GPU 버전을 소개한다. 주요 내용은 GPU MMU 기능을 활용하여 안드로이드 장치를 루팅하는 방법을 설명한다.

2. Linux 커널 VMM 101 (Linux Kernel VMM 101)
리눅스 커널의 가상 메모리 관리(VMM)에 대한 기본 개념을 설명한다. 주요 구조체로는 mm_struct와 vm_area_struct가 있으며, 각각 프로세스의 메모리 맵과 가상 메모리 영역을 관리한다.

구조체 설명
  - mm_struct:

  ```
  struct mm_struct {
    struct vm_area_struct *mmap; /* list of VMAs */
    struct rb_root mm_rb;
    unsigned long mmap_base; /* base of mmap area */
    pgd_t *pgd;
    struct file __rcu *exe_file; /* reference to executable file */
  };
  ```

  - vm_area_struct:

  ```
  struct vm_area_struct {
    unsigned long vm_start;
    unsigned long vm_end;
    struct rb_node vm_rb;
    unsigned long vm_flags;
    const struct vm_operations_struct *vm_ops;
    unsigned long vm_pgoff;
    struct file *vm_file;
  };
  ```
가상 메모리 주소 변환
  - PGD, PMD, PTE를 통한 페이지 프레임 주소 변환 과정을 설명한다.
    - pgd_offset(), pmd_offset(), pte_offset() 함수를 사용하여 각 단계의 주소를 계산한다.

3. KSMA (Kernel Space Mirroring Attack)
KSMA는 커널 공간을 미러링하여 직접 읽기/쓰기를 가능하게 하는 공격 기법이다. ARMv8-64 아키텍처의 페이지 테이블을 조작하여 커널 주소 공간에 접근한다.

ARMv8-64 레벨 설명

레벨 0, 1, 2, 3 디스크립터 형식
안드로이드에서는 레벨 0 테이블이 없다.

공격 방법
커널 가상 주소에 직접 읽기/쓰기
Level 1 테이블을 사용하여 1GB 영역을 설정하고, PA를 매핑하여 커널 메모리에 접근

4. GPU 버전 KSMA (GPU Version of KSMA Exploitation Technique)
GPU 메모리 관리를 활용하여 KSMA를 수행하는 방법을 설명한다. CPU와 GPU가 동일한 가상 주소를 공유하고 독립된 주소 공간을 가진다.

GPU 메모리 관리
Unified Memory Sharing: CPU와 GPU가 동일한 가상 주소를 공유
Virtual Memory Management: 가상 주소와 물리 메모리 관리, GPU MMU의 동작 원리 설명

가상 주소 관리 구조체
  - kbase_va_region:

  ```
  struct kbase_va_region {
    struct rb_node rblink;
    struct list_head link;
    u64 start_pfn; // virtual address
    size_t nr_pages;
    size_t initial_commit;
    unsigned long flags;
    struct kbase_mem_phy_alloc *cpu_alloc;
    struct kbase_mem_phy_alloc *gpu_alloc;
    struct list_head jit_node;
    u16 jit_usage_id;
    u8 jit_bin_id;
    int va_refcnt;
  };
  ```

  - kbase_mem_phy_alloc
  
  ```
  struct kbase_mem_phy_alloc {
    struct kref kref;
    atomic_t gpu_mappings;
    atomic_t kernel_mappings;
    size_t nents;
    struct tagged_addr *pages;
    struct list_head mappings;
    struct list_head evict_node;
    size_t evicted;
    struct kbase_va_region *reg;
    enum kbase_memory_type type;
    struct kbase_vmap_struct *permanent_map;
    u8 properties;
    u8 group_id;
    union {umm, alias, native, user_buf} imported;
  };
  ```

물리 페이지 관리
페이지 할당: kctx->mem_pools에서 할당, 부족하면 kbdev->mem_pools에서 할당, 마지막으로 커널에서 할당
페이지 해제: kctx->mem_pools에 추가, 부족하면 kbdev->mem_pools에 추가, 마지막으로 커널에 반환

5. MMU (Memory Management Unit)
GPU MMU의 동작을 설명한다. CPU MMU와의 비교를 통해 GPU MMU의 동작 원리를 이해하고, 커널 드라이버를 통해 학습한다.

ATE (Address Translation Entry) 디스크립터 형식
페이지 테이블 항목의 디스크립터 형식을 설명
ENTRY_ATTR_BITS, ENTRY_ACCESS_RW, ENTRY_ACCESS_RO, ENTRY_SHARE_BITS, ENTRY_ACCESS_BIT, ENTRY_NX_BIT 등

6. 블록 디스크립터 (Block Descriptor)
유효한 블록 엔트리를 작성하는 방법을 설명한다. GPU MMU가 블록 디스크립터를 지원하는지 확인하고, 어느 레벨(L2-2MB, L1-1GB, L0-512GB)에서 사용할 수 있는지 설명한다.

7. GPU 읽기/쓰기 (GPU READ/WRITE)
GPU 명령어 세트를 역공학하여 GPU 메모리에 접근하는 방법을 설명한다. OpenCL을 사용하여 GPU에서 메모리 읽기/쓰기를 수행한다.

OpenCL code example
```
const char* gpu_code =
__kernel void rw_mem(__global unsigned long *p0, __global unsigned long *p1, __global unsigned long *p2) {  // p0 – dest, p1 – src, p2 – rw_flag
   size_t idx = get_global_id(0);
   if (p2[idx]) { // write
       __global unsigned long *addr = (__global unsigned long)(p0[idx]);
       addr[0] = p1[idx];
   } else { // read
       __global unsigned long *addr = (__global unsigned long *)(p1[idx]);
       p0[idx] = addr[0];
   }
};
```
8. 결론 (Conclusion)
GPU 메모리 관리를 통한 KSMA 기법의 효율성을 설명하며, 안드로이드 장치의 보안성을 높이기 위한 방안을 제안한다.

이 발표자료는 GPU MMU 기능을 활용한 KSMA 기법을 자세히 설명하며, 안드로이드 장치에서의 루팅을 위한 새로운 공격 표면과 기술을 탐구한다.

