# 시스템 콜

## 개요 및 정의
- 응용 소프트웨어는 커널 모드에 접근할 수 없다. 그렇다면 정말 아무것도 할 수 없는 것일까? 그건 아니다.
- 프로세스가 시스템의 자원이나, 서비스를 필요로 할 경우 운영체제에게 요청할 수 있다. 우리는 이것을 '시스템 콜'이라고 부른다.
- 정확히 말하면 시스템 콜 또한 인터럽트의 일종이다.

## 동작 과정
<p align="center"><img src="../images/os_system_call_process.jpg" width="600"></p>

- 응용 프로그램에서 시스템 콜을 호출하면, 0x80 인터럽트가 발생한다.
  -  일반적으로는 API 형태로 Wrapped 되어 있기 때문에 우리는 의식하지 않고 사용함.
- IDT (Interrupt Descripter Table; 인터럽트의 종류를 판단하기 위해 사용) 에서 0x80은 system_call() 이기 때문에, 커널이 시스템 콜과 관련된 동작을 수행함.

### 매개변수 전달
- 하지만 그림에서도 알 수 있듯 시스템 콜의 종류는 정말 다양하기 때문에, 어떤 시스템 콜을 수행해야 하지? 라는 질문에 답하기 위해선 추가적인 매개변수가 필요하다.
- 그렇다면 매개변수를 어떤 식으로 전달해야 할까?
- 매개변수를 레지스터에 담는다.
  - 하지만, 매개변수가 레지스터의 수보다 많은 경우가 존재하므로 적합하지 않은 방법이다.
- 매개변수를 연속적 (블록) 또는 불연속적 (테이블 사용)으로 메모리에 담은 뒤, 주소를 레지스터에 담는다.
  - 크기에 제한이 없으므로 많이 사용된다.
- 스택에 매개변수를 담는다.
  - 이 경우, 스택에 담긴 모든 데이터를 레지스터에 옮기고 스택에 원소를 담아야 하기 때문에, 결과적으로 첫 번째 방법과 큰 차이가 없다.


## 종류
### fork
> 새로운 process 생성

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    printf("pid : %d", (int) getpid()); // pid : 29146
    
    int rc = fork();					// 주목
    
    if (rc < 0) {
        exit(1);
    }									// (1) fork 실패
    else if (rc == 0) {					// (2) child 인 경우 (fork 값이 0)
        printf("child (pid : %d)", (int) getpid());
    }
    else {								// (3) parent case
        printf("parent of %d (pid : %d)", rc, (int)getpid());
    }
}
```
```
pid : 29146

parent of 29147 (pid : 29146)

child (pid : 29147)
```
- 위와 같이 출력한다. (parent와 child의 순서는 non-deterministic함. 즉, 확신할 수 없음. scheduler가 결정하는 일임.)

#### 코드 해석
- PID : 프로세스 식별자. UNIX 시스템에서는 PID는 프로세스에게 명령을 할 때 사용함.
- fork()가 실행되는 순간. 프로세스가 하나 더 생기는데, 이 때 생긴 프로세스(Child)는 fork를 만든 프로세스(Parent)와 (almost) 동일한 복사본을 갖게 된다.
- 이 때 OS는 위와 똑같은 2개의 프로그램이 동작한다고 생각하고, fork()가 return될 차례라고 생각한다.
  - 그 때문에 새로 생성된 Process (child)는 main에서 시작하지 않고, if 문부터 시작하게 된다.
- 그러나, 차이점이 있다. 바로 child와 parent의 fork() 값이 다르다는 점이다. 따라서, 완전히 동일한 복사본이라 할 수 없다.

```
Parent의 fork()값 => child의 pid 값

Child의 fork()값 => 0
```


### wait
> child 프로세스가 종료될 때까지 기다리는 작업

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    printf("pid : %d", (int) getpid()); // pid : 29146
    
    int rc = fork();					// 주목
    
    if (rc < 0) {
        exit(1);
    }									// (1) fork 실패
    else if (rc == 0) {					// (2) child 인 경우 (fork 값이 0)
        printf("child (pid : %d)", (int) getpid());
    }
    else {								// (3) parent case
        int wc = wait(NULL)				// 추가된 부분
        printf("parent of %d (wc : %d / pid : %d)", wc, rc, (int)getpid());
    }
}
```
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    printf("pid : %d", (int) getpid()); // pid : 29146
    
    int rc = fork();					// 주목
    
    if (rc < 0) {
        exit(1);
    }									// (1) fork 실패
    else if (rc == 0) {					// (2) child 인 경우 (fork 값이 0)
        printf("child (pid : %d)", (int) getpid());
    }
    else {								// (3) parent case
        int wc = wait(NULL)				// 추가된 부분
        printf("parent of %d (wc : %d / pid : %d)", wc, rc, (int)getpid());
    }
}
```
- wait를 통해서, child의 실행이 끝날 때까지 기다려준다.
- parent가 먼저 실행되더라도, wait()는 child가 끝나기 전에는 return하지 않으므로, 반드시 child가 먼저 실행됨.

### exec
> 단순 fork는 동일한 프로세스의 내용을 여러 번 동작시킬 때 사용함. child에서는 parent와 다른 동작을 하고 싶을 때는 exec를 사용할 수 있음.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    printf("pid : %d", (int) getpid()); // pid : 29146
    
    int rc = fork();					// 주목
    
    if (rc < 0) {
        exit(1);
    }									// (1) fork 실패
    else if (rc == 0) {					// (2) child 인 경우 (fork 값이 0)
        printf("child (pid : %d)", (int) getpid());
        char *myargs[3];
        myargs[0] = strdup("wc");		// 내가 실행할 파일 이름
        myargs[1] = strdup("p3.c");		// 실행할 파일에 넘겨줄 argument
        myargs[2] = NULL;				// end of array
        execvp(myarges[0], myargs);		// wc 파일 실행.
        printf("this shouldn't print out") // 실행되지 않음.
    }
    else {								// (3) parent case
        int wc = wait(NULL)				// 추가된 부분
        printf("parent of %d (wc : %d / pid : %d)", wc, rc, (int)getpid());
    }
}
```

- exec가 실행되면, execvp(실행 파일, 전달 인자) 함수는, code segment 영역에 실행 파일의 코드를 읽어와서 덮어 씌운다.
- 씌운 이후에는, heap, stack, 다른 메모리 영역이 초기화되고, OS는 그냥 실행한다.
  - 즉, 새로운 Process를 생성하지 않고, 현재 프로그램에 wc라는 파일을 실행한다.
  - 그로 인해, execvp() 이후의 부분은 실행되지 않는다.
