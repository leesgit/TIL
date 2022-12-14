## 데드락 (DeadLock, 교착 상태)

두개 이상의 프로세스나 스레드가 서로 자원을 얻지 못하여 다음일 처리 하지 못하고 무한히 대기하는 상태를 말한다.
시스템적으로 한정된 자원을 여러 사용자가 사용하려고 하여 발생하는 교착 상태이다.(외나무 다리에서 서로 비켜주기만을 기다리는 상황과 유사하다.)

<br>

### 데드락이 발생하는경우 경우

처리해야될 자원1, 2가 있고 둘 모두를 처리해야되는 프로세스 1, 2가 있다고 가정해보자

1. 프로세스1이 자원 1를 차지한다.

2. 프로세스2가 자원 2를 차지한다.

3. 프로세스1이 자원 2를 차지하려고 하지만 프로세스2가 이미 자원을 차지하고 있어 대기 상태에 들어간다.

4. 프로세스2 또한 자원 1을 차지하려고 하지만 프로스세스1이 이미 자원을 차지하고 있어 대기 상태에 들어가게 된다.

5. 프로세스 1, 2가 서로의 작업이 끝나길 기다리면 무한 대기상태(교착상태)에 빠지게 된다.


### 데드락의 발생 조건

4가지 조건 모두가 만족할 경우에만 데드락이 발생한다.


1. #### 상호 배제(Mutual exclusion)

하나의 자원에 하나의 프로세스만 접근 가능한 경우이다.

2. #### 점유 대기(Hold and wait)

프로세스가 자원을 가지고 있는 상태에서 다른 프로세스가 자원을 가지기 위해 대기 해야 하는 경우이다.

3. #### 비선점(선점 불가)(No preemption)

하나의 프로세스가 자원을 가지고 있을 경우 다른 프로세스가 강제로 자원을 가져오지 못하는 경우이다.

4. #### 순환 대기(Circular wait)

프로세스 1, 2가 서로 자원을 가지고 있는 상태에서 다른 자원을 가지기 위해 대기하는 경우이다.(구지 1,2가 아니여도 여러 프로세스 들이 순환을 대기하는 경우)

<br>

### 데드락(DeadLock) 처리

#### 교착 상태를 예방 & 회피

1. ##### 예방(prevention)

   교착 상태의 4가지 발생 조건 중 하나 이상을 해결하면 피할 수 있다.

    상호배제 예방 : 여러 프로세스가 자원을 공유하여 사용할 수 있게 한다. (트랜잭션과 같은 자원의 동기화 문제가 발생할 수 있다.)
    점유대기 예방 : 프로세스들이 모든 자원을 처리하기 까지 기다리고 다시 자원을 처리하는 것을 말한다.
    비선점 예방 : 프로세스에 우선순위를 두어 따라 자원을 선점하게 한다.
    순환대기 예방 : 방향이나 순서등을 정하여 한쪽 순서나 방향에서만 자원을 요구할 수 있게 한다. (1, 2 자원을 사용할 경우 1번 이용 후 2번을 사용하게 하는 등)

2. ##### 회피(avoidance)

데드락이 빠질 가능성이 있는지 없는지를 미리 검사하여 빠질 가능성이 없을 경우에만 자원을 하는식으로 문제 발생을 피하는 방법을 말한다. (교착 상태를 허용하지 않는다.)


#### 은행원 알고리즘(Banker's Algorithm)
   
안정상태를 유지할 수 있는 프로세스에만 자원만 할당하고 불안정상태라면 다른 프로세스가 자원의 사용을 종료할 때 까지 대기하는 방법이다.

1. ##### 안정 상태

자원을 할당할 때 교착 상태를 일으키지 않으면서 프로세스가 요구한 만큼 자원을 할당할 수 있는 상태를 말한다. (안전한 상태인지 확인하는 부분에서 오버헤드가 발생하다.)

2. ##### 불안정 상태

프로세스에 자원을 할당하게 될 경우 교착 상태가 발생하는 경우을 말한다. 
   

#### 교착 상태를 탐지 & 회복

교착 상태 일으킨 프로세스를 탐색하여 종료등을 통하여 순환 대기를 회복하는 방법을 말한다. (교착 상태는 허용 이를 탐지하고 회복한다.)

1. ##### 탐지(Detection)

   자원 할당 그래프를 통해 교착 상태를 탐지한다.
   
   탐지 알고리즘을 실행시키기 때문에 오버헤드 발생한다.

2. ##### 회복(Recovery)

   탐지에서 발견된 순환 대기 부분을 아래와 같은 방법으로 없애는 것을 말한다.
    1. 순환 대기가 없어질 때 까지 프로세스를 종료한다.
    2. 순환 대기 중인 프로세스의 제어권을 뺏는다.
