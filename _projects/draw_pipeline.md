---
layout: project
title: 드로우 파이프라인 개선
company: 이지로보틱스
period: 2025.08 – 2025.10
tech:
  - C++
  - OpenGL
  - GLSL
---

### 개요
오브젝트 유형별로 분리되어 사용되던 기존 렌더러 구조의 한계를 개선하고자,
외부 프로그램 구조에 의존하지 않으면서도 다양한 렌더링 요소를 범용적으로 처리할 수 있는
Command 기반 렌더러 파이프라인을 설계 및 구현했습니다.

이를 통해 렌더링 호출 흐름을 단순화하고,
Material 기반 파이프라인과 자연스럽게 연계되는 구조를 구축하는 것을 목표로 했습니다.
<br><br>

### 요구
- 기존 렌더러는 Solid, Wire, Model 등 오브젝트 유형별로 각각의 렌더러 클래스를 정의하여 사용하고 있었습니다.<br>
각 렌더러는 서로 다른 쉐이더를 사용하기도 했지만, 렌더러 자체는 쉐이더 프로그램에 대한 책임 없이
월드 행렬 바인딩과 드로우 콜만 수행하는 단순한 구조였습니다.
<br><br>
- 이로 인해 작업자는 렌더러의 Draw 함수 호출 이전에
쉐이더 바인딩, 리소스 바인딩 등을 별도로 수행해야 했고,
렌더러는 사실상 “그리기 호출을 위한 래퍼”로만 사용되는 경우가 많았습니다.
<br><br>
- 또한 렌더러 구현 방식이 싱글톤, static 클래스 등으로 혼재되어 있어
구조적 일관성이 부족하고 유지보수성이 저하된 상태였습니다.
<br><br>
- Material 기반 파이프라인 도입 이후에는
파이프라인 상태와 오브젝트 간의 결합도가 증가했고,
드로우 호출 또한 OpenGL API가 아닌 Driver 인터페이스를 통해 수행되면서
기존 렌더러의 기능적 역할이 더욱 축소되었습니다.
<br><br>

### 수행 업무
- Command 기반 렌더러 아키텍처 설계 및 핵심 구조 구현<br><br>
- 기존 오브젝트 단위 렌더러 구조 제거 및 단일 렌더링 파이프라인으로 통합<br><br>
- 인스턴싱 렌더링 흐름 통합 및 자동화
<br><br>

### 설계 방안
1. 외부 프로그램 의존성 제거
  기존 렌더러는 다음과 같은 형태로 정의되어 있었습니다.
  ```cpp
    SolidRenderer::Draw(SolidObject const* obj);
    WireRenderer::Draw(WireObject const* obj);
  ```
  이 구조는 렌더러가 외부 프로그램의 도메인 객체에 직접 의존하게 만들어
  렌더링 로직의 재사용성과 확장성을 크게 제한하는 문제가 있었습니다.
  이를 해결하기 위해, 렌더러에 전달되는 데이터 타입을
  렌더링에 필요한 범용 데이터로 한정했습니다.

    - RenderPrimitiveHandle
    - Material
    - RasterState
    - Transform 등
    
    작업자는 AddCommand() 함수를 통해 렌더링 작업을 예약하며, 렌더러는 위와 같은 데이터만을 기반으로 동작하도록 설계했습니다.<br><br>
  
2. 인스턴싱 렌더링 통합
  기존에는 인스턴싱을 지원하는 렌더러가 별도로 존재했기 때문에
  렌더링 흐름이 이원화되어 있었습니다.<br>
  이를 통합하기 위해:
  - 커맨드 생성 시 다수의 Transform을 전달할 수 있도록 설계
  - 전달된 Transform 목록을 기반으로 자동으로 인스턴싱 데이터 생성
  - 필요 시, 사전 구성된 Transform 버퍼를 BufferObjectHandle 형태로 전달 가능하도록 구성
<br><br>

3. Tagged Parameter 기반 Command 생성
  하나의 커맨드를 생성하기 위해 Point, Line, Triangle마다 
  별도의 구조체를 사용했습니다. <br>
  이는 각 프리미티브가 필요로 하는 파라미터가 다르기 때문입니다.<br><br>
  그러나 구조체 기반 방식은:
    - 매번 변수를 선언해야 하는 번거로움
    - 멤버 순서에 대한 의존성
    - 사용자 실수로 잘못된 값 전달 가능성<br>
  이를 해결하기 위해 C++20 Template, Concept, SFINAE를 활용한
  Tagged Parameter 기반 커맨드 생성 방식을 도입했습니다.<br>
  ```cpp
  renderer.AddCommand(
    tag::point::material(mi),
    tag::point::renderprimitive(rp),
    tag::point::transform(world),
    tag::point::size(ptSize)
  );
  ```
  <br>
  이 방식은 다음과 같은 장점을 가집니다.
    - 파라미터 순서에 의존하지 않는 API 제공
    - 불필요한 중간 구조체 및 임시 변수 제거
    - Concept 및 SFINAE 기반 제약을 통해 잘못된 파라미터 조합을 컴파일 타임에 차단
  ```cpp
  renderer.AddCommand(
    tag::point::material(mi),
    tag::point::renderprimitive(rp),
    tag::line::transform(world), // 잘못된 태그 조합
    tag::point::size(ptSize)
  );
  ```
  ```cpp
  renderer.AddCommand(
    tag::tri::material(mi),
    tag::tri::renderprimitive(rp),
    tag::tri::transform(world),
    tag::tri::width(width) // Triangle에 유효하지 않은 파라미터
  );
  ```



### 문제
- 새로운 렌더러를 적용하고 테스트하는 과정에서 특정 하드웨어에서 예기치 못한 문제가 발생했습니다.<br><br>
  워크스테이션 노트북으로 출고된 일부 Nvidia GPU 환경에서
  인스턴싱된 객체가 렌더링되지 않거나, 모든 인스턴스가 동일한 위치에 그려지는 현상이 발생했습니다.<br><br>

### 해결방법
1. 문제 재현 및 조사
  - Nsight Graphics를 활용해 문제 발생 환경에서 프레임 캡처 및 드로우 콜 분석
  - Nvidia Developer Forum 및 Khronos Group Forum을 통해 유사 사례 조사
  - 생성형 AI를 활용해 유사 드라이버 이슈 리포트 수집<br><br>

2. 원인 가설 수립
  - 현상 분석 결과, 인스턴싱 인덱싱에 사용되는 gl_BaseInstance 값이 특정 하드웨어에서 정상적으로 전달되지 않는다고 판단
  - 테스트 결과, Nvidia 제어판의 “쓰레드 최적화(Threaded Optimization)” 옵션이
    켜짐 또는 자동 상태일 때 gl_BaseInstance가 항상 0으로 전달되는 것을 확인<br><br>

3. 구조적 해결
  - 기존 구조에서는 하나의 큰 Transform 버퍼를 바인딩한 뒤 base instance 값을 통해 쉐이더에서 월드 행렬을 인덱싱
  - 드라이버 의존 문제를 회피하기 위해 다음과 같이 수정했습니다.
        - base instance 사용을 제거
        - 각 드로우 콜마다 Transform 버퍼에 오프셋을 적용하여 재바인딩
        - 쉐이더에서는 항상 base instance = 0을 기준으로 접근하도록 수정
          
<br><br>


### 성과
- 오브젝트 유형별로 분산되어 있던 렌더러 구조를 단일 Command 기반 파이프라인으로 통합<br><br>
- 렌더링 호출 흐름 단순화 및 Material 기반 파이프라인과의 결합도 개선<br><br>
- Tagged Parameter 기반 API 설계를 통해 컴파일 타임 검증 강화 및 런타임 오류 감소<br><br>
- 특정 GPU/드라이버 환경에서 발생하던 인스턴싱 렌더링 오류를 구조적 개선으로 해결<br><br>


### 향후 계획
- 전역 객체로 사용중인 렌더러 객체를 'Builder'와 'Pass'로 분할하여 커맨드 생성과 사용 책임을 분산<br><br>
- 내부 커맨드 관리 구조를 단순화하고 각 커맨드에 대하여 키를 할당해 커맨드 자동 정렬 등의 기능 확장<br><br>


### 사용 기술
- C++20
- OpenGL
- GLSL
- 자체 RHI 추상화 계층