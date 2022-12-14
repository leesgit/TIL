## 메모리 계층 구조 Memory Hierachy 
<br>

메모리의 필요에 따라 여러 저장장치의 종류를 나눔으로써 빠르게 접근 할 수있도록 설계하는 것을 메모리 계층 구조라고 한다.

<br>

### 레지스터

Cpu 내부에 존재하는 저장장치로 프로세서가 바로 사용할 수 있는 데이터를 담고 있는 영역을 레지스터라고 한다.

주로 현재 계산에 사용중인 데이터를 저장하는데 사용된다. (CPU가 '요청을 처리하는데 필요한 데이터'를 저장하는 기억장치라고 생각하면 된다.)

### 캐시 메모리

Cpu 내부에 존재하는 레지스터 다음으로 빠른 메모리이다. 
접근 속도에 따라 L1 캐시, L2 캐시, L3 캐시 등 여러 단계로 나뉘며 메인 메모리에서 자주 사용되는 위치의 데이터를 저장하고 있어 빠르게 접근하여 데이터를 불러올 수 있다.

용량 L3>L2>L1
속도 L1>L2>L3

캐싱(Caching) : 캐시(Cache)라고 하는 빠른 메모리 영역으로 데이터를 가져와서 접근하는 방식이다.(휘발성, 흔히 사용하는 이미지 캐시를 떠올리면 된다.)

<br>

### 메인 메모리(주기억장치)

#### RAM 

사용자가 빠른 액세스를 하기 위해 데이터를 단기간 동안 저장하는 기억장치이다. 
사용자가 컴퓨터를 켠 시점부터 CPU는 연산을 하고 동작에 필요한 모든 내용이 전원이 유지되는 동안 RAM에 저장된다. 

<br>

### 보조기억장치 (SSD, HDD)

#### SSD

SSD(솔리드 스테이트 드라이브)는 기계식인 하드 디스크 드라이브(HDD)와 다르게 순수 전자식으로 작동하므로 하드 디스크의 문제점인 긴 탐색 시간, 반응 시간, 기계적 지연, 실패율, 소음을 크게 줄여 준다.

#### HDD

하드 디스크 드라이브(HDD))는 비휘발성에 순차접근이 가능한 보조 기억장치이다.

