---
layout: project
title: Material 기반 렌더링 구조 도입
company: 이지로보틱스
period: 2025.03 – 2025.05
tech:
  - C++
  - OpenGL
  - GLSL
  - Windows API
---

### 개요
전역으로 생성·관리되던 Shader–ShaderManager 구조를
Material–MaterialInstance 구조로 전환하여
렌더링 파이프라인 상태 관리성을 향상시키고, 전반적인 파이프라인 구조를 개선했습니다.
<br><br>

### 요구
- 기존 Shader는 쉐이더 프로그램 바인딩과 uniform 변수 스트리밍만을 담당하고,
파이프라인 상태(Depth, Blend 등)는 외부에서 직접 설정해야 하는 구조였습니다.<br>
이로 인해 렌더링 코드가 상태 설정 순서에 강하게 의존하게 되었고,
렌더 패스 간 상태 추적이 어려워지는 문제가 발생했습니다.<br><br>
최신 렌더링 API에서는 쉐이더 프로그램과 파이프라인 상태가 긴밀하게 결합되어 동작하므로,
분리되어 있던 두 요소를 하나로 통합하는 것은 최신 API 도입을 위한 필수 과제였습니다.
<br><br>
- 모든 Shader가 싱글턴 객체인 ShaderManager에 의해 전역으로 관리되면서,
다른 OpenGL Context 혹은 다른 Scene에서 Shader를 사용할 경우
이전 렌더링에서 설정된 상태가 이후 렌더링에 영향을 주는 문제가 반복적으로 발생해
유지보수성에 어려움이 있었습니다.
<br><br>

### 수행 업무
- Filament의 Material–MaterialInstance 구조를 분석하고 이를 기반으로 구조를 구현·적용
- ShaderManager를 대체하는 Material 관리 객체 개발
- Shader 프로그램과 MaterialPackage를 통합 관리하는 기능 개발
<br><br>

### 문제
- Filament의 Material–MaterialInstance 구조 자체는 유의미했으나,
  쉐이더 코드와 Material Package를 관리하는 방식까지 그대로 도입하기에는
  빌드 파이프라인 복잡도와 신규 기능 추가 비용이 과도하다는 문제가 있었습니다.
<br><br>
Filament는 쉐이더와 Material을 아래 그림과 같은 구조로 관리합니다.

<div style="text-align: center; margin: 20px 0;">
  <img src="{{ '/assets/images/filament shader & material.png' | relative_url }}" 
       alt="Filament Material Management Structure" 
       style="width: 100%; max-width: 600px; border-radius: 8px; border: 1px solid #ddd;">
  <p style="font-size: 0.8em; color: #888;">[그림 1] Filament의 Shader 및 Material 관리 구조</p>
</div>

- 범용적으로 사용되는 구조체나 조명 계산에 사용되는 BRDF 함수 등을
CMake 기반 컴파일을 통해 라이브러리로 생성하고,
각 GLSL 쉐이더 파일을 shaders.h / shaders.c 형태로 묶어 제공합니다.

<div style="text-align: center; margin: 20px 0;">
  <img src="{{ '/assets/images/filament shader 헤더.png' | relative_url }}" 
       alt="Filament Material Management Structure" 
       style="width: 100%; max-width: 600px; border-radius: 8px; border: 1px solid #ddd;">
  <p style="font-size: 0.8em; color: #888;">[그림 2] shaders.h</p>
</div>
<div style="text-align: center; margin: 20px 0;">
  <img src="{{ '/assets/images/filament shader c.png' | relative_url }}" 
       alt="Filament Material Management Structure" 
       style="width: 100%; max-width: 600px; border-radius: 8px; border: 1px solid #ddd;">
  <p style="font-size: 0.8em; color: #888;">[그림 3] shaders.c</p>
</div>

- 모든 쉐이더 코드는 하나의 바이트 배열에 포함되며,
정의된 상수를 통해 접근하여 프로그램에서 사용됩니다.
Material 또한 동일한 방식으로 lib와 resources.h / .c 형태로 제공됩니다.
<br><br>이러한 구조와 퍼뮤테이션(permutation) 방식은 여러 렌더링 라이브러리에서 유사하게 사용되지만,
신규 기능 개발 비용이 크고 올바른 사용을 위한 추가적인 학습 및 교육이 필요하다는 문제가 있었습니다.
<br><br>

### 해결방법
- 쉐이더 코드 관리는 기존 방식을 유지하되,
Material Package를 소스 코드 상에서 직접 정의하여 사용할 수 있도록 매크로 기반 구조를 도입했습니다.<br><br>
MATERIAL_PARAMETER 매크로를 통해 Material에서 사용하는 uniform 데이터를 선언하고,
MATERIAL_PROPERTY 매크로를 통해 depth, blend 등의 파이프라인 상태를 정의하도록 구성했습니다.
<br><br>
- 쉐이더 코드는 기존에 윈도우 리소스(Window Resource)로 등록·관리되고 있었기 때문에,
해당 리소스 ID를 제공하도록 구성하고,
Material 생성 시 이를 전달받아 사용하도록 구현했습니다.
<br><br>

### 설계 관점
- PSO(Pipeline State Object) 개념을 염두에 둔 Material 구조를 설계하여,
  DirectX 12 및 Vulkan 파이프라인 모델로의 확장을 고려<br><br>
- PSO는 일반적으로 생성 이후 변경이 어려운 객체이지만,
  Material은 하나의 PSO 개념을 표현하면서도
  런타임에서 상태를 유연하게 변경할 수 있도록 설계<br><br>
- 실제 PSO 객체 생성 및 캐싱은 각 Driver 레벨에서 담당하도록 하여,
  조직 규모상 코드 레벨의 잦은 수정과 실험을 수용할 수 있는 구조를 선택<br><br>

### 성과
- 렌더링 구조를 Material 중심으로 재구성하여
쉐이더 프로그램과 파이프라인 상태를 일관성 있게 관리할 수 있는 구조를 확립
<br><br>
- 전역 Shader 객체로 인해 발생하던 상태 오염 문제를
Material 단위로 상태를 분리함으로써 근본적으로 해결
<br><br>


### 향후 계획
- 공용으로 사용되는 구조체 및 함수들을 분리하여
Filament와 유사한 형태의 shaders.lib 구조로 관리
<br><br>
- Slang의 specialization 및 Reflection API를 활용하여,
  컴파일 타임 퍼뮤테이션과 런타임 파이프라인 구성을
  유연하게 결합하는 구조 검토
<br><br>

### 사용 기술
- C++20
- OpenGL
- GLSL
- 자체 RHI 추상화 계층
- Windows API