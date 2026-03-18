프로젝트명: 배터리 제조라인 AI 비전검사 시스템 개발
개요
배터리 제조 공정 4개 라인(스택매거진, 커터매거진, 탭접힘, 소팅머신)에 대한 실시간 AI 비전검사 시스템을 설계 및 구현. Python 기반 AI 연구 모델을 TensorRT C++ 추론 DLL로 변환하여 현장 GPU 서버에 배포하고, C# WPF 호스트 앱과 연동하여 실시간 OK/NG/Questionable 판정 및 시각화를 수행.
담당 역할
•	4종 AI 추론 DLL 아키텍처 설계 및 C++ 구현 (C ABI export)
•	Python AI 모델의 전처리/후처리 로직을 C++ TensorRT로 1:1 이식
•	C# .NET 9 WPF 호스트 앱에서 DLL 동적 로드 및 P/Invoke 연동 구현
•	4개 DLL 공통 C ABI 규격 설계 및 구조체 레이아웃 통일
기술 상세
1. 2단계 파이프라인 추론 엔진 (C++17)
•	1단계 Trigger(YOLOv8 객체검출): 프레임에서 Cell/PNP 영역을 검출하여 키프레임 판정
•	2단계 Defect(3-class 분류): 키프레임 crop 영역에 대해 Abnormal/Normal/Empty 분류
•	키프레임 그룹 관리자 구현: 최대 5장/그룹, gap 기반 자동 finalize, 그룹 내 최고 점수 프레임만 콜백 전달 (Python KeyframeGroupManager 로직 1:1 이식)
2. TensorRT / ONNX Runtime 듀얼 백엔드
•	TensorRT 10.x FP16 .engine 기반 실시간 추론 (주 백엔드)
•	ONNX Runtime 1.22.x .onnx 기반 CPU/GPU 추론 (폴백)
•	모델 메타 JSON 자동 파싱: 입출력 텐서 스펙, 클래스명, precision, bbox_format 등을 런타임에 로드하여 모델 교체 시 DLL 재빌드 불필요
•	Initialize 시 빈 이미지 5회 warmup 추론으로 TensorRT 커널 컴파일 지연 제거
3. 전처리 파이프라인 (Python 알고리즘 1:1 이식)
•	Letterbox resize: 비율 유지 + 114 패딩, 좌표 역변환 (letterbox → 원본 좌표)
•	BGR→RGB 변환, /255 정규화, channel-wise mean/std 정규화 (모델 JSON에서 동적 로드)
•	NMS(Non-Maximum Suppression): per-class IoU 기반 중복 제거
•	Softmax 확률 파싱: FP16 half→float 변환 포함
4. 후처리 및 판정 로직
•	6개 임계값 기반 3-bucket 판정: OK/NG/기타 클래스별 Q(Questionable)/AB(Abnormal) 이중 임계값
•	Cell score × 0.2 + Defect score × 0.8 결합 점수로 그룹 내 최적 프레임 선택
•	판정 결과 + bbox + 시각화 이미지를 콜백으로 실시간 전달
5. 비동기 Push-Callback 아키텍처
•	Thread-safe 프레임 큐 (mutex + condition_variable, 용량 100, 블로킹 push)
•	Trigger 워커 → Defect 큐 → Defect 워커 2단계 파이프라인 스레딩
•	PLC 설비 상태 연동: NotifyEquipStatus로 RUN/STOP 전환, 비활성 시 큐 flush + 키프레임 상태 자동 초기화
6. C# WPF 호스트 앱 연동 (DynamicInferenceSession)
•	NativeLibrary 기반 DLL 동적 로드: 4개 DLL을 ModelType enum으로 선택, 동일 API로 호출
•	슬롯 기반 모델 경로 전달: MAX_MODEL_SLOTS=8 고정 포인터 배열로 DLL별 가변 모델 수 지원
•	Marshal 기반 네이티브 구조체 마샬링, 콜백 delegate GC 방지, 이미지 포인터 즉시 복사 등 P/Invoke 안전 패턴 적용
7. 4종 DLL 공통 ABI 설계
DLL	공정	모델 구조	특이사항
HmStkDLL	스택매거진	Trigger(검출) + Defect(분류)	키프레임 그룹핑, PLC 상태 연동
HmCutterDLL	커터매거진	Trigger + Defect	HmStkDLL과 동일 ABI
LG_ENSOL_DLL	탭접힘	Tab Model + Tape Model	기존 VACL_MODULE 래핑
SmAdDLL	소팅머신	KeyframeDetector + FaultDetector + AnomalyDetector	이상탐지(Anomaly) 모듈 포함
기술 스택
•	언어: C++17 (MSVC v143), C# 13.0 (.NET 9)
•	AI 추론: TensorRT 10.x, ONNX Runtime 1.22.x, CUDA 12.x, cuDNN 9.x
•	영상처리: OpenCV 4.11.x
•	UI: WPF (.NET 9), MVVM
•	빌드: Visual Studio 2022, x64 Release/Debug
성과
•	Python 프로토타입 대비 추론 지연 시간 대폭 감소 (TensorRT FP16 엔진 + warmup 적용)
•	4개 공정 DLL을 공통 C ABI로 통일하여 호스트 앱 코드 재사용 극대화
•	모델 메타 JSON 기반 설계로 모델 버전 교체 시 DLL 재빌드 없이 현장 배포 가능
•	비동기 Push-Callback + PLC 상태 연동으로 설비 가동/정지 시 무중단 전환 구현
---
