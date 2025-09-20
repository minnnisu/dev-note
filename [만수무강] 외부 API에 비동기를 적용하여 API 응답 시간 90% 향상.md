## 1. 문제 상황

만수무강 서비스에는 사용자가 음성 파일의 내용을 텍스트로 변환하여 서버에 녹음 파일과 변환된 텍스트를 함께 저장하는 기능을 제공하고 있습니다. 이 과정에서 음성 파일의 내용을 텍스트로 변환하기 위해 Whisper의 STT 기능을 외부 API로 함께 사용하여 개발하였습니다.

### 병목 발생

<p align="center">
  <img src="https://github.com/user-attachments/assets/62a5b29c-9e0d-4c69-9e00-3b53cc465eae" width="1000" />
</p>

기능 개발이 끝나고 Postman을 통해 API 테스트를 진행한 결과 **2~3초의 높은 응답 지연이 발생**하여 사용자 경험에 영향을 주고 있었습니다.
더 나은 사용자 경험을 위해 파일 저장 과정에서 발생하는 병목이 해결되어야 하는 상황입니다.

<br>

## 2. 목표
일반적으로 응답시간이 3초만 넘어도 사용자의 이탈이 발생한다고 합니다. 그렇기에 사용자의 이탈을 방지하기 위해 **응답시간을 1s 미만이 되도록 개선하는 것을 목표로 설정**하였습니다.

<br>

## 3. 해결 방안

### 1. 병목 지점 파악
먼저 음성 녹음 저장을 진행하는 핵심 로직인 `RecordService`의 `saveRecord()` 메소드의 일부분을 살펴보겠습니다.

``` java
@Transactional
public RecordSave.Dto saveRecord(User user, Transcription.Request request) {

    // ...

    MultipartFile recordFile = request.getFile();

    if (recordFile == null) {
        throw new CustomErrorException(ErrorType.RecordFileNotFound);
    }
    try {
        AudioFileSaveDto audioFileSaveDto = fileService.saveAudioFile(recordFile);

        WhisperTranscription.Response transcription = openAIClientService.createTranscription(request); // 병목 지점
        String transcriptionText = transcription.getText();

        Record newRecord = recordRepository.save(
                Record.of(
                        validPatient,
                        audioFileSaveDto.getFileName(),
                        transcriptionText,
                        audioFileSaveDto.getAudioDuration()));

        return RecordSave.Dto.getInfo(newRecord);
    } catch (InternalErrorException e) {

    // ...

    }
}

```

해당 메소드는 다음 작업을 순서대로 진행합니다.
```
1. 음성 녹음 파일을 S3에 저장
2. Whisper API를 통해 SST 작업 수행
3. DB 내 음성 메타데이터 및 텍스트 변환 결과 저장
```
해당 메소드를 디버깅한 결과, Whisper API를 호출하고 응답을 받아오는 과정에서 대부분의 시간이 소요되고 있습니다.  
Whisper API 호출 후 응답이 올 때까지 서버가 대기해야 하므로 여기서 병목이 발생하여 전체 응답 시간이 길어졌던 것이였습니다.
그러므로 **해당 외부 API 호출 부분만 개선해도 전체 API 지연 시간을 크게 줄일 수 있다**고 판단했습니다.

### 2. 병목 개선 방안 모색

#### **1. 비동기 처리**

클라이언트가 보낸 음성 파일을 우선 저장하고 곧바로 응답을 반환한 뒤, 이후에 Whisper API 호출을 비동기적으로 수행하는 방식입니다.
이러면 사용자는 빠르게 응답을 받을 수 있어 사용자 경험이 개선됩니다.
하지만 서버가 바쁠 경우 쓰레드 부족으로 인해 STT를 작업이 지연될 수 있습니다.


#### **2. @Scheduled 이용**

클라이언트가 보낸 음성 파일을 우선 저장하고 @Scheduled를 이용하여 일정 시간(1분, 1시간 등) 마다 STT를 수행하는 방식입니다.
이는 구현이 단순하다는 장점이 있지만 변환할 음성 파일이 없을 경우에도 매번 스케줄링을 실행하기 때문에 필요없는 자원 사용이 발생할 수 있습니다.
또한 쌓인 작업들을 처리하기 위해 동시에 Whisper API를 여러 번 호출할 경우 `429 Too Many Requests`와 같은 거부가 발생할 수도 있습니다.

이 외에도 Kafka와 같은 메시지 큐를 활용하는 등 여러가지 방법이 있지만 **현재의 설계에서 크게 벗어나지 않으면서 단점보다 장점이 많은 비동기 처리 방식을 선택**하였습니다.

<br>

## 4. 개선된 코드 적용

새로운 `RecordAsyncService`를 생성하고, `updateTranscriptionAsync()` 메소드를 통해 비동기적으로 STT를 수행하도록 수정했습니다.

``` java
@Transactional
public RecordSave.Dto saveRecord(User user, Transcription.Request request) {

 // ...

    MultipartFile recordFile = request.getFile();

    if (recordFile == null) {
        throw new CustomErrorException(ErrorType.RecordFileNotFound);
    }
    try {
        AudioFileSaveDto audioFileSaveDto = fileService.saveAudioFile(recordFile);
        Record newRecord = recordRepository.save(
                Record.of(
                        validPatient,
                        audioFileSaveDto.getFileName(),
                        "녹음 내용을 분석하고 있어요!\n잠시만 기다려주세요.",
                        audioFileSaveDto.getAudioDuration()));

        // 비동기로 STT 진행
        recordAsyncService.updateTranscriptionAsync(newRecord, request);

        return RecordSave.Dto.getInfo(newRecord);
    } catch (InternalErrorException e) {

       // ...

    }
}
```
```java
@Service
@RequiredArgsConstructor
public class RecordAsyncService {

    // ...

    @Async
    public void updateTranscriptionAsync(Record record, Transcription.Request request) {
        try {
            WhisperTranscription.Response transcription = openAIClientService.createTranscription(request);
            String transcriptionText = transcription.getText();

            record.setContent(transcriptionText);
            recordRepository.save(record);
        } catch (Exception e) {
            log.error("Failedto transcribe audio for record {}", e.getMessage());
            record.setContent("음성변환에 실패하였습니다.");
            recordRepository.save(record);
        }
    }
}

```

<br>

## 5. 결과

| <img width="1000" alt="image" src="https://github.com/user-attachments/assets/bf1ff6a7-edab-4007-8b4b-651787c9abf7" /> | <img width="1476" alt="image" src="https://github.com/user-attachments/assets/af49b5d0-a297-4c04-b882-0a2f2d379e14" /> |
|:---:|:---:|
| 녹음 저장 API 호출 | 녹음 조회 API 호출 |

개선된 코드를 적용하여 Postman을 통해 음성 저장 API를 다시 호출해본 결과, 응답 시간이 100~200ms 수준으로 약 90% 개선되었습니다. 이로써 목표로 하였던 1s보다 더 빠른 응답 시간을 이끌어 낼 수 있었습니다.

결과적으로, 해당 API의 높은 응답 시간은 Whisper API를 동기적으로 호출한 데서 비롯된 병목 현상이었으며, 비동기 처리를 통해 근본적인 문제를 해결할 수 있었습니다.

