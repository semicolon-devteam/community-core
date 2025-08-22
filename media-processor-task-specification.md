# Media-Processor 마이크로서비스 개발 업무지시서

## 📋 프로젝트 개요

**목표**: 클라우드플레어 타임아웃 문제 해결을 위한 비동기 파일 업로드 시스템 구현

**배경**: 현재 동기식 파일 업로드로 인해 대용량 파일 업로드 시 클라우드플레어에서 타임아웃 발생. 비동기 처리를 통해 사용자 경험 개선 및 안정성 확보 필요.

## 🎯 핵심 요구사항

### 1. 새로운 API 엔드포인트 개발

#### 1.1 POST `/api/media/upload-async`
**요청 형식**:
```json
{
  "postId": 123,
  "files": [
    {
      "fileName": "sample_image.jpg",
      "fileSize": 3145728,
      "fileType": "image/jpeg",
      "uuid": "file-uuid-1",
      "base64Data": "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQ..."
    }
  ]
}
```

**응답 형식**:
```json
{
  "success": true,
  "message": "파일 업로드가 시작되었습니다",
  "uploadId": "upload-session-uuid",
  "files": [
    {
      "uuid": "file-uuid-1",
      "status": "pending"
    }
  ]
}
```

#### 1.2 GET `/api/media/upload-progress/{postId}`
**응답 형식**:
```json
{
  "postId": 123,
  "overallProgress": 75,
  "files": [
    {
      "fileName": "sample_image.jpg",
      "fileSize": 3145728,
      "fileType": "image/jpeg",
      "fullPath": "/uploads/images/sample_image.jpg",
      "uuid": "file-uuid-1",
      "url": "/storage/uploads/images/sample_image.jpg",
      "thumbnailUrl": "/storage/uploads/images/thumbnails/sample_image_thumb.jpg",
      "status": "completed",
      "progress": 100,
      "uploadedAt": "2024-01-15T10:30:00Z"
    },
    {
      "fileName": "large_video.mp4",
      "fileSize": 27262976,
      "fileType": "video/mp4",
      "fullPath": "/uploads/videos/large_video.mp4",
      "uuid": "file-uuid-2",
      "url": "/storage/uploads/videos/large_video.mp4",
      "thumbnailUrl": "/storage/uploads/videos/thumbnails/large_video_thumb.jpg",
      "status": "watermarking",
      "progress": 75,
      "uploadedAt": null
    }
  ]
}
```

### 2. 파일 처리 워크플로우

#### 2.1 상태 관리
파일별 상태를 다음과 같이 관리:
- `pending`: 업로드 대기
- `uploading`: 수파베이스 업로드 중
- `watermarking`: 워터마크 처리 중 (이미지/비디오)
- `completed`: 완료
- `failed`: 실패

#### 2.2 처리 단계
1. **파일 수신 및 검증**
   - 파일 크기, 형식 검증
   - 중복 파일 체크
   - UUID 기반 임시 저장

2. **수파베이스 업로드**
   - 청크 단위 업로드로 progress 추적
   - 실패 시 재시도 로직 (최대 3회)
   - 업로드 완료 시 URL 생성

3. **후처리**
   - **이미지**: 썸네일 생성, 워터마크 추가
   - **비디오**: 썸네일 추출, 워터마크 추가
   - **기타 파일**: 바로 완료 처리

4. **데이터베이스 업데이트**
   - posts 테이블의 attachments 필드(JSON) 업데이트
   - 모든 파일 완료 시 posts.status를 NULL → DRAFT로 변경

### 3. 기술 스펙

#### 3.1 데이터베이스 스키마
기존 posts 테이블 활용:
```sql
-- posts 테이블의 attachments 컬럼 (JSON 타입)
-- 예시 데이터:
[
  {
    "fileName": "sample_image.jpg",
    "fileSize": 3145728,
    "fileType": "image/jpeg",
    "fullPath": "/uploads/images/sample_image.jpg",
    "uuid": "file-uuid-1",
    "url": "/storage/uploads/images/sample_image.jpg",
    "thumbnailUrl": "/storage/uploads/images/thumbnails/sample_image_thumb.jpg",
    "status": "completed",
    "progress": 100,
    "uploadedAt": "2024-01-15T10:30:00Z"
  }
]
```

#### 3.2 Redis 캐싱
업로드 진행도 실시간 추적을 위한 Redis 활용:
```
Key: upload_progress:{postId}
Value: JSON (위 응답 형식과 동일)
TTL: 1시간
```

#### 3.3 큐 시스템
파일별 비동기 처리를 위한 작업 큐:
- 큐 이름: `file_processing_queue`
- 작업자(Worker) 수: 3-5개 병렬 처리
- 재시도: 최대 3회, 지수 백오프

### 4. 에러 처리

#### 4.1 파일 업로드 실패
```json
{
  "fileName": "failed_file.jpg",
  "status": "failed",
  "progress": 0,
  "error": "파일 업로드 실패: 네트워크 오류"
}
```

#### 4.2 처리 실패 시나리오
- **네트워크 오류**: 3회 재시도 후 실패 처리
- **파일 손상**: 즉시 실패 처리, 사용자에게 재업로드 요청
- **용량 초과**: 즉시 실패 처리, 명확한 오류 메시지

### 5. 성능 요구사항

#### 5.1 응답 시간
- 비동기 업로드 시작: < 3초
- 진행도 조회: < 1초
- 대용량 파일(100MB): 5분 이내 완료

#### 5.2 동시 처리
- 동시 업로드 세션: 최대 100개
- 파일별 병렬 처리: 최대 5개
- 진행도 조회: 무제한 (캐싱 활용)

### 6. 보안 고려사항

#### 6.1 파일 검증
- 파일 시그니처 검증 (매직 넘버)
- 악성 코드 스캔 (선택사항)
- 파일 크기 제한: 이미지 10MB, 비디오 100MB

#### 6.2 인증
- JWT 토큰 기반 사용자 인증
- postId 소유권 검증
- API 요청 제한 (Rate Limiting)

## 🚀 구현 단계

### Phase 1: 기본 API 개발 (1주)
- [ ] 비동기 업로드 API 개발
- [ ] 진행도 조회 API 개발
- [ ] Redis 연동
- [ ] 기본 에러 처리

### Phase 2: 파일 처리 로직 (1주)
- [ ] 수파베이스 업로드 로직
- [ ] 이미지/비디오 후처리
- [ ] 큐 시스템 구현
- [ ] 재시도 로직

### Phase 3: 최적화 및 테스트 (0.5주)
- [ ] 성능 최적화
- [ ] 단위/통합 테스트
- [ ] 부하 테스트
- [ ] 문서화

## 📝 참고사항

### 기존 시스템 연동
현재 프론트엔드에서 다음과 같이 구현되어 있습니다:
- FileAttachment 인터페이스 확장 완료
- 진행도 UI 컴포넌트 완성
- 2초 간격 폴링 로직 구현

### 테스트 시나리오
1. **정상 케이스**: 이미지 + 비디오 혼합 업로드
2. **대용량 파일**: 100MB 비디오 파일
3. **네트워크 불안정**: 중간에 연결 끊김
4. **동시 업로드**: 여러 사용자의 동시 업로드

### 모니터링
- 업로드 성공률
- 평균 처리 시간
- 에러율 및 에러 유형
- 큐 대기 시간

## ✅ 완료 기준

1. 모든 API가 명세대로 동작
2. 진행도가 실시간으로 정확히 추적됨
3. 에러 발생 시 적절한 복구 및 메시지 제공
4. 성능 요구사항 충족
5. 프론트엔드와 정상 연동 확인

---

**담당팀**: Media-Processor 개발팀  
**우선순위**: 높음  
**완료 목표일**: 2주 이내  
**리뷰어**: 백엔드 팀장, 프론트엔드 팀장