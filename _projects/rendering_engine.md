---
layout: project
title: 멀티 백엔드 렌더링 엔진 개발
company: 이지로보틱스
period: 2024.07 – 2025.07
tech:
  - C++
  - OpenGL
  - GLSL
  - Windows API
---

### 개요
최신 렌더링 API(DirectX 12, Vulkan 등) 도입을 목표로,
기존 OpenGL 중심 구조의 한계를 극복하기 위해 멀티 백엔드 렌더링 엔진 라이브러리를 설계 및 개발했습니다.<br>
장기적으로 다양한 그래픽 API를 유연하게 수용할 수 있는 구조를 목표로 삼았습니다.

### 요구
- Immediate Mode 기반의 레거시 OpenGL 기능으로 인해 최신 GPU 및 드라이버와의 호환성 문제가 지속적으로 보고되었으며,
이를 해결하고 장기적으로 최신 렌더링 API를 도입할 필요성이 있었습니다.<br><br>
- OpenGL API를 직접 호출하여 사용하고 있었고,
이에 대한 추상화 계층이나 상태 관리 구조가 없어 렌더링 작업 간 상태 추적이 어려워 유지보수에 문제가 있었습니다.

### 수행 업무
- Google에서 개발 중인 오픈 소스 렌더링 라이브러리 Filament의 엔진 구조를 분석하고,
이를 기반으로 멀티 백엔드 렌더링 엔진 아키텍처를 설계 및 도입
- OpenGL Core Profile 기능의 완전 지원을 1차 목표로 OpenGLDriver를 구현
- 팀원과 영역을 분담하여 레거시 OpenGL 사용 코드를 Driver 인터페이스 기반 구조로 대체하는 리팩토링 수행

### 문제
- Filament의 구조를 분석하고 이를 기반으로 엔진을 설계하는 과정에서,
Filament의 OpenGL Context 관리 방식과 자사 프로그램의 Context 관리 방식이 상이하여 그대로 적용할 수 없는 문제가 발생했습니다.
<br><br>
Filament는 주로 두 개의 OpenGL Context를 사용합니다.
하나는 메인 렌더링을 담당하는 Context이며, 다른 하나는 리소스 생성 및 해제를 담당하는 서브 Context로 구성되어
멀티 스레드 환경에서 동작하도록 설계되어 있습니다.
<br><br>
반면 자사 프로그램은 윈도우 창 하나당 하나의 OpenGL Context를 할당하여 사용하고 있었기 때문에,
OpenGLDriver가 훨씬 많은 Context 객체를 관리해야 할 필요가 있었습니다.<br><br>

### 해결방법
- 여러 개의 OpenGL Context를 지원하기 위해 Context, HDC, SwapChain 개념을 하나의 단위로 결합했습니다.<br><br>
모든 윈도우 창은 하나의 Context를 가지며, 각 Context는 하나의 백버퍼를 가지므로
이는 의미적으로 창 하나당 하나의 SwapChain을 갖는 구조로 해석했습니다.<br><br>
- OpenGLPlatform 객체에서 모든 Context, HDC, SwapChain을 통합 관리하도록 설계하고,
생성 및 삭제 과정 또한 OpenGLPlatform을 통해서만 이루어지도록 구성했습니다.<br><br>
- OpenGL의 Context 간 리소스 공유 기능을 활용하여,
새로운 Context 생성 시 내부에서 관리하는 Default OpenGL Context를 공유 Context로 연결함으로써
여러 Context 간 리소스가 원활히 공유되도록 했습니다.<br>
이 Default Context를 통해 Context 전환 시 OpenGL Context 바인딩이 무효화되지 않도록 보장했습니다.<br><br>
- OpenGLDriver는 OpenGLPlatform이 관리하는 Context 객체에 대한 참조만 유지하도록 하여,
Context 생명주기 관리는 Platform이 담당하고,
Driver는 OpenGL API 호출 및 렌더링 작업에만 집중하도록 책임을 분리했습니다.

### 성과
- OpenGL API와 상위 애플리케이션 간 의존성을 제거하여 렌더링 코드의 구조적 복잡도를 크게 낮춤
- 멀티 백엔드 확장을 전제로 한 엔진 구조를 확립하여, 향후 DirectX 12 및 Vulkan 도입을 위한 기반 마련
- 렌더링 API 교체 시 기존 상위 코드의 수정 없이 Driver 교체만으로 대응 가능하도록 설계

### 향후 계획
- Driver 구조를 기반으로 DirectX 12 혹은 Vulkan Driver 구현 예정
- Command Buffer 기반 렌더링 파이프라인으로의 확장 검토 및 메인/렌더 스레드 분리

### 사용 기술
- C++20
- OpenGL
- GLSL
- 자체 RHI 추상화 계층
- Windows API