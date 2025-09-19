# [만수무강] 외부 API에 비동기를 적용하여 API 응답 시간 90% 향상

만수무강에서는 사용자가 음성을 녹음하고 저장하면, 이를 텍스트로 변환하고 서버에 녹음 파일과 변환된 텍스트를 저장하는 기능을 제공하고 있습니다.

<img width="1000" alt="image" src="https://github.com/user-attachments/assets/62a5b29c-9e0d-4c69-9e00-3b53cc465eae" />

하지만 초기 구현에서는 음성 파일을 서버에 저장할 때 **2~3s의 높은 응답 지연이 발생하여 사용자 경험에 영향을 주고 있었습니다**.
높은 사용자 경험을 위해 파일 저장 과정에서 발생하는 병목을 해결해보고자 합니다.

## 1. 병목 지점 파악
서버에서 음성을 받아 OpenAI에서 제공하는 Whisper API의 STT라는 외부 API를 호출하여 텍스트 변환을 진행합니다.

음성 저장 API를 디버깅 해보았을 때 Whisper API를 호출하고 응답을 받아오는데 대부분의 시간이 소요되는 것을 알 수 있었습니다. 외부 API를 호출하는 부분만 개선해도 전체적인 API의 지연시간이 대폭 감소할 것이라고 판단했습니다.

## 2. 병목 지점 개선 방안
아래는 기존의 음성 녹음을 저장하는 API의 Service 레이어 로직입니다.

이 중에서 병목이 발생한다고 생각되는 부분은 `openAIClientService.createTranscription(request)`으로 Whisper를 통해 STT 작업을 진행하는 메소드입니다.

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

어디서 병목을 발생하는지 파악했고 어떻게 이 병목을 어떻게 개선할 수 있을까요? **저는 비동기 방식을 사용하기로 결정했습니다.**
클라이언트가 보낸 음성 녹음을 먼저 저장하여 클라이언트에게 응답을 전송하고 이후에 SST 작업을 진행하도록 개선하고자 합니다. 이를 위해 코드를 아래와 같이 수정하였습니다.

**새로운 `RecordAsyncService` Service를 생성하고 `updateTranscriptionAsync()` 메소드를 통해 클라이언트에게 응답을 보내며 비동기적으로 SST를 수행합니다.**

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

다시 Postman을 통해 음성 저장 API를 호출한 결과 **응답 시간이 100~200ms로 약 90% 가량 대폭 개선되었습니다.**
이로써 해당 API의 높은 응답 시간은 Whisper API를 동기적으로 호출하였기 때문임이 확인되었습니다.

