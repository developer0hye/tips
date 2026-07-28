# PE 포맷 이해하기

Windows에서 `.exe`, `.dll`, `.sys`, `.ocx`, `.cpl`은 전부 같은 컨테이너 포맷을 쓴다. **PE(Portable Executable)** 다. Windows 개발을 하다 보면 "왜 이 DLL을 못 찾지?", "왜 이 주소에 로드되지?", "이 바이너리가 어떤 함수를 쓰지?", "이거 패킹된 건가?" 같은 질문을 반드시 만나는데, 이 모든 답은 PE 헤더 안에 있다. 디버거나 도구가 보여주는 정보도 결국 PE를 파싱한 결과일 뿐이므로, 포맷 자체를 알면 도구가 없거나 도구가 틀렸을 때도 직접 확인할 수 있다.

이 문서는 PE 구조를 훑고, 의존성 없는 parser를 직접 만들어 보고, 실무에서 자주 쓰는 지점(loader 동작, hooking, packing 판별)까지 연결한다.

---

## 1. 왜 알아야 하나 — 증상에서 원인으로

PE를 아는 실질적인 이득은 **"빌드는 됐는데 실행이 안 된다"류의 문제를 추측이 아니라 확인으로 해결하게 되는 것**이다. 이 계열 문제는 소스 코드를 아무리 봐도 답이 없다. 원인이 소스가 아니라 산출물과 로딩 과정에 있기 때문이다.

Windows 개발에서 실제로 만나는 증상과, 그때 PE의 어디를 보면 되는지 대응표다.

| 겪는 증상 | PE에서 확인할 것 | 절 |
|---|---|---|
| 실행 즉시 `0xC000007B` (`STATUS_INVALID_IMAGE_FORMAT`) | COFF `Machine`. 32비트 프로세스에 64비트 모듈(또는 반대)을 물린 경우가 대부분 | §2 |
| "DLL을 찾을 수 없습니다" / 내 PC에서는 되는데 배포하면 안 됨 | Import directory. 어떤 DLL·CRT를 요구하는지. **delay import는 여기 안 잡히니 따로 본다** | §4 |
| 크래시 덤프의 주소를 소스 위치로 되돌려야 함 | `RVA = VA − 모듈 로드 base`. ASLR 때문에 VA는 실행마다 바뀌지만 **RVA는 빌드마다 고정**이라 map/PDB로 함수를 특정할 수 있다 | §3 |
| 심볼(PDB)이 안 붙어서 스택이 주소로만 보인다 | Debug directory의 PDB 경로 + GUID/Age. 바이너리가 기억하는 서명과 갖고 있는 PDB가 다르면 디버거가 **조용히** 무시한다 | §4 |
| 내가 만든 DLL의 함수를 밖에서 못 찾는다 | Export directory. `dllexport`/`.def` 누락인지, C++ name mangling 때문인지 즉시 갈린다 | §4 |
| 보안 점검/납품 요건에서 "ASLR·DEP·CFG 켜라" | `DllCharacteristics` 플래그. CI에서 자동 검사할 수 있다 | §2 |
| 산출물이 이유 없이 크다 | section별 `SizeOfRawData`. 대개 `.rsrc`가 범인이다 | §2 |
| 우리 제품이 antivirus에 오탐된다 | packing으로 읽히는 신호(W+X section, 빈약한 import, 높은 entropy) | §7 |
| 서명했는데 파일 내용이 바뀌어 있다 / 설치본마다 다르다 | overlay. 서명 범위 **밖**에 데이터를 붙일 수 있다 | §7 |
| 드라이버가 특정 IRQL에서만 BSOD | `PAGE` section에 올라간 코드인지 | §2 |
| 파일·클립보드·출력 API를 가로채는 기능을 구현해야 함 | IAT / EAT / inline hooking | §5 |

부수적으로 얻는 것이 하나 더 있다. **도구가 보여주는 화면을 읽을 수 있게 된다.** Dependency Walker의 의존성 트리, Process Explorer의 모듈 목록, WinDbg의 `!dh`·`lm`, `dumpbin` 출력은 전부 PE 헤더를 그대로 늘어놓은 것이다. 필드 이름을 알면 이 도구들이 갑자기 읽히고, 반대로 도구가 없거나 결과가 의심스러울 때 직접 확인할 수 있다.

### 어디까지 알면 되나

역할에 따라 필요한 깊이가 다르다. 전부 알아야 하는 건 아니다.

- **일반 앱 개발자** — §2의 헤더 개요, §3 RVA/file offset, §4의 Import/Export. 여기까지면 위 표의 앞쪽 절반(의존성·배포·덤프 해석·심볼)이 해결된다.
- **빌드·배포·CI 담당** — 여기에 §2의 `DllCharacteristics`, §7의 서명과 overlay를 더한다. 산출물 검증을 자동화하는 데 필요한 최소 집합이다.
- **보안·저수준 개발** (드라이버, DRM/DLP, 바이너리 분석) — §5 loader 동작과 §6 직접 파싱까지 전부. hooking과 packing 판별은 이 이해 위에서만 정확해진다.

---

## 2. 전체 구조

파일을 위에서부터 순서대로 읽으면 이렇게 생겼다.

```
+---------------------------+  offset 0
| IMAGE_DOS_HEADER          |  "MZ" ... e_lfanew(0x3C) -> NT Headers 위치
+---------------------------+
| DOS Stub                  |  "This program cannot be run in DOS mode."
+---------------------------+  offset = e_lfanew
| Signature "PE\0\0"        |
| IMAGE_FILE_HEADER         |  COFF header (20 bytes)
| IMAGE_OPTIONAL_HEADER     |  이름과 달리 전혀 optional 하지 않다
|   +-- DataDirectory[16]   |  Import/Export/Reloc/... 로 가는 인덱스
+---------------------------+
| IMAGE_SECTION_HEADER * N  |  section table (각 40 bytes)
+---------------------------+
| .text  (code)             |
| .rdata (읽기 전용 + IAT)  |
| .data  (읽기/쓰기)        |
| .rsrc  (resource)         |
| .reloc (재배치 정보)      |
| ...                       |
+---------------------------+
| (overlay — 있을 수도 있음)|  마지막 section 뒤에 붙은 데이터
+---------------------------+
```

### DOS header와 DOS stub — 왜 아직도 있나

PE 파일은 1990년대 초의 하위 호환 때문에 여전히 `MZ`로 시작한다. DOS에서 실행하면 DOS stub이 돌아 "이건 DOS용이 아니다"라고 출력하고 끝난다. 오늘날 실질적으로 의미 있는 필드는 **딱 두 개**다.

- `e_magic` (offset 0): `MZ` (0x5A4D). 아니면 PE가 아니다.
- `e_lfanew` (offset **0x3C**): NT header가 있는 **file offset**.

즉 PE 파싱의 첫 두 줄은 항상 "0에서 MZ 확인 → 0x3C에서 4바이트 읽어 그리로 점프"다. `e_lfanew`는 고정값이 아니다. DOS stub 크기가 링커/컴파일러마다 달라서 0x80일 때가 많지만 **절대 가정하면 안 된다.**

### COFF file header (IMAGE_FILE_HEADER)

`PE\0\0` 4바이트 뒤에 오는 20바이트.

| 필드 | 의미 |
|---|---|
| `Machine` | CPU 아키텍처. `0x014C` = i386, `0x8664` = amd64, `0xAA64` = arm64 |
| `NumberOfSections` | section table 항목 수 |
| `TimeDateStamp` | 빌드 시각(Unix epoch). 요즘은 reproducible build 때문에 해시값을 넣기도 해서 시각으로 신뢰하면 안 된다 |
| `SizeOfOptionalHeader` | optional header 크기. **section table 위치를 계산할 때 반드시 이 값을 쓴다** |
| `Characteristics` | 파일 속성 비트. `0x0002` EXECUTABLE_IMAGE, `0x2000` DLL, `0x0020` LARGE_ADDRESS_AWARE |

`Characteristics`의 `0x2000` 비트가 EXE와 DLL을 가르는 유일한 공식 구분이다. 확장자는 아무 의미 없다 — `.dll`을 `.exe`로 바꿔도 여전히 DLL이고, 반대도 마찬가지다.

### Optional header (IMAGE_OPTIONAL_HEADER)

이름이 "optional"인 건 COFF object file에는 없기 때문이고, 실행 이미지에는 **필수**다. loader가 가장 많이 보는 헤더다.

| 필드 | 의미 |
|---|---|
| `Magic` | `0x10B` = PE32(32비트), `0x20B` = PE32+(64비트) |
| `AddressOfEntryPoint` | 진입점의 **RVA**. EXE면 CRT startup, DLL이면 `DllMain` wrapper, 드라이버면 `DriverEntry` |
| `ImageBase` | 선호 로드 주소. EXE 기본값 32비트 `0x400000` / 64비트 `0x140000000`, DLL은 `0x10000000` |
| `SectionAlignment` | 메모리에 올릴 때 정렬 단위. 보통 page 크기인 `0x1000` |
| `FileAlignment` | 파일 안에서의 정렬 단위. 보통 `0x200` |
| `SizeOfImage` | 메모리에 매핑됐을 때 총 크기(SectionAlignment 배수). loader가 이만큼 예약한다 |
| `SizeOfHeaders` | 헤더 + section table 전체 크기(FileAlignment 배수) |
| `Subsystem` | `1` NATIVE(**드라이버**), `2` WINDOWS_GUI, `3` WINDOWS_CUI(console) |
| `DllCharacteristics` | 보안 완화 기능 플래그 (아래) |
| `NumberOfRvaAndSizes` | DataDirectory 항목 수. 사실상 항상 16이지만 이 값을 쓰는 게 안전하다 |

`DllCharacteristics`는 보안 리뷰에서 제일 먼저 확인하는 값이다.

| 비트 | 이름 | 의미 |
|---|---|---|
| `0x0020` | HIGH_ENTROPY_VA | 64비트 ASLR 엔트로피 확대 |
| `0x0040` | DYNAMIC_BASE | **ASLR**. 없으면 항상 `ImageBase`에 고정 로드 |
| `0x0080` | FORCE_INTEGRITY | 로드 시 서명 검증 강제 |
| `0x0100` | NX_COMPAT | **DEP**. 데이터 영역 실행 금지 |
| `0x0400` | NO_SEH | SEH 사용 안 함 |
| `0x4000` | GUARD_CF | **CFG** (Control Flow Guard) |

**PE32와 PE32+의 차이**는 몇 개 필드의 폭뿐이다. `ImageBase`, `SizeOfStackReserve/Commit`, `SizeOfHeapReserve/Commit`이 4바이트 → 8바이트가 되고, PE32에만 있는 `BaseOfData` 필드가 PE32+에서는 사라진다. 그 결과 **DataDirectory 시작 위치가 optional header 기준 PE32는 +96, PE32+는 +112**로 달라진다. 파서를 짤 때 가장 흔히 틀리는 지점이다.

### Section table과 section

section header는 40바이트씩 이어진다.

| 필드 | 의미 |
|---|---|
| `Name[8]` | 8바이트 고정. **정확히 8글자면 null terminator가 없다** |
| `VirtualSize` | 메모리에서의 실제 크기 |
| `VirtualAddress` | 메모리에서의 위치(**RVA**) |
| `SizeOfRawData` | 파일에서의 크기(FileAlignment로 올림) |
| `PointerToRawData` | 파일에서의 위치(**file offset**) |
| `Characteristics` | 권한/속성. `0x20000000` EXECUTE, `0x40000000` READ, `0x80000000` WRITE |

관례적인 section 이름 (강제가 아니라 관례다):

| 이름 | 내용 |
|---|---|
| `.text` | 코드 (R-X) |
| `.rdata` | 읽기 전용 데이터. import/export 테이블, IAT가 보통 여기 산다 |
| `.data` | 초기화된 읽기/쓰기 데이터 (RW-) |
| `.bss` | 초기화되지 않은 데이터. `SizeOfRawData == 0`이고 메모리에서만 존재한다 |
| `.rsrc` | 아이콘, 버전 정보, 매니페스트 등 resource |
| `.reloc` | base relocation 정보 |
| `.pdata` | x64 예외 처리용 `RUNTIME_FUNCTION` 테이블 |
| `.edata` / `.idata` | export / import 테이블 (요즘 링커는 `.rdata`에 합친다) |
| `.tls` | thread local storage |

드라이버(`.sys`)에서는 `INIT`(초기화 후 버려짐, discardable), `PAGE`(pageable) 같은 이름이 **의미를 가진다.** `PAGE` section에 있는 코드는 IRQL이 DISPATCH_LEVEL 이상일 때 접근하면 페이지 폴트로 BSOD가 난다.

---

## 3. RVA, VA, file offset — 가장 중요한 개념

PE를 다룰 때 헷갈리는 주소가 세 가지다.

- **file offset**: 파일 처음부터의 바이트 위치. 파일을 `fread`로 읽을 때 쓰는 것.
- **RVA (Relative Virtual Address)**: 메모리에 매핑됐을 때, `ImageBase`로부터의 상대 offset. **PE 헤더 안의 거의 모든 주소는 RVA다.**
- **VA (Virtual Address)**: 실제 메모리 절대 주소. `VA = 실제 로드된 base + RVA`.

세 개가 다른 이유는 **파일 레이아웃과 메모리 레이아웃이 다르기 때문**이다. 파일에서는 `FileAlignment`(0x200)로, 메모리에서는 `SectionAlignment`(0x1000)로 정렬되므로 section마다 간격이 어긋난다. 그래서 헤더에 적힌 RVA를 가지고 파일 안에서 데이터를 찾으려면 반드시 변환해야 한다.

```
RVA가 속한 section을 찾는다:
    section.VirtualAddress <= RVA < section.VirtualAddress + section.VirtualSize

file_offset = RVA - section.VirtualAddress + section.PointerToRawData
```

§6에서 볼 `notepad.exe`의 section table에서 `.pdata`를 보면 명확하다.

```
.pdata   RVA 0x37000   raw 0x35000
```

`.pdata` 안의 RVA `0x37100`은 파일에서 `0x37100 - 0x37000 + 0x35000` = `0x35100`에 있다. 그냥 RVA를 file offset으로 쓰면 완전히 엉뚱한 데이터를 읽는다.

> **함정:** DataDirectory 16개 중 **`[4] Security`(Authenticode 서명)만 RVA가 아니라 file offset**이다. 서명 데이터는 section 밖(파일 끝)에 붙기 때문에 RVA로는 표현할 수가 없다. 이걸 모르고 일괄 변환하면 서명 파싱이 깨진다.

---

## 4. DataDirectory — 나머지 전부로 가는 색인

optional header 끝의 16개 `{RVA, Size}` 쌍이다. 여기가 PE의 목차 역할을 한다.

| # | 이름 | 용도 |
|---|---|---|
| 0 | Export | 이 모듈이 **내보내는** 함수 |
| 1 | Import | 이 모듈이 **가져다 쓰는** 함수 |
| 2 | Resource | 아이콘, 문자열, 버전, 매니페스트 |
| 3 | Exception | x64 unwind 정보 (`.pdata`) |
| 4 | Security | Authenticode 서명 (**file offset**) |
| 5 | BaseReloc | 재배치 정보 |
| 6 | Debug | PDB 경로/GUID (CodeView) |
| 9 | TLS | TLS 데이터 + **TLS callback** |
| 10 | LoadConfig | `/GS` 쿠키, SafeSEH, CFG 테이블 |
| 12 | IAT | Import Address Table 영역 |
| 13 | DelayImport | 지연 로드 import |
| 14 | CLR | .NET 메타데이터 (있으면 managed 바이너리) |

### Import — 어떻게 DLL 함수가 연결되는가

Import directory는 `IMAGE_IMPORT_DESCRIPTOR` 배열이고, 전부 0인 항목으로 끝난다. DLL 하나당 하나씩이다.

```c
typedef struct {
    DWORD OriginalFirstThunk;  // INT (Import Name Table)로 가는 RVA
    DWORD TimeDateStamp;
    DWORD ForwarderChain;
    DWORD Name;                // DLL 이름 문자열의 RVA
    DWORD FirstThunk;          // IAT (Import Address Table)로 가는 RVA
} IMAGE_IMPORT_DESCRIPTOR;
```

핵심은 **테이블이 두 개(INT와 IAT)이고, 파일 상태에서는 내용이 같다**는 것이다. 둘 다 "어떤 함수를 원하는지"를 가리키는 포인터 배열이다.

- **INT** (`OriginalFirstThunk`): 원본. 각 항목은 `IMAGE_IMPORT_BY_NAME`(hint 2바이트 + 이름 문자열)의 RVA. 최상위 비트가 1이면 이름이 아니라 **ordinal**로 import 한다는 뜻이다.
- **IAT** (`FirstThunk`): loader가 **덮어쓰는** 자리. 실제 함수 주소가 여기 채워진다.

로드 시 loader는 INT를 순회하며 각 함수를 찾아 그 주소를 IAT의 같은 인덱스에 **써 넣는다.** 그래서 컴파일된 코드의 함수 호출은 이렇게 생겼다.

```asm
call qword ptr [rip + 0x1234]   ; <- IAT 슬롯을 간접 호출
```

직접 `call CreateFileW`가 아니라 **IAT 슬롯을 통한 간접 호출**이다. 이 한 단계의 indirection이 PE에서 가장 실무적으로 중요한 설계다.

- **IAT hooking**이 가능한 이유가 이것이다. IAT 슬롯 하나만 자기 함수 주소로 바꾸면, 그 모듈의 모든 `CreateFileW` 호출이 내 함수로 온다. 코드를 한 바이트도 안 건드린다. DLP/DRM 제품이 파일·클립보드·출력 API를 가로챌 때 쓰는 고전적 기법이다.
- INT를 따로 남겨두는 이유도 여기 있다. IAT가 주소로 덮여도 "원래 어떤 함수였는지"는 INT에 남아 있어, 나중에 원본을 복구하거나 분석할 수 있다.

**Delay import**(directory 13)는 첫 호출 시점까지 로드를 미룬다. 실행 시작이 빨라지고 없어도 되는 DLL을 optional dependency로 만들 수 있지만, 대신 **정적 분석에서 import 목록에 안 잡힌다.** 어떤 바이너리가 무엇을 쓰는지 조사할 때 directory 1만 보고 판단하면 놓친다.

### Export — DLL이 내보내는 것

```c
typedef struct {
    ...
    DWORD Base;                   // ordinal 시작 번호 (보통 1)
    DWORD NumberOfFunctions;      // EAT 항목 수
    DWORD NumberOfNames;          // 이름이 있는 export 수
    DWORD AddressOfFunctions;     // EAT: 함수 RVA 배열
    DWORD AddressOfNames;         // 이름 문자열 RVA 배열 (ASCII 정렬됨)
    DWORD AddressOfNameOrdinals;  // 위 배열과 짝을 이루는 ordinal 인덱스 배열
} IMAGE_EXPORT_DIRECTORY;
```

`GetProcAddress("Foo")`가 하는 일이 정확히 이것이다.

1. `AddressOfNames`에서 `"Foo"`를 **binary search** 한다 (ASCII 순으로 정렬돼 있어서 가능하다).
2. 찾은 인덱스 `i`로 `AddressOfNameOrdinals[i]`를 읽어 ordinal 인덱스를 얻는다.
3. `AddressOfFunctions[ordinal]`이 함수의 RVA다.

`NumberOfFunctions`와 `NumberOfNames`가 다를 수 있다는 점에 주의한다. 이름 없이 ordinal로만 export 되는 함수가 있기 때문이다.

**Forwarded export**라는 특수 케이스가 있다. EAT의 RVA가 export directory 자기 영역 안을 가리키면, 그건 함수가 아니라 `"NTDLL.RtlAllocateHeap"` 같은 **문자열**이고 "그쪽으로 가서 찾아라"라는 뜻이다. `kernel32.dll`의 상당수가 `ntdll`이나 API set으로 forward 된다. §6의 `notepad.exe` import 목록에 `api-ms-win-core-*.dll`이 잔뜩 보이는데, 이건 디스크에 실제 파일이 없는 **API Set**이라는 가상 DLL 이름이고 loader가 실제 모듈로 해석해 준다.

### Base relocation — ASLR이 동작하는 방법

코드 안에는 절대 주소가 박혀 있을 수밖에 없다(전역 변수 주소 등). 모듈이 `ImageBase`가 아닌 곳에 로드되면 그 주소들을 전부 고쳐야 한다. `.reloc`이 "고쳐야 할 위치 목록"이다.

구조는 페이지 단위 블록의 연속이다.

```c
typedef struct {
    DWORD VirtualAddress;  // 이 블록이 담당하는 4KB 페이지의 RVA
    DWORD SizeOfBlock;     // 헤더 8바이트 포함한 블록 전체 크기
    // WORD entries[]      // 상위 4비트 = type, 하위 12비트 = 페이지 내 offset
} IMAGE_BASE_RELOCATION;
```

한 항목이 WORD 하나인 이유가 여기 있다. 페이지 크기가 4KB(=2^12)라 하위 12비트로 페이지 안의 위치를 전부 표현할 수 있고, 남는 4비트에 타입을 넣는다. 주요 타입은 `0` ABSOLUTE(정렬용 padding, 아무 것도 안 함), `3` HIGHLOW(32비트 값 보정), `10` DIR64(64비트 값 보정)다.

보정은 단순하다. `delta = 실제 로드 주소 - ImageBase`를 각 위치의 값에 더한다.

`.reloc`이 없으면 재배치가 불가능하므로 반드시 `ImageBase`에 로드돼야 하고, 그 자리가 이미 차 있으면 로드가 실패한다. **`.reloc`을 제거하는 링커 옵션(`/FIXED`)을 쓰면 ASLR이 사실상 무력화된다** — 보안 리뷰 체크 항목이다.

### TLS callback — entry point보다 먼저 실행되는 코드

TLS directory의 `AddressOfCallBacks`는 함수 포인터 배열을 가리킨다. 여기 등록된 함수들은 **`AddressOfEntryPoint`보다 먼저**, 그리고 프로세스/스레드 생성·종료 시마다 호출된다.

정상 용도는 thread-local 객체의 생성/소멸이지만, "entry point보다 먼저 돈다"는 성질 때문에 anti-debugging과 packer의 unpacking stub이 즐겨 쓴다. **entry point에만 breakpoint를 걸고 분석하면 이미 늦는다.** 악성코드나 보호된 바이너리를 볼 때 TLS directory부터 확인하는 이유다.

---

## 5. Loader는 무엇을 하는가

`.exe`를 실행하거나 `LoadLibrary`를 부르면 대략 이 순서로 진행된다.

1. **파일 매핑** — 파일을 통째로 복사하는 게 아니라 section 단위로 매핑한다. `PointerToRawData` → `VirtualAddress`로 옮겨지므로 파일 이미지와 메모리 이미지는 레이아웃이 다르다.
2. **주소 결정** — ASLR이 켜져 있거나 `ImageBase`가 이미 점유됐으면 다른 주소에 올린다.
3. **재배치 적용** — 2에서 주소가 옮겨졌으면 `.reloc`을 순회해 보정한다.
4. **import 해결** — 의존 DLL을 재귀적으로 로드하고, 각 함수 주소를 찾아 **IAT에 써 넣는다.** 실패하면 그 유명한 "DLL을 찾을 수 없습니다"다.
5. **메모리 보호 적용** — section `Characteristics`대로 페이지 권한을 건다. 이때 `.text`는 RX가 되어 쓰기 불가가 된다. inline hooking이 `VirtualProtect`를 먼저 불러야 하는 이유다.
6. **TLS callback 호출.**
7. **entry point 호출** — EXE는 CRT startup → `main`, DLL은 `DllMain(DLL_PROCESS_ATTACH)`.

이 흐름을 알고 있으면 다음이 자연스럽게 설명된다.

- **DLL이 안 찾아진다** → 4단계 실패. search order(앱 디렉터리 → System32 → PATH …)와 의존성 트리를 봐야 한다.
- **Manual mapping / reflective loading** — 1~6단계를 직접 구현하면 `LoadLibrary` 없이 메모리에서 DLL을 올릴 수 있다. 정당한 용도(플러그인 격리, 보호 로직)와 악용(injection 은닉) 양쪽에 쓰인다.
- **Hooking 세 가지 지점** — 각각 loader의 다른 단계에 대응한다.

  | 기법 | 대상 | 특징 |
  |---|---|---|
  | IAT hooking | 호출자 모듈의 IAT 슬롯 | 쉽고 안정적. 단, 동적 `GetProcAddress` 호출은 못 잡는다 |
  | EAT hooking | 대상 DLL의 export 테이블 | hook 설치 이후의 `GetProcAddress` 조회를 잡는다 |
  | Inline hooking | 함수 첫 명령어 (트램폴린) | 전부 잡지만 명령어 길이 계산이 필요하고 취약하다 |

---

## 6. 직접 파싱해 보기

의존성 없이 `struct`만으로 헤더를 읽는 최소 parser다. 위에서 설명한 순서(MZ → e_lfanew → COFF → optional → section table → RVA 변환 → import)를 그대로 코드로 옮긴 것이다.

```python
"""의존성 없이 PE 헤더를 읽는 최소 parser."""
import struct
import sys

SUBSYSTEM = {1: "NATIVE (driver)", 2: "WINDOWS_GUI", 3: "WINDOWS_CUI"}
MACHINE = {0x014C: "i386", 0x8664: "amd64", 0xAA64: "arm64"}
DIRS = [
    "Export", "Import", "Resource", "Exception", "Security", "BaseReloc",
    "Debug", "Architecture", "GlobalPtr", "TLS", "LoadConfig", "BoundImport",
    "IAT", "DelayImport", "CLR", "Reserved",
]


def main(path):
    data = open(path, "rb").read()

    # 1. DOS header -> e_lfanew (0x3C) -> NT headers
    assert data[:2] == b"MZ", "DOS signature 아님"
    nt = struct.unpack_from("<I", data, 0x3C)[0]
    assert data[nt:nt + 4] == b"PE\0\0", "PE signature 아님"

    # 2. COFF file header (NT signature 4바이트 뒤, 20바이트)
    machine, nsec, _tds, _psym, _nsym, opt_size, chars = struct.unpack_from(
        "<HHIIIHH", data, nt + 4)

    # 3. Optional header
    opt = nt + 24
    magic = struct.unpack_from("<H", data, opt)[0]
    pe32plus = magic == 0x20B
    entry = struct.unpack_from("<I", data, opt + 16)[0]
    base = struct.unpack_from("<Q" if pe32plus else "<I", data,
                              opt + 24 if pe32plus else opt + 28)[0]
    sect_align, file_align = struct.unpack_from("<II", data, opt + 32)
    size_image, size_hdrs = struct.unpack_from("<II", data, opt + 56)
    subsys, dllchars = struct.unpack_from("<HH", data, opt + 68)
    dd = opt + (112 if pe32plus else 96)   # DataDirectory 시작 오프셋
    ndd = struct.unpack_from("<I", data, dd - 4)[0]

    print(f"{path}")
    print(f"  Format        : {'PE32+' if pe32plus else 'PE32'} / "
          f"{MACHINE.get(machine, hex(machine))}")
    print(f"  Characteristics: 0x{chars:04X}"
          f"{'  [DLL]' if chars & 0x2000 else ''}")
    print(f"  ImageBase     : 0x{base:X}")
    print(f"  EntryPoint    : RVA 0x{entry:X}  (VA 0x{base + entry:X})")
    print(f"  Subsystem     : {SUBSYSTEM.get(subsys, subsys)}")
    print(f"  DllCharacteristics: 0x{dllchars:04X}  "
          f"ASLR={bool(dllchars & 0x0040)} DEP={bool(dllchars & 0x0100)} "
          f"CFG={bool(dllchars & 0x4000)}")
    print(f"  Align         : section 0x{sect_align:X} / file 0x{file_align:X}")
    print(f"  SizeOfImage   : 0x{size_image:X}   SizeOfHeaders: 0x{size_hdrs:X}")

    print("  DataDirectory :")
    for i in range(ndd):
        rva, size = struct.unpack_from("<II", data, dd + i * 8)
        if rva or size:
            # [4] Security만 RVA가 아니라 file offset이다.
            kind = "off" if i == 4 else "RVA"
            print(f"    [{i:2}] {DIRS[i]:<12} {kind} 0x{rva:<8X} size 0x{size:X}")

    # 4. Section table = optional header 바로 뒤, 40바이트 * NumberOfSections
    sec = opt + opt_size
    print(f"  Sections ({nsec}):")
    sections = []
    for i in range(nsec):
        off = sec + i * 40
        name = data[off:off + 8].rstrip(b"\0").decode("latin-1")
        vsize, vaddr, rsize, raddr = struct.unpack_from("<IIII", data, off + 8)
        flags = struct.unpack_from("<I", data, off + 36)[0]
        perm = ("R" if flags & 0x40000000 else "-") + \
               ("W" if flags & 0x80000000 else "-") + \
               ("X" if flags & 0x20000000 else "-")
        sections.append((vaddr, vsize, raddr))
        print(f"    {name:<8} RVA 0x{vaddr:<8X} vsize 0x{vsize:<8X} "
              f"raw 0x{raddr:<8X} rsize 0x{rsize:<8X} {perm} 0x{flags:08X}")

    # 5. RVA -> file offset 변환 후 import DLL 이름 나열
    def rva2off(rva):
        for vaddr, vsize, raddr in sections:
            if vaddr <= rva < vaddr + vsize:
                return rva - vaddr + raddr
        return None

    imp_rva = struct.unpack_from("<I", data, dd + 1 * 8)[0]
    if imp_rva:
        print("  Imports:")
        p = rva2off(imp_rva)
        while True:
            int_rva, _, _, name_rva, iat_rva = struct.unpack_from("<IIIII", data, p)
            if not (int_rva or name_rva or iat_rva):
                break
            n = rva2off(name_rva)
            dll = data[n:data.index(b"\0", n)].decode("latin-1")
            # INT(OriginalFirstThunk)를 걸어 함수 개수만 센다
            t = rva2off(int_rva or iat_rva)
            step = 8 if pe32plus else 4
            cnt = 0
            while struct.unpack_from("<Q" if pe32plus else "<I", data, t + cnt * step)[0]:
                cnt += 1
            print(f"    {dll:<24} {cnt} functions (IAT RVA 0x{iat_rva:X})")
            p += 20


if __name__ == "__main__":
    main(sys.argv[1])
```

`notepad.exe`에 돌려보면 이렇게 나온다 (import 목록은 줄임).

```
C:\Windows\System32\notepad.exe
  Format        : PE32+ / amd64
  Characteristics: 0x0022
  ImageBase     : 0x140000000
  EntryPoint    : RVA 0x19C0  (VA 0x1400019C0)
  Subsystem     : WINDOWS_GUI
  DllCharacteristics: 0xC160  ASLR=True DEP=True CFG=True
  Align         : section 0x1000 / file 0x1000
  SizeOfImage   : 0x5A000   SizeOfHeaders: 0x1000
  DataDirectory :
    [ 1] Import       RVA 0x309C0    size 0x3FC
    [ 2] Resource     RVA 0x3A000    size 0x1E1D0
    [ 3] Exception    RVA 0x37000    size 0x1218
    [ 5] BaseReloc    RVA 0x59000    size 0x304
    [ 6] Debug        RVA 0x2E2F0    size 0x70
    [10] LoadConfig   RVA 0x29790    size 0x148
    [12] IAT          RVA 0x298D8    size 0xB68
    [13] DelayImport  RVA 0x303E0    size 0xE0
  Sections (8):
    .text    RVA 0x1000     vsize 0x267E2    raw 0x1000     rsize 0x27000    R-X 0x60000020
    fothk    RVA 0x28000    vsize 0x1000     raw 0x28000    rsize 0x1000     R-X 0x60000020
    .rdata   RVA 0x29000    vsize 0xA6C8     raw 0x29000    rsize 0xB000     R-- 0x40000040
    .data    RVA 0x34000    vsize 0x2740     raw 0x34000    rsize 0x1000     RW- 0xC0000040
    .pdata   RVA 0x37000    vsize 0x1218     raw 0x35000    rsize 0x2000     R-- 0x40000040
    .didat   RVA 0x39000    vsize 0xF8       raw 0x37000    rsize 0x1000     RW- 0xC0000040
    .rsrc    RVA 0x3A000    vsize 0x1E1D0    raw 0x38000    rsize 0x1F000    R-- 0x40000040
    .reloc   RVA 0x59000    vsize 0x35C      raw 0x57000    rsize 0x1000     R-- 0x42000040
  Imports:
    GDI32.dll                25 functions (IAT RVA 0x29938)
    USER32.dll               87 functions (IAT RVA 0x29A08)
    api-ms-win-crt-string-l1-1-0.dll 3 functions (IAT RVA 0x2A3B8)
    ... (총 51개)
```

출력에서 읽어낼 수 있는 것들:

- `.data`의 `vsize 0x2740 > rsize 0x1000` — 초기화되지 않은 전역 변수 영역이라 파일에는 없고 메모리에서만 존재한다. 이래서 `SizeOfImage`가 파일 크기보다 클 수 있다.
- `.pdata`의 RVA(`0x37000`)와 raw(`0x35000`)가 어긋난다 — 앞서 본 RVA 변환이 필요한 이유.
- `fothk`는 표준 이름이 아니다 — Windows의 hotpatch용 thunk section이다. section 이름은 관례일 뿐이라는 실례.
- `[13] DelayImport`가 있고 `.didat` section이 존재한다 — 지연 로드 대상이 따로 있다.

**드라이버**(`tcpip.sys`)에 돌리면 성격 차이가 바로 보인다.

```
  Subsystem     : NATIVE (driver)
  DataDirectory :
    [ 0] Export       RVA 0x288000   size 0x32
    [ 4] Security     off 0x350000   size 0x25D0     <- file offset!
```

`Subsystem = 1 (NATIVE)`이고, 커널 모듈은 서명이 필수라 `[4] Security`가 채워져 있다. `notepad.exe`에는 이 항목이 없는데, Windows 시스템 바이너리는 파일 자체가 아니라 catalog 파일(`.cat`)로 서명되기 때문이다.

### 실무에서 쓰는 도구들

직접 파싱은 이해와 자동화에 좋지만, 평소에는 도구를 쓴다.

| 도구 | 용도 |
|---|---|
| `dumpbin /headers /imports /exports /disasm` | Visual Studio 동봉. CI에서 쓰기 좋다 |
| `link /dump` | `dumpbin`과 동일 (같은 바이너리다) |
| **PE-bear**, **CFF Explorer**, **PEview** | GUI로 헤더를 편집·탐색 |
| **pefile** (Python) | 스크립트 자동화의 사실상 표준 |
| **LIEF** (Python/C++) | 파싱 + **수정/재작성**까지. PE/ELF/Mach-O 공통 API |
| **Detect It Easy (DiE)** | packer/compiler 시그니처 판별 |
| `sigcheck` (Sysinternals) | 서명 검증, 버전 정보 |

---

## 7. 실무에서 자주 부딪히는 지점

### 패킹/난독화 판별

PE 헤더만 봐도 "이거 수상하다"는 신호를 상당히 잡을 수 있다.

- **`VirtualSize`가 `SizeOfRawData`보다 압도적으로 크다** — 런타임에 코드를 풀어놓을 공간을 잡아둔 것.
- **entry point가 `.text`가 아닌 section에 있다** — 정상 컴파일 산출물은 거의 항상 첫 코드 section에 진입점이 있다.
- **import가 비정상적으로 적다** (`LoadLibrary` + `GetProcAddress`만 있다시피) — 실제 API는 런타임에 동적으로 찾겠다는 뜻.
- **section에 쓰기+실행 권한이 동시에** (`0xE0000020`) — 자기 코드를 수정하겠다는 선언.
- **section 이름이 이상하다** — `UPX0`/`UPX1`, `.themida`, 또는 무작위 문자열.
- **엔트로피가 높다** (7.5 이상) — 압축/암호화된 데이터의 특징.

단, 이 신호들은 **정상 상용 소프트웨어에서도 흔하다.** 라이선스 보호, anti-tampering, installer가 같은 기법을 쓴다. 단독 근거로 악성 판정을 하면 오탐이 쏟아진다.

### Overlay

마지막 section의 `PointerToRawData + SizeOfRawData` 이후에 데이터가 더 붙어 있으면 그게 **overlay**다. PE 구조상 정의된 영역이 아니라 loader는 무시하지만, 실제로는 자주 쓰인다 — installer의 압축 페이로드, self-extracting archive, 설정 데이터, 그리고 **Authenticode 서명**이 여기 있다.

파일 크기와 `PointerToRawData + SizeOfRawData`의 최댓값을 비교하면 overlay 존재와 크기를 알 수 있다. installer를 분석할 때 정작 중요한 내용물이 여기 다 있는 경우가 많다.

### Authenticode 서명

`[4] Security`가 가리키는 곳(file offset)에 `WIN_CERTIFICATE` 구조체가 있고, 그 안에 PKCS#7 SignedData가 들어 있다. 서명 해시를 계산할 때는 **세 부분을 제외**한다.

1. optional header의 `CheckSum` 필드
2. DataDirectory의 `[4] Security` 항목 자체
3. 인증서 데이터 그 자체

당연한 이유다 — 서명을 붙이면 이 값들이 바뀌므로, 포함시키면 서명이 자기 자신을 무효화한다. 여기서 나오는 실무 함정이 있다. **파일 끝에 데이터를 붙여도 서명은 유효하게 남을 수 있다.** installer가 서명된 뒤에 설정을 append 하는 정상 패턴이기도 하고, 과거 여러 CVE의 원인이기도 하다. 서명 검증 로직을 직접 짤 땐 `WinVerifyTrust`를 쓰고, 서명 범위 밖 데이터를 신뢰하지 않도록 설계해야 한다.

### 파서를 짤 때 밟기 쉬운 지뢰

PE 파일은 **신뢰할 수 없는 입력**이다. Windows loader가 관대하게 받아주는 기형 PE가 많고, 파서를 공격하려고 일부러 조작한 파일도 있다.

- `e_lfanew`가 0x80이라고 가정하지 않는다. 파일 크기를 넘는 값일 수도 있다 (**반드시 범위 검사**).
- section table 위치는 `SizeOfOptionalHeader`로 계산한다. 구조체 크기를 하드코딩하지 않는다.
- section 이름은 8바이트 정확히 채워지면 null terminator가 없다. `strcpy` 계열 금지.
- PE32/PE32+ 분기를 놓치면 DataDirectory 위치가 16바이트 틀어져 전부 쓰레기를 읽는다.
- **모든 RVA를 변환 전에 검증한다.** 어떤 section에도 속하지 않는 RVA는 정상적으로 존재한다.
- Import descriptor 순회는 "전부 0인 항목"에서 멈추는데, **파일 끝까지 0이 안 나오면 무한 루프**가 된다. 반복 횟수와 offset 상한을 둔다.
- `NumberOfSections`, `NumberOfRvaAndSizes`를 그대로 믿고 루프를 돌리지 않는다. 조작된 큰 값이 들어올 수 있다.
- section이 서로 겹치거나, `SizeOfRawData`가 파일 크기를 넘거나, `VirtualAddress`가 정렬 안 된 PE도 실제로 로드된다.

요약하면 **모든 offset/size 산술 결과를 파일 크기와 대조하고, 모든 루프에 상한을 두는 것**이다. 위의 예제 parser는 설명을 위해 이 검증을 전부 생략했다. 프로덕션에 그대로 쓰면 안 된다.

---

## 8. 더 볼 것

- [PE Format — Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format) — 공식 스펙. 필드 정의의 최종 근거다.
- [`winnt.h`](https://learn.microsoft.com/en-us/windows/win32/api/winnt/) — `IMAGE_*` 구조체 원본 정의. 결국 이걸 보게 된다.
- Ange Albertini, [**Corkami PE poster**](https://github.com/corkami/pics/blob/master/binary/pe101/README.md) — PE 구조 한 장 요약. 책상에 붙여둘 만하다.
- [corkami/pocs](https://github.com/corkami/pocs/tree/master/PE) — 스펙의 경계를 시험하는 기형 PE 모음. 파서 테스트 케이스로 훌륭하다.
- `pefile`, `LIEF` 소스 — 실제 세계의 예외 처리를 어떻게 하는지 볼 수 있다.

---

## 한 줄 요약

> PE는 "메모리에 어떻게 올릴지"를 적어둔 설계도다. **RVA와 file offset이 다르다**는 것, 그리고 **함수 호출이 IAT를 거치는 간접 호출**이라는 두 가지만 확실히 잡으면 로딩·의존성·hooking·패킹 문제의 대부분이 설명된다.
