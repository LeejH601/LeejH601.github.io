---
layout: project
title: 렌더링 품질 향상
company: 이지로보틱스
period: 2024.04 – 2024.06
tech:
  - C++
  - OpenGL
  - GLSL
  - Windows API
---

# 개요
자사 프로그램의 렌더링 품질을 개선하기 위해
PBR(Physically Based Rendering) 기반 렌더링 기법을 도입하여 보다 사실적인 시각적 표현을 구현하고,
GDC 2017 「FrameGraph: Extensible Rendering Architecture in Frostbite」 세션에서 소개된 FrameGraph 구조를 참고하여
렌더 패스 관리 구조를 개선했습니다.

이를 통해 렌더링 파이프라인을 체계화하고,
기능 확장과 유지보수가 용이한 구조를 구축하는 것을 목표로 했습니다.
<br><br>

# 요구
- 자사 프로그램을 사용하는 고객사로부터 기존 대비 더 사실적인 렌더링 품질에 대한 요구가 지속적으로 제기되었습니다.<br><br>
- 기존 렌더링 파이프라인은 기능 확장과 품질 옵션 추가 시 관리가 어려운 구조로,
렌더 패스 간 의존성을 체계적으로 관리할 필요성이 존재했습니다.
<br><br>

# 수행 업무
- PBR, IBL(Imaged Based Lighting), SSAO(Screen Space Ambient Occlusion) 구현 및 적용<br><br>
- GDC FrameGraph 발표 자료 분석 및 Filament FrameGraph 구조 분석 후 사내 렌더러에 맞게 구현<br><br>
- 디퍼드 렌더링 파이프라인 설계 및 적용
<br><br>

# 문제
- 기존 파이프라인에서는 Aliasing 해결을 위해 OpenGL MSAA 버퍼를 사용하고 있었으나, 디퍼드 렌더링 파이프라인에서는 MSAA 적용에 구조적 제약이 존재했습니다.<br><br>
- 디퍼드 파이프라인용 대안으로 FXAA, SMAA를 구현해 테스트했으나, 작은 오브젝트 및 얇은 기하 구조에서 Aliasing 품질이 충분하지 않았습니다.<br><br>
- TAA 역시 구현하여 테스트했으나, 프로그램 특성상 사용자 입력에 따라 프레임 레이트가 크게 저하되는 구간이 많아 Temporal 기법 적용에 한계가 존재했습니다.<br><br>


# 해결방법
- 디퍼드 환경에서 AA 품질 개선을 위한 추가 연구·개발은 비용 대비 효과가 제한적이라 판단하여, MSAA를 유지하되 렌더링 파이프라인 구조를 변경하는 방향으로 접근<br><br>
- 프로그램 특성상 다수의 동적 광원을 사용하는 경우가 드물다는 점에 착안하여, Geometry 패스를 단순 G-buffer 생성 단계가 아닌 Forward Lighting & Geometry 패스로 변경<br>
  해당 패스에서 MSAA 텍스처인 Direct, Specular, Indirect, Indirect Specular 버퍼에 조명 결과를 직접 저장하고, 이후 후처리 패스에서 이를 합성하도록 구성<br><br>
- SSAO는 MSAA 텍스처와 구조적으로 결합하기 어려워, 최종 합성 전 단계에서 FXAA를 적용하여 과도한 Aliasing만 보정하도록 처리<br><br>


# 성과
- PBR 기반 렌더링 기법 도입을 통해 사용자 요구에 부합하는 렌더링 품질 개선<br><br>
- FrameGraph 기반 디퍼드 파이프라인을 구축하여 기존 저품질 파이프라인을 포함한 여러 렌더링 옵션을 설정에 따라 손쉽게 전환하고 유지보수할 수 있는 구조로 개선<br><br>


# 향후 계획
- AA 관련 기술에 대한 추가 연구를 통해 메모리 사용량 증가를 유발하는 MSAA 의존도를 점진적으로 축소<br><br>
- 그림자, 간접광 등 사실적인 표현을 강화하기 위한 렌더링 기능 확장<br><br>


# 사용 기술
- C++20
- OpenGL
- GLSL
- 자체 RHI 추상화 계층
- Windows API