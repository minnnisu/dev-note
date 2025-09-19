# [만수무강] 외부 API에 비동기를 적용하여 API 응답 시간 90% 향상

만수무강에서는 사용자가 음성을 녹음하고 저장하면, 이를 텍스트로 변환하고 서버에 녹음 파일과 변환된 텍스트를 함께 저장하는 기능을 제공하고 있습니다.

<img width="1000" alt="image" src="https://github.com/user-attachments/assets/62a5b29c-9e0d-4c69-9e00-3b53cc465eae" />

하지만 초기 구현에서는 음성 파일을 서버에 저장할 때 **2~3초의 높은 응답 지연이 발생**하여 사용자 경험에 영향을 주고 있었습니다.  
더 나은 사용자 경험을 위해 파일 저장 과정에서 발생하는 병목을 해결하고자 합니다.

## 1. 병목 지점 파악
서버는 음성을 받아 OpenAI에서 제공하는 Whisper API(STT, Speech-To-Text)를 호출하여 텍스트 변환을 진행합니다.  

음성 저장 API를 디버깅한 결과, Whisper API를 호출하고 응답을 받아오는 과정에서 대부분의 시간이 소요되고 있음을 확인했습니다.  
즉, **외부 API 호출만 개선해도 전체 API 지연 시간을 크게 줄일 수 있다고 판단**했습니다.

## 2. 병목 지점 개선 방안
아래는 기존의 음성 녹음을 저장하는 API의 Service 레이어 로직입니다.

병목 발생 가능성이 있는 부분은 `openAIClientService.createTranscription(request)` 메소드로, Whisper API를 통해 STT 작업을 수행하는 지점입니다.

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

Whisper API 호출은 응답이 올 때까지 서버가 대기해야 하므로 전체 응답 시간이 길어졌습니다.
이를 해결하기 위해 비동기 처리를 적용했습니다.

즉, 클라이언트가 보낸 음성 파일을 우선 저장하고 곧바로 응답을 반환한 뒤, 이후에 Whisper API 호출을 별도로 수행하도록 구조를 변경했습니다.

### 2-1. 개선된 코드

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
```
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

## 3. 병목 지점 개선 결과

<img width="1000" alt="image" src="https://github.com/user-attachments/assets/bf1ff6a7-edab-4007-8b4b-651787c9abf7" />

Postman으로 음성 저장 API를 다시 호출해본 결과,
응답 시간이 100~200ms 수준으로 약 90% 개선되었습니다.

결과적으로, 해당 API의 높은 응답 시간은 Whisper API를 동기적으로 호출한 데서 비롯된 병목 현상이었으며, 비동기 처리를 통해 근본적인 문제를 해결할 수 있었습니다.

