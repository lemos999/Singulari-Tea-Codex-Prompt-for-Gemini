# Singulari-Tea Codex: A Modular Architecture for Dynamic Narrative Simulation
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

## 1. 개요 (Overview)

이 문서는 'Singulari-Tea Codex' 프로젝트의 기술 아키텍처, 설계 원칙 및 구현 세부 사항을 설명하는 공식 기술 백서입니다. 
본 프로젝트는 단일 책임 원칙(Single Responsibility Principle)에 기반한 모듈형 프롬프트 아키텍처를 통해, 복잡하고 상태 의존적인 서사 시뮬레이션의 안정성, 확장성, 유지보수성을 확보하는 것을 목표로 합니다.

## 2. 기술적 배경 및 실행 환경

### 2.1. 목표 LLM: Google Gemini 2.5 Pro

본 아키텍처는 Google Gemini 2.5 Pro 모델의 특성을 최대한 활용하도록 설계되었습니다. 시뮬레이션의 성공적인 구동은 다음 두 가지 핵심 요소에 크게 의존합니다.

1.  강력한 프롬프트 이행 능력 (High-Fidelity Prompt Adherence): 수십 개의 분리된 모듈의 지시사항과 복잡한 상호작용 로직을 오류 없이 순차적으로 이행하는 능력이 필수적입니다.
2.  대용량 컨텍스트 창 (Large Context Window): 시뮬레이션의 모든 상태를 기록하는 `SHN` 객체와 전체 모듈 코드를 컨텍스트 내에 유지해야 하므로, 최소 1M 토큰 이상의 컨텍스트 창을 갖춘 모델이 요구됩니다.

### 2.2. 알려진 한계 (Known Limitations)

-   언어 고증 및 번역의 불안정성: `F0_language_engine_seal` 모듈이 수행하는 실시간 언어 고증 및 번역 기능은 간헐적으로 '컨텍스트 오염(Context Pollution)' 현상으로 인해 의도대로 작동하지 않을 수 있습니다. 
이는 프롬프트 로직의 결함이라기보다는, 복잡한 컨텍스트 내에서 특정 지시사항의 우선순위를 일관되게 유지하는 현세대 LLM의 근본적인 성능 한계에 기인하는 것으로 추론됩니다. 
차후 Gemini 3.0 Pro와 같은 차세대 모델이 출시되면, 이러한 시뮬레이터의 일관성 문제가 크게 개선될 것으로 전망합니다.

### 2.3. 타 모델 호환성 (Compatibility with Other LLMs)

본 프롬프트 아키텍처를 GPT-5나 Claude 4 등의 타사 모델에서 직접 사용하는 것은 권장되지 않으며, 성공적인 실행을 보장하기 어렵습니다. 주된 이유는 다음과 같습니다.

-   시스템 프롬프트 간섭: 대부분의 상용 모델은 사용자가 제어할 수 없는 강력한 기본 시스템 프롬프트(예: "You are a helpful assistant...")를 가지고 있습니다. 이는 시뮬레이션이 요구하는 완전한 페르소나 제어를 방해하고, 예기치 않은 동작을 유발합니다.
-   출력 요약 경향: 일부 모델은 긴 출력을 자동으로 요약하거나, 턴제 상호작용에 최적화되지 않은 응답을 생성하는 경향이 있습니다. 이는 시뮬레이션의 상태 정보와 UI 구조를 손상시킬 수 있습니다.

## 3. 핵심 아키텍처 (Core Architecture)

### 3.1. 핵심 개념 (Core Concepts)

-   SHN (Soulforged Chronicle): 시뮬레이션의 모든 상태(주인공, 세계, 발견 등)를 포함하는 중앙 JSON 객체. 모든 모듈의 유일한 '진실의 원천(Single Source of Truth)'으로 기능합니다.
-   모듈형 설계 (Modular Design): 모든 기능은 `prompt_src/` 내에서 명확한 책임을 가진 독립된 파일로 분리됩니다. 이는 코드의 재사용성을 높이고 디버깅을 용이하게 합니다.
-   E0 Master CoT Loop: 사용자 입력에 따라 필요한 모듈을 순차적으로 호출하여 턴을 처리하는 중앙 오케스트레이션 엔진입니다. 모든 데이터 흐름과 실행 순서는 E0에 의해 제어됩니다.

### 3.2. 실행 흐름 (Execution Flow)
<img width="863" height="872" alt="Image" src="https://github.com/user-attachments/assets/0b41f171-f8bd-48dc-b196-65c4d7a909f0" />

#### 3.2.1. 초기화 시퀀스 (Initialization Sequence)

1.  [G0] Onboarding: 사용자 입력을 분석하여 새 시뮬레이션 시작 또는 SHN 데이터 로드를 결정합니다.
2.  [C1] Epic Chronicle: 주인공의 정체성 및 핵심 데이터(특히 언어 프로필)를 생성하여 SHN에 저장합니다.
3.  [G1/G2] World & Narrative Generation: SHN에 기록된 데이터를 기반으로 세계의 법칙을 선언하고 초기 서사를 생성합니다. 이 과정에서 발생하는 모든 캐릭터 발화는 `F0_language_engine_seal`의 검증을 거칩니다.
4.  [E0] Handoff: 초기화 완료 후, 제어권을 메인 루프인 E0으로 전달합니다.

#### 3.2.2. 메인 인터랙션 루프 (Main Interaction Loop)

매 턴마다 `E0` 모듈은 다음의 '생각의 사슬(Chain-of-Thought)'을 엄격하게 따릅니다.

1.  Intent Parsing: 사용자 입력을 해석합니다.
2.  World State Update: 물리(`SYS_PHYSICS`), 건강(`SYS_HEALTH`) 등 시뮬레이션 엔진을 호출하여 SHN 데이터를 갱신합니다.
3.  Data-to-Text Conversion: `H0` 모듈이 갱신된 수치 데이터를 서술형 텍스트로 변환합니다.
4.  Narrative Generation: `D0`, `D1` 등 서사 모듈을 호출하여 메인 스토리를 생성합니다.
5.  Choice Generation: `H2` 모듈이 현재 상황에 맞는 선택지를 생성합니다.
6.  Final Rendering: `H1` 모듈이 모든 텍스트 컴포넌트를 조합하여 최종 UI를 출력합니다.

## 4. 결론 (Conclusion)

'Singulari-Tea Codex'는 단일 책임 원칙을 프롬프트 엔지니어링에 적용하여 복잡한 동적 시뮬레이션의 개발 및 유지보수를 체계화한 사례입니다. 이 모듈형 아키텍처는 다음과 같은 명확한 이점을 제공합니다.

-   안정성 (Stability): 모듈 간의 낮은 결합도는 수정 작업이 시스템 전체에 미치는 부작용을 최소화합니다.
-   확장성 (Scalability): 새로운 기능을 기존 코드에 영향을 주지 않고 독립된 모듈로 추가할 수 있습니다.
-   유지보수성 (Maintainability): 문제 발생 시 책임 범위가 명확한 특정 모듈만 수정하면 되므로 디버깅이 용이합니다.

## 5. 사용법 및 정책 (Usage & Policy)

### 5.1. 사용 가이드 (Getting Started)

1.  프로젝트 설정: `project-structure.txt`에 명시된 디렉토리 구조에 따라 프롬프트 파일을 배치합니다.
2.  모듈 수정: `prompt_src/` 내에서 필요한 모듈의 내용을 수정하거나 새로운 모듈을 추가합니다.
3.  번들링: 제공된 번들러 스크립트(`prompt-bundler.js`)를 사용하여 `prompt_src/` 내의 모든 모듈을 하나의 `master_prompt.txt` 파일로 병합합니다.
4.  실행: 생성된 `master_prompt.txt(Singulari-Tea Codex_vx_xxxxxx.txt)`의 전체 내용을 Gemini 2.5 Pro를 지원하는 환경에 입력하여 시뮬레이션을 시작합니다.

### 5.2. 소스코드 정책 (Source Code Policy)

-   본 리포지토리에 포함된 모든 `*.prompt.txt` 파일은 자유롭게 열람 및 수정할 수 있습니다.
-   단, `prompt-bundler.js`를 포함한 번들링 관련 스크립트의 소스코드는 비공개입니다.
-   이 프로젝트는 apache license 2.0에 따라 배포됩니다. 


# Singulari-Tea Codex: A Modular Architecture for Dynamic Narrative Simulation

## 1. Overview

This document is the official technical whitepaper for the 'Singulari-Tea Codex' project, detailing its technical architecture, design principles, and implementation specifics. The project aims to achieve stability, scalability, and maintainability in complex, state-dependent narrative simulations through a modular prompt architecture based on the Single Responsibility Principle.

## 2. Technical Background and Execution Environment

### 2.1. Target LLM: Google Gemini 2.5 Pro

This architecture is specifically designed to leverage the capabilities of the Google Gemini 2.5 Pro model. The successful execution of the simulation is highly dependent on the following two key factors:

1.  High-Fidelity Prompt Adherence: The ability to execute the instructions and complex interaction logic of dozens of separate modules sequentially and without error is essential.
2.  Large Context Window: A model with a context window of at least 1M tokens is required to maintain the `SHN` (the simulation state object) and the entire module codebase within the context.

### 2.2. Known Limitations

-   Instability in Linguistic Fidelity and Translation: The real-time linguistic verification and translation functions performed by the `F0_language_engine_seal` module may occasionally malfunction due to a phenomenon referred to as 'Context Pollution'. This is presumed not to be a flaw in the prompt logic but rather a fundamental performance limitation of the current generation of LLMs in consistently maintaining the priority of specific instructions within a complex context. It is anticipated that with the release of next-generation models like Gemini 3.0 Pro, this issue of simulator consistency will be significantly improved.

### 2.3. Compatibility with Other LLMs

Directly using this prompt architecture on other models such as GPT-5 or Claude 4 is not recommended, and successful execution cannot be guaranteed. The primary reasons are as follows:

-   System Prompt Interference: Most commercial models have powerful, non-configurable base system prompts (e.g., "You are a helpful assistant..."). This interferes with the complete persona control required by the simulation and can lead to unexpected behavior.
-   Output Summarization Tendency: Some models have a tendency to summarize long outputs or generate responses that are not optimized for turn-based interaction. This can corrupt the simulation's state information and UI structure.

## 3. Core Architecture

### 3.1. Core Concepts

-   SHN (Soulforged Chronicle): A central JSON object that contains the entire state of the simulation (protagonist, world, discoveries, etc.). It functions as the 'Single Source of Truth' for all modules.
-   Modular Design: All functionalities are separated into independent files with clear responsibilities within the `prompt_src/` directory. This enhances code reusability and simplifies debugging.
-   E0 Master CoT Loop: A central orchestration engine that processes turns by sequentially calling the necessary modules in response to user input. All data flow and execution order are controlled by E0.

### 3.2. Execution Flow
<img width="863" height="872" alt="Image" src="https://github.com/user-attachments/assets/0b41f171-f8bd-48dc-b196-65c4d7a909f0" />

#### 3.2.1. Initialization Sequence

1.  [G0] Onboarding: Analyzes user input to determine whether to start a new simulation or load SHN data.
2.  [C1] Epic Chronicle: Generates the protagonist's identity and core data (especially the linguistic profile) and saves it to the SHN.
3.  [G1/G2] World & Narrative Generation: Declares the laws of the world and generates the initial narrative based on the data stored in the SHN. All character utterances during this process are validated by the `F0_language_engine_seal`.
4.  [E0] Handoff: After initialization is complete, control is passed to the main loop, E0.

#### 3.2.2. Main Interaction Loop

For each turn, the `E0` module strictly follows this Chain-of-Thought (CoT):

1.  Intent Parsing: Interprets the user's input.
2.  World State Update: Calls simulation engines like physics (`SYS_PHYSICS`) and health (`SYS_HEALTH`) to update the SHN data.
3.  Data-to-Text Conversion: The `H0` module converts the updated numerical data into descriptive text.
4.  Narrative Generation: Calls narrative modules such as `D0` and `D1` to generate the main story.
5.  Choice Generation: The `H2` module generates context-appropriate choices.
6.  Final Rendering: The `H1` module assembles all text components into the final UI for output.

## 4. Conclusion

The 'Singulari-Tea Codex' is a case study in systematizing the development and maintenance of complex dynamic simulations by applying the Single Responsibility Principle to prompt engineering. This modular architecture offers the following clear advantages:

-   Stability: Loose coupling between modules minimizes the side effects of modifications on the entire system.
-   Scalability: New features can be added as independent modules without affecting existing code.
-   Maintainability: When issues arise, debugging is simplified as it's confined to specific modules with clear responsibilities.

## 5. Usage & Policy

### 5.1. Getting Started

1.  Project Setup: Arrange the prompt files according to the directory structure specified in `project-structure.txt`.
2.  Module Modification: Modify the content of existing modules or add new ones within the `prompt_src/` directory.
3.  Bundling: Use the provided bundler script (`prompt-bundler.js`) to merge all modules from `prompt_src/` into a single `master_prompt.txt` file.
4.  Execution: Input the entire content of the generated `master_prompt.txt(Singulari-Tea Codex_v1_250829.txt)` into an environment that supports Gemini 2.5 Pro to start the simulation.

### 5.2. Source Code Policy

-   All `*.prompt.txt` files included in this repository are open for viewing and modification.
-   However, the source code for bundling-related scripts, including `prompt-bundler.js`, is not public.(Commercial use requires preserving the license information)
-   This project is licensed under the apache license 2.0.
