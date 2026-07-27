https://share.gemini.google/HCvbOXCWy6Zn






#if 조건부 컴파일 매크로가 복잡하게 얽혀 있고, 초기화 단계에서 모든 메모리를 일괄 작성하는 구조에서는 일반적인 정적 분석 도구가 "초기화 시 발생한 Write"를 "유효한 사용"으로 오인하기 때문에 미사용 메모리를 찾아내기 어렵습니다.
​하지만 **구간 분리형 정적 분석(Phase-Aware Static Analysis)**과 Linux OS 레벨의 런타임 액세스 추적 기법을 결합하면 초기화 이후 전혀 읽히거나 쓰이지 않는(Dead Data) 영역을 명확히 찾아낼 수 있습니다.
​
1. 구간 분리형 정적 분석 (Clang AST Matcher 활용)
​일반 정적 분석 도구는 코드 전체에서의 참조 여부만 따집니다. 이를 해결하려면 **초기화 함수(Init Phase)**와 **메인 로직(Runtime Phase)**을 분리하여 분석해야 합니다.
​원리:
​콜 그래프(Call Graph)를 통해 main() 진입 후 init_system() 구간과 main_loop() 구간을 분리합니다.
​Clang AST Matcher를 통해 특정 구조체 멤버/배열(MemberExpr, ArraySubscriptExpr)의 참조 위치를 수집합니다.
​Runtime Phase 구간에서의 Read/Write 횟수가 0인 필드를 도출합니다.
​구현 방법:
​Clang 기반 커스텀 툴을 작성하여 Init 구역에 속한 AST Node의 Access는 무시하고, Runtime 구역의 AST Node에서 해당 구조체 심볼이 등장하는지 카운팅합니다.
​
2. Multi-Configuration Matrix 분석 (매크로 조합 검증)
​#if FEATURE_A와 같은 매크로로 인해 특정 형상에서는 쓰이지만 다른 형상에서는 쓰이지 않아 메모리가 낭비될 수 있습니다.
​방안:
​빌드 시스템(CMake/Make)에서 존재하는 **주요 #if 매크로 조합(Configuration Matrix)**을 정의합니다.
​각 형상별로 정적 분석을 자동 수행하여 **"특정 형상 A에서만 유효하고, 형상 B/C/D에서는 초기화만 되고 버려지는 구조체 필드/배열"**의 리스트를 매핑합니다.
​적용점:
​로직 코드뿐만 아니라 구조체 정의(Header File) 내부에도 동일한 #if FEATURE_A 매크로를 적용하여, 해당 형상 빌드 시 구조체 크기 자체가 줄어들도록 리팩토링합니다.
​
3. [실무 추천] Linux Soft-Dirty Bit를 활용한 메모리 추적
​Shared Memory와 Huge Page를 사용하는 시스템 특성상, OS 레벨에서 초기화 이후 실제로 터치(Touch)되지 않는 메모리 페이지를 정밀하게 추적하는 방법이 가장 확실합니다.
​작동 원리: Linux Kernel의 Soft-Dirty 비트를 초기화 직후 리셋한 뒤, 테스트 러닝을 진행하여 안 읽힌 페이지 오프셋을 역추적합니다.
