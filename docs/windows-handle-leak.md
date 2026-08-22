# Windows 핸들 누수 — 메모리는 멀쩡한데 프로세스가 죽는다

며칠씩 켜져 있는 Windows 프로세스가 어느 날 이유 없이 죽는다. 메모리를 봐도 정상이고, 프로파일러도 조용하고, 크래시 지점은 방금 고친 코드와 아무 상관이 없다. 이 조합이 나오면 **핸들 누수**를 의심해야 한다.

핸들 누수는 메모리 누수와 성질이 다르다. 새는 양이 작아서 눈에 안 띄고, 기존 도구가 못 잡고, 증상이 원인에서 멀리 떨어진 곳에서 나타난다. 대신 원리를 알면 관측과 증명이 오히려 메모리 누수보다 쉽다 — 숫자가 정수로 딱 떨어지기 때문이다.

이 문서는 핸들이 무엇인지부터 시작해서, 왜 기존 도구에 안 잡히는지, 어떻게 관측하고 증명하는지, 그리고 애초에 안 새게 짜는 법까지 다룬다. 실제 사례로는 OpenCV의 MSMF 카메라 백엔드에서 잡은 Event 핸들 누수를 쓴다.

---

## 1. 왜 알아야 하나 — 증상에서 원인으로

핸들 누수를 아는 실질적인 이득은 **"오래 돌리면 죽는데 재현이 안 된다"류의 문제를 추측이 아니라 확인으로 해결하게 되는 것**이다. 이 계열 문제는 크래시 지점을 아무리 파도 답이 없다. 원인이 그곳이 아니기 때문이다.

| 겪는 증상 | 의심할 것 | 절 |
|---|---|---|
| 며칠 돌리면 죽는데, 재시작하면 멀쩡하고 재현이 안 된다 | 프로세스 수명에 비례해 쌓이는 자원. 핸들 누수의 가장 전형적인 얼굴 | §4 |
| 메모리는 정상인데 프로세스가 불안정하다 | 핸들은 힙이 아니다. 메모리 그래프에 안 나타난다 | §3 |
| 멀쩡하던 `CreateFile`·`CreateThread`가 갑자기 실패한다 | 핸들 고갈. `ERROR_NO_SYSTEM_RESOURCES`, `ERROR_NOT_ENOUGH_MEMORY` | §4 |
| ASAN·CRT 디버그 힙·프로파일러가 전부 깨끗하다고 한다 | 이 도구들은 유저 힙만 본다. 커널 객체는 관측 범위 밖 | §3 |
| 다 썼는데 파일이 안 지워진다 / 장치가 안 놓인다 | 핸들이 살아 있으면 커널 객체도 안 죽는다 | §4 |
| UI가 오래 쓰면 깨지거나 창이 안 열린다 | GDI/USER 객체 한도(기본 10,000). 커널 핸들과 **별도 쿼터** | §4 |
| 라이브러리를 반복 open/close 하는 코드에서만 그렇다 | 라이브러리의 정리 경로. 실제로 흔하다 | §6 |
| 리소스 누수를 고쳤는데 정말 고쳐졌는지 증명해야 한다 | build-matched A/B + 워밍업 제외 | §5 |

## 2. 핸들이 무엇인가

핸들은 **포인터가 아니라 번호표**다. 여기서 출발하면 나머지가 전부 따라온다.

`CreateEvent`를 부르면 실제 Event 객체는 **커널 주소 공간**에 만들어진다. 내 프로세스 메모리가 아니다. 나한테 돌아오는 `HANDLE`은 그 객체를 가리키는 번호 — 정확히는 프로세스마다 하나씩 있는 **핸들 테이블의 인덱스**다.

```
  유저 모드 (내 프로세스)          |  커널 모드
                                   |
  HANDLE h = 0x1A4  ───────────┐   |
                               │   |
  +------------------------+   │   |   +------------------------+
  |  프로세스 핸들 테이블   |   │   |   |  Event 객체            |
  |  ...                   |   └───┼──>|  RefCount: 1           |
  |  0x1A4 -> [객체 포인터] |───────┼──>|  signaled: false       |
  |  ...                   |       |   +------------------------+
  +------------------------+       |    nonpaged pool
                                   |
  CloseHandle(h)                   |    RefCount 0 -> 커널이 해제
```

핵심은 두 가지다.

- **소유권이 명시적이다.** 커널 객체는 참조 카운트를 들고 있다. 핸들을 열면 +1, `CloseHandle`하면 -1. 0이 되어야 커널이 객체를 해제한다.
- **자동 회수가 없다.** C++ 소멸자도, GC도, allocator도 이 카운트를 안 건드린다. `CloseHandle`을 직접 부르는 것 외에 방법이 없다.

> 코트를 맡기고 번호표를 받는 것과 같다. 코트는 창고에 있고 내 손엔 종이 한 장뿐이다. 번호표를 안 내면 코트는 창고에 영원히 남는다. **번호표가 가볍다는 게 문제를 가리는 지점이다.**

### 핸들로 다루는 것들

Event만이 아니다. Windows에서 커널이 관리하는 것은 대부분 핸들로 온다.

`File` `Event` `Mutant`(뮤텍스) `Semaphore` `Thread` `Process` `Section`(파일 매핑) `Key`(레지스트리) `Token` `Timer` `Job` `IoCompletion` `WaitCompletionPacket` `Directory` `ALPC Port`

Process Explorer에서 타입 이름으로 그대로 보인다. **어떤 타입이 느는지가 범인을 지목하는 정보**다. Event가 늘면 동기화 코드를, File이 늘면 I/O 경로를, Thread가 늘면 스레드 생성 코드를 보면 된다.

## 3. 메모리 누수와 무엇이 다른가

| | 메모리 누수 | 핸들 누수 |
|---|---|---|
| 새는 것 | 내 프로세스 힙 바이트 | 커널 객체 + 테이블 한 칸 |
| 사는 곳 | 유저 주소 공간 | 커널 주소 공간 (nonpaged pool) |
| 크기 | 새는 만큼 그대로 | 객체당 수십~수백 바이트 |
| 메모리 그래프 | 우상향으로 보임 | **미동도 없음** |
| ASAN / CRT 디버그 힙 | 잡음 | **못 잡음** |
| 회수 수단 | GC, allocator, RAII | `CloseHandle` 뿐 |
| 프로세스 종료 시 | 어차피 회수 | 어차피 회수 |

실무에서 중요한 줄은 **"메모리 그래프에 미동도 없음"** 이다. 리소스 누수를 의심할 때 사람들이 가장 먼저 보는 게 작업 관리자의 메모리 열인데, 핸들 누수는 여기서 정확히 아무 신호도 주지 않는다. Event 하나가 수백 바이트라 10,000개를 새도 몇 MB다. 그래서 **"메모리는 정상이니 누수는 아니다"라는 결론**으로 조사가 끝나 버린다.

핸들 누수가 오래 살아남는 이유가 이 한 줄에 다 있다.

## 4. 새면 무슨 일이 일어나는가

### 한도까지는 안 간다

프로세스당 핸들 상한은 이론상 16,777,216개(2²⁴)다. 이 숫자까지 갈 일은 없다. **훨씬 먼저 다른 데서 터진다.**

핸들 테이블과 커널 객체는 **nonpaged pool**을 쓴다. 페이지 아웃되지 않는, 물리 메모리에 고정된 영역이다. 여기가 압박받으면 증상이 **엉뚱한 곳에서** 난다.

- 멀쩡하던 `CreateFile`이 `ERROR_NO_SYSTEM_RESOURCES`(1450)로 실패
- `CreateThread` 실패 → 스레드 풀이 굶음
- 원인 코드와 아무 상관없는 모듈이 죽음
- 심하면 시스템 전체가 느려짐 (nonpaged pool은 머신 공용이다)

디버깅이 어려운 이유가 여기 있다. **크래시 스택에 범인이 안 나온다.** 카메라 코드가 핸들을 샜는데 죽는 건 로그 모듈이다.

### 객체가 안 죽는다

크기보다 이쪽이 더 문제인 경우가 많다. 핸들이 살아 있으면 그 커널 객체도 참조 카운트가 0이 안 되므로 **못 죽는다.**

- 파일 핸들 → 그 파일이 안 지워진다. 업데이터가 실패한다
- 장치 핸들 → 장치가 안 놓인다. 다른 프로세스가 카메라를 못 연다
- 프로세스 핸들 → **좀비 프로세스**. 종료됐는데 목록에 남는다
- Section 핸들 → 매핑된 메모리가 안 풀린다

"파일이 다른 프로세스에서 사용 중입니다"의 상당수가 이거다.

### GDI/USER는 별도 쿼터다

혼동하기 쉬운 지점이다. `CreatePen`, `CreateBrush`, `CreateDC`, `CreateBitmap` 같은 GDI 객체와 윈도우·메뉴 같은 USER 객체는 **커널 핸들과 다른 쿼터**를 쓴다.

| 종류 | 프로세스당 기본 한도 |
|---|---|
| 커널 핸들 | 16,777,216 |
| GDI 객체 | **10,000** |
| USER 객체 | **10,000** |

10,000은 실제로 도달하는 숫자다. 리스트를 그릴 때마다 브러시를 만들고 `DeleteObject`를 안 부르면 하루 만에 닿는다. 증상은 **"UI가 갑자기 안 그려진다"**, "창이 안 열린다"로 나온다 — 크래시가 아니라 렌더링 실패라 더 헷갈린다. 작업 관리자에 GDI 개체, USER 개체 열을 따로 추가할 수 있다.

### 프로세스가 끝나면 다 회수된다

Windows가 종료 시 핸들 테이블을 통째로 정리한다. 그래서 **핸들 누수는 프로세스 수명을 못 넘긴다.** 30초 돌고 끝나는 스크립트는 100개를 새도 아무 일 없다.

바꿔 말하면 이건 **장기 실행 프로세스만의 병**이다.

- 서비스, 백그라운드 에이전트, 키오스크
- 서버 프로세스
- 며칠씩 띄워두는 데스크톱 앱

CI에서도, 단위 테스트에서도, 개발자 로컬에서도 안 잡힌다. 고객사에서만 잡힌다. 리포트가 잘 안 올라오고 올라와도 재현이 안 되는 이유다.

## 5. 어떻게 관측하나

총 핸들 수가 느는 건 **증상**이고, 범인을 지목하려면 **객체 타입별로 쪼개야** 한다. 도구를 이 순서로 올린다.

| 도구 | 얻는 것 | 쓸 때 |
|---|---|---|
| 작업 관리자 (핸들 열 추가) | 총계만 | 누수 여부 1차 확인 |
| **Process Explorer** | **타입별 분해** | 범인 타입 지목. 여기서 대부분 끝난다 |
| `GetProcessHandleCount()` | 총계, 코드 안에서 | 자동화된 A/B 측정 |
| `NtQuerySystemInformation`<br>(`SystemExtendedHandleInformation`) | 타입별, 코드 안에서 | 회귀 테스트에 넣을 때 |
| WinDbg `!htrace` | **누가 열었는지 스택** | 타입은 알는데 코드를 못 찾을 때 |
| Application Verifier (Handles) | 잘못된 핸들 사용·이중 close | 개발 중 상시 |

`!htrace`는 추적을 먼저 켜야 한다.

```
!htrace -enable        # 추적 시작
  ... 재현 시나리오 실행 ...
!htrace -diff          # 그 사이 열리고 안 닫힌 핸들 + 생성 스택
```

### 코드 안에서 세기

A/B를 자동화하려면 이게 가장 간단하다.

```cpp
#include <windows.h>
#include <cstdio>

DWORD handle_count()
{
    DWORD n = 0;
    GetProcessHandleCount(GetCurrentProcess(), &n);
    return n;
}

int main()
{
    for (int i = 0; i < 100; ++i)
    {
        do_one_cycle();                       // 의심되는 open/close 한 사이클
        printf("cycle %3d: handles=%lu\n", i, handle_count());
    }
}
```

### 측정에서 반드시 지킬 것

숫자를 뽑는 것보다 **그 숫자가 증거가 되게 만드는 것**이 어렵다. 세 가지를 지켜야 한다.

1. **워밍업 구간을 버린다.** 라이브러리 초기화, 스레드 풀 생성, 지연 로딩된 DLL 때문에 초반 몇 사이클은 핸들이 정상적으로 는다. 이건 누수가 아니다. **정상 상태에 들어간 뒤의 기울기**를 봐야 한다.
2. **build-matched A/B로 한다.** 컴파일러, 빌드 옵션, 대상 장치, 워크로드, 실행 파일까지 전부 고정하고 **패치 한 줄만** 다르게 한 두 빌드를 비교한다. 버전이 다른 두 빌드를 비교하면 무엇이 원인인지 말할 수 없다.
3. **"0"이 아니라 "기울기 0"을 본다.** 핸들 수는 정상 코드에서도 사이클마다 ±1~2 출렁인다. 캐시, 지연 정리, 다른 스레드 때문이다. 증명해야 하는 건 절대값이 아니라 **선형 증가가 사라졌다는 것**이다. 사이클을 충분히(50~100) 돌려 밴드가 일정하게 유지되는지 본다.

## 6. 실제 사례 — OpenCV MSMF의 Event 핸들 누수

교과서 사례라 그대로 옮긴다. ([opencv/opencv#29563](https://github.com/opencv/opencv/pull/29563), OpenCV 4.15.0에 포함)

Windows에서 OpenCV의 기본 카메라 백엔드는 **MSMF(Microsoft Media Foundation)** 다. `cv::VideoCapture(0)`은 백엔드를 명시하지 않으면 이 경로로 간다.

MSMF의 `IMFSourceReader`는 **비동기**다. `ReadSample()`은 즉시 반환하고, 프레임이 준비되면 나중에 다른 스레드에서 콜백이 불린다. 그런데 OpenCV가 노출하는 `cap.read(frame)`은 **동기·블로킹** API다. 그 간극을 메우려고 `SourceReaderCB`가 Win32 Event를 하나 만든다.

```cpp
SourceReaderCB() :
    m_nRefCount(0),
    m_hEvent(CreateEvent(NULL, FALSE, FALSE, NULL)),   // 핸들 생성
    ...
```

`read()` 쪽 스레드가 이 이벤트에서 대기하고, 콜백이 프레임을 받으면 `SetEvent`으로 깨우는 구조다. 그리고 소멸자는 이랬다.

```cpp
virtual ~SourceReaderCB()
{
    CV_LOG_INFO(NULL, "terminating async callback");   // 로그만 찍는다
}
```

`SourceReaderCB`는 COM 참조 카운트 객체라 **소멸자는 정상적으로 불렸다.** 객체 자체는 새지 않았다. 다만 그 안에서 커널 핸들을 안 닫았을 뿐이다.

### 왜 오래 안 잡혔나

§3의 표가 그대로 답이다.

- Event 객체가 작아서 **메모리 그래프가 미동도 안 했다**
- 힙이 아니라 **누수 탐지 도구 범위 밖**이었다
- 카메라를 한 번 열고 끝내는 일반적인 사용에서는 **아무 일도 안 일어난다**
- 카메라를 주기적으로 재연결하는 장기 실행 프로세스에서만 드러나는데, 그런 환경은 재현 리포트를 만들기 어렵다

같은 코드가 4.12.0, 4.13.0, 당시 `4.x`, 5.0.0 태그에 모두 있었다.

### 증명

§5의 세 규칙을 적용한 A/B다. 컴파일러·CMake 옵션·카메라·워크로드·실행 파일 동일, 차이는 패치 한 줄.

```cpp
for (int i = 0; i < 100; ++i)
{
    cv::VideoCapture cap;
    if (!cap.open(0, cv::CAP_MSMF)) return 1;

    cv::Mat frame;
    cap.read(frame);
    cap.release();
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
}
```

| 빌드 | Event 핸들, 사이클 4→10 | 전체 프로세스 핸들, 4→10 |
|---|---:|---:|
| Unpatched | **+7** | +20 |
| Patched | **0** | +12 |

사이클 1~3을 버린 게 워밍업 제외다. 패치된 빌드는 100/100 사이클을 완주했고, 51~100 구간에서 Event 핸들 밴드 1, 전체 핸들 밴드 2로 **선형 증가가 사라졌다.**

전체 핸들이 패치 후에도 +12인 데 주목할 것. 이건 다른 정상적인 초기화들이다. **총계만 봤으면 아무것도 증명 못 했다.** 타입별로 쪼갰기 때문에 Event 열의 7 → 0이 보였다.

### 패치

```cpp
 virtual ~SourceReaderCB()
 {
+    if (m_hEvent)
+        CloseHandle(m_hEvent);
     CV_LOG_INFO(NULL, "terminating async callback");
 }
```

핸들을 만든 것도 쓴 것도 `SourceReaderCB` 하나뿐이니, **소유권 경계가 정확히 이 소멸자**였다. 두 줄이다. 어려운 건 두 줄이라고 말할 수 있게 되기까지다.

## 7. 애초에 안 새게 짜는 법

### RAII로 감싼다 — 손으로 닫지 않는다

`CloseHandle`을 손으로 부르는 코드는 언젠가 샌다. early return, 예외, 조건 분기 중 하나가 반드시 빠진다.

```cpp
// 나쁨 — 중간에 return 하나만 추가되면 샌다
HANDLE h = CreateEvent(NULL, FALSE, FALSE, NULL);
if (!do_something()) return false;    // 여기서 샘
CloseHandle(h);

// 좋음 — WIL (Windows Implementation Library)
#include <wil/resource.h>
wil::unique_handle h(CreateEvent(NULL, FALSE, FALSE, NULL));
wil::unique_event e;  e.create();     // 더 직접적

// 좋음 — ATL
#include <atlbase.h>
ATL::CHandle h(CreateEvent(NULL, FALSE, FALSE, NULL));

// 의존성 없이 — unique_ptr + custom deleter
struct HandleDeleter {
    using pointer = HANDLE;
    void operator()(HANDLE h) const { if (h && h != INVALID_HANDLE_VALUE) CloseHandle(h); }
};
using unique_handle = std::unique_ptr<HANDLE, HandleDeleter>;
```

### 실패값이 API마다 다르다 — 가장 흔한 함정

이걸 틀리면 RAII 래퍼까지 같이 망가진다.

| 함수 | 실패 시 반환 |
|---|---|
| `CreateFile` | `INVALID_HANDLE_VALUE` (**-1**) |
| `CreateEvent`, `CreateMutex`, `CreateSemaphore` | `NULL` |
| `OpenProcess`, `OpenEvent` | `NULL` |
| `CreateThread` | `NULL` |
| `CreateToolhelp32Snapshot` | `INVALID_HANDLE_VALUE` |
| `CreateFileMapping` | `NULL` (성공해도 `INVALID_HANDLE_VALUE` 아님) |

`if (h == INVALID_HANDLE_VALUE)`로 `CreateEvent`를 검사하면 **실패한 NULL을 성공으로 취급**한다. 반대도 마찬가지다. 래퍼를 쓸 때도 어느 쪽 무효값을 쓰는 타입인지 봐야 한다(WIL은 `unique_handle`과 `unique_hfile`을 구분한다).

### CloseHandle이 아닌 것들

전부 `CloseHandle`이 아니다. 잘못 부르면 안 닫히거나 손상된다.

| 만든 것 | 닫는 함수 |
|---|---|
| `CreateFile`, `CreateEvent`, `CreateThread`, `CreateFileMapping` | `CloseHandle` |
| `FindFirstFile` | **`FindClose`** |
| `RegOpenKeyEx`, `RegCreateKeyEx` | **`RegCloseKey`** |
| `CreatePen`, `CreateBrush`, `CreateBitmap` (GDI) | **`DeleteObject`** |
| `CreateDC` | **`DeleteDC`** / `GetDC`는 **`ReleaseDC`** |
| `CreateWindow` | **`DestroyWindow`** |
| `socket` | **`closesocket`** |
| `GetCurrentProcess()`, `GetCurrentThread()` | **닫지 않는다** (pseudo-handle) |
| `GetStdHandle` | **닫지 않는다** |

### 소유권 경계를 한 곳으로 정한다

MSMF 사례의 교훈이 정확히 이거다. 핸들을 만든 곳과 닫는 곳이 **같은 객체**여야 한다.

- 생성자에서 만들었으면 소멸자에서 닫는다
- 함수가 핸들을 반환하면 **누가 닫는지를 이름이나 주석으로 명시**한다
- 핸들을 받아만 쓰는 함수는 절대 닫지 않는다
- COM 참조 카운트 객체 안의 핸들도 예외가 아니다. 객체가 회수돼도 핸들은 안 닫힌다

### 이중 close를 조심한다

닫은 핸들을 또 닫으면 그냥 실패로 끝나지 않는다. 그 사이 같은 값이 **다른 객체에 재사용**됐으면 남의 핸들을 닫는다. 원인과 한참 떨어진 곳에서 나는 버그가 된다. 디버거가 붙어 있으면 `STATUS_INVALID_HANDLE` 예외가 뜬다. RAII를 쓰면 대부분 자동으로 방지된다.

### 개발 중에 상시로 켜둘 것

- **Application Verifier**의 Handles 검사 — 잘못된 핸들 사용과 이중 close를 즉시 예외로 만든다
- **장기 실행 시나리오를 테스트에 넣는다.** open/close를 100회 돌리고 핸들 기울기를 확인하는 테스트 하나면, 이 계열 버그 대부분이 개발 단계에서 잡힌다

## 8. 체크리스트

의심될 때 이 순서로 간다.

1. 작업 관리자에 **핸들** 열을 추가한다. 시간에 따라 느는가?
2. **GDI 개체 / USER 개체** 열도 본다. 쿼터가 다르다 (§4)
3. **Process Explorer**로 어떤 **타입**이 느는지 본다. 여기서 볼 코드가 정해진다
4. 재현 시나리오를 100 사이클 돌린다. **워밍업 몇 사이클은 버린다**
5. 여전히 코드를 못 찾으면 WinDbg `!htrace -enable` → 재현 → `!htrace -diff`로 **생성 스택**을 본다
6. 고친 뒤에는 **build-matched A/B**로 확인한다. 패치 한 줄만 다른 두 빌드
7. 판정 기준은 절대값 0이 아니라 **기울기 0**이다

## 한 줄 요약

> 핸들은 커널이 들고 있는 자원의 번호표이고, `CloseHandle` 외에 회수하는 주체가 없다. 메모리 그래프에도 누수 탐지 도구에도 안 잡히므로 **총 핸들 수를 타입별로 보는 것**이 유일한 관측 수단이며, 프로세스 종료로 회수되기 때문에 **오래 사는 프로세스에서만** 문제가 된다.

## 참고

- [Handles and Objects — Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/sysinfo/handles-and-objects)
- [Process Explorer](https://learn.microsoft.com/en-us/sysinternals/downloads/process-explorer)
- [Application Verifier](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/application-verifier)
- [WIL — Windows Implementation Library](https://github.com/microsoft/wil)
- [opencv/opencv#29563 — videoio(MSMF): close leaked SourceReaderCB Event handle](https://github.com/opencv/opencv/pull/29563)
