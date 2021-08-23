# Embedded OS Project
임베디드 OS 개발 프로젝트 Navilos를 통해 ARM 기반 펌웨어/RTOS의 원리와 구조 파악하기.

(Navilos : https://github.com/navilera/Navilos)

<br/>

# Development Environment

**OS :** Ubuntu 14.04 LTS (VMware)

**Compiler :** ARM-GCC

**QEMU :** ARM RealView PB-A8

<br/>

# Contents

**TREE**

```
.
├── Makefile
├── boot
│   ├── Entry.S
│   ├── Handler.c
│   └── Main.c
├── hal
│   ├── HalInterrupt.h
│   ├── HalTimer.h
│   ├── HalUart.h
│   └── rvpb
│       ├── Interrupt.c
│       ├── Interrupt.h
│       ├── Regs.c
│       ├── Timer.c
│       ├── Timer.h
│       ├── Uart.c
│       └── Uart.h
├── include
│   ├── ARMv7AR.h
│   ├── MemoryMap.h
│   ├── memio.h
│   ├── stdarg.h
│   ├── stdbool.h
│   └── stdint.h
├── kernel
│   ├── Kernel.c
│   ├── Kernel.h
│   ├── event.c
│   ├── event.h
│   ├── msg.c
│   ├── msg.h
│   ├── synch.c
│   ├── synch.h
│   ├── task.c
│   └── task.h
├── lib
│   ├── armcpu.c
│   ├── armcpu.h
│   ├── stdio.c
│   ├── stdio.h
│   ├── stdlib.c
│   └── stdlib.h
└── navilos.ld
```

**Makefile**

- 빌드 자동화를 위한 Makefile이다. 
- 추가로 QEMU를 실행하는 명령어도 작성하여 make run 명령어를 통해 빌드와 QEMU 실행까지 한번에 동작한다.

**boot** 

- Entry : 어셈블리 파일이다. 메인으로 진입하기 전 실행되는 파일로 익셉션 벡터 테이블과 익셉션 핸들러로 구성되어 있다. 리셋 익셉션 핸들러 마지막에 main 진입 코드가 존재한다.
- Handler : 인터럽트 핸들러를 구현한 파일이다. ARM은 모든 인터럽트를 IRQ/FIQ 핸들러로 처리한다. 따라서 IRQ/FIQ 두 개의 핸들러 함수가 존재한다(IRQ/FIQ 핸들러 내에서 다시 개별 인터럽트의 핸들러를 구분하여 실행시킨다). 
- Main : main 함수가 존재하는 파일이다.

**HAL** 

- Uart : 콘솔 입출력용으로 사용하기 위한 하드웨어 모듈이다. printf() 함수의 일부 기능을 구현한 debug_printf() 함수를 만들어 디버깅 용도로 사용하고, 키보드 입력을 인터럽트로 수신한다.
- Timer : 시간을 측정하기 위한 하드웨어 모듈이다. delay() 함수를 만든다.
- Interrupt : 인터럽트 컨트롤러 하드웨어 모듈이다. 주변장치 하드웨어를 인터럽트 컨트롤러에 연결하여 최종적으로 인터럽트 전담 처리 코드가 있는 인터럽트 서비스 루틴으로 진입하게 한다.

**Kernel** 

- task : 운영체제가 자원을 할당하여 처리를 할 경우의 일의 단위라고 할 수 있다. 개별 태스크 자체를 추상화하기 위한 자료 구조로서 태스크 컨트롤 블록을 가진다.
- event : 인터럽트와 태스크 간의 연결매체이다. RTOS 커널이 태스크를 관리하고 있으므로 인터럽트 핸들러에서 실행되는 기능을 태스크로 옮기는 것이 더 좋다(태스크 간에도 발생한다). 
- msg : 임의의 데이터를 메시지라는 이름으로 전송하는 기능이다. 이벤트와 함께 이벤트와 관련된 데이터를 전송하여 두 기능을 효과적으로 활용한다.   
- synch : 크리티컬 섹션 이라고 판단되는 작업이 끝날 때까지 컨텍스트 스위칭이 발생하지 않는 상태로 만들어주는 기능이다.

**각 파트의 세부 개발은 아래 블로그 참고.**

**(Blog : https://sangukisme.github.io/categories/#embedded-rtos)**

<br/>

# Demo

### **UART Interrupt Handler.**

```c
static void interrupt_handler(void)
{
    uint8_t ch = Hal_uart_get_char();
   
	if (ch == 'U')
	{
		Kernel_send_events(KernelEventFlag_Unlock);
		return;
	}

	if (ch == 'X')
	{
        Kernel_send_events(KernelEventFlag_CmdOut);
        return;
	}

	Hal_uart_put_char(ch);
    Kernel_send_msg(KernelMsgQ_Task0, &ch, 1);
    Kernel_send_events(KernelEventFlag_UartIn);
}
```

- UART 인터럽트 핸들러로 등록된 함수. UART 입력(키보드 입력)이 발생하면 실행된다.
- 'U' 입력 : Unlock 이벤트 플래그를 발생시킨다.
- 'X' 입력 : CmdOut 이벤트 플래그를 발생시킨다.
- 그 외 입력 : UartIn 이벤트 플래그를 발생시킨다. 동시에 입력된 문자를 Task0의 메시지 큐로 보낸다.

### Critical Section 함수.

```c
static uint32_t shared_value;
static void Test_critical_section(uint32_t p, uint32_t taskId)
{
    //Kernel_lock_mutex();
    
    debug_printf("LOCK");
    debug_printf("User Task #%u Send=%u\n", taskId, p);
    shared_value = p;
    delay(1000);
    Kernel_yield();
    debug_printf("User Task #%u Shared Value=%u\n", taskId, shared_value);
    debug_printf("UNLOCK");
    
    //Kernel_unlock_mutex();
}
```

- 크리티컬 섹션 함수로 여러 태스크 혹은 여러 코어가 공유하는 변수(shared_value)의 값을 함수 내에서 사용한다.
- 중간에 스케줄링 함수인 Kernel_yield()를 호출하여 억지로 스케줄링을 한다. 크리티컬 섹션 상태에서의 공유 자원의 값이 원치않는 값으로 바뀌는 것을 확인하기 위함이다.
- 동기화(mutex) 사용시 : 공유 자원에 대한 잠금(lock)에 성공한 태스크가 공유 자원에 대한  해제(unlock)를 하기 전까지 다른 태스크가 공유 자원에 접근 할 수 없다.
- 동기화(mutex) 미사용시 : 공유 자원에 먼저 접근한 태스크가 작업을 마치기 전에 다른 태스크에서 공유 자원의 값을 바꿔버릴 수 있다.
- 동기화가 정상적으로 작동한다면 아래 예시처럼 LOCK과 UNLOCK사이에서 출력되는 Send와 Shared Value의 값이 같아야 한다. 
  - LOCK
  - User Task #'a' Send='b' 
  - ... 스케줄링 ...
  - User Task #'a' Shared Value='b'
  - UNLOCK

### **Task0.**

```c
void User_task0(void)
{
    while(true)
    {
        KernelEventFlag_t handle_event = Kernel_wait_events(KernelEventFlag_UartIn|KernelEventFlag_CmdOut);
        switch(handle_event)
        {
        
        ... 중략 ...
            
            case KernelEventFlag_CmdOut:
                Test_critical_section(5, 0);
                break;
        }
        Kernel_yield();
    }
}
```

- 이벤트 플래그를 읽는다.
- 'X' 키를 입력할 때마다 크리티컬 섹션 함수를 호출하여 공유 자원에 숫자 5를 저장한다.
- while문의 마지막에 스케줄링을 통해 다음 Task로 넘어간다.

### **Task1.**

```c
void User_task1(void)
{
    while(true)
    {
        KernelEventFlag_t handle_event = Kernel_wait_events(KernelEventFlag_CmdIn|KernelEventFlag_Unlock);
        switch(handle_event)
        {
        
        ... 중략 ...
            
            case KernelEventFlag_Unlock:
                Kernel_unlock_mutex();
                break;
        }
        Kernel_yield();
    }
}
```

- 이벤트 플래그를 읽는다.
- 'U' 키를 입력할 때마다 공유 자원에 대한 잠금을 해제(unlock)한다. 
- while문의 마지막에 스케줄링을 통해 다음 Task로 넘어간다.

### **Task2.**

```c
void User_task2(void)
{
    while(true)
    {
        Test_critical_section(3, 2);
        Kernel_yield();
    }
}
```

- 크리티컬 섹션 함수를 호출하여 공유 자원에 숫자 3을 저장한다.
- while문의 마지막에 스케쥴링을 통해 다음 Task로 넘어간다.

### **Output.**

**Case1. 동기화(mutex) 미사용**

<p align="center"><img src="img/동기화 미사용.png"></p>

- 동기화를 사용하지 않으면 공유 자원에 먼저 접근한 태스크가 작업을 마치기 전에 다른 태스크에서 공유 자원의 값을 바꿔버릴 수 있다.
- 위 그림처럼 Task2에서 공유 자원에 접근해 2를 저장하고 이후 Task0에서 공유 자원에 접근해 5를 저장한다. 그 결과 Task2는 공유 자원을 자신이 저장한 2가 아닌 5라는 잘못된 값을 가지고 남은 작업이 이루어진다.

**Case2. 동기화(mutex) 사용**

<p align="center"><img src="img/동기화 사용.png"></p>

- 동기화를 사용하면 공유 자원에 대한 잠금(LOCK)에 성공한 태스크가 공유 자원에 대한  해제(UNLOCK)를 하기 전까지 다른 태스크가 공유 자원에 접근 할 수 없다.
- 위 그림처럼 공유 자원에 접근할 때 잠금(LOCK)을 하기 때문에 다른 태스크가 접근할 수 없다. 만약 다른 태스크에서 접근할 경우 스케줄링을 통해 잠금이 해제(UNLOCK)되면 다시 잠금을 한 뒤 접근하게 된다.  
- Semaphore가 아닌 mutex를 사용하였기 때문에 'U'키를 눌러 강제로 잠금을 해제시켜도 해제되지 않는다. 잠금에 성공한 소유주만이 해당 잠금을 해제할 수 있다.
