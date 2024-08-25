# 프로젝트에서 파일 검증로직 개선하기

어려운 기술을 사용하는 작업보다. 진부해 보이는 작업이라도 완벽하게 만드려는 노력이 성장에 도움된다는 말을 들은것 같다.  
이 글은 나름대로 예외처리를 정확하게하려는 노력이담긴 기록용 글이다.  

지난번에 [프로젝트에서 파일 검증로직 만들기](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/fileValidaterStudy/MakeValidater.md)
에서 파일 업로드 기능 구현시에 고려했던 부분들을 다루었었다.  
이번 글은 그 기능의 개선편이다.  

팀원과의 코드리뷰 결과 기존 코드에서 예외처리에서 모호한 부분이 몇 군데 발견되었다.  

```java


/** 이름과 URL 추출 */
private String[] extractDataNameAndUrl(UploadDataRequest request){
        log.info("[DataService.extractDataNameAndUrl]");
        DataType dataType = request.getDataType();
        if(dataType.equals(DataType.LINK) && request.getLink() != null){
        String dataName = request.getLink();
        validateData(dataName,dataType);
        String dataUrl = request.getLink();
        return new String[]{dataName,dataUrl};
        }else if (request.getFile() != null) {
        String dataName = request.getFile().getOriginalFilename();
        validateData(dataName,dataType);
        String dataUrl = s3Uploader.uploadFileToS3(request.getFile(), S3_FOLDER);
        return new String[]{dataName,dataUrl};
        }
        else {
        throw new DataException(NONE_FILE);
        }
        }
```

바로 이 부분이다. 이 부분에서는 데이터 타입이 링크일때와 파일일때를 `if(dataType.equals(DataType.LINK)`
조건과 `else if (request.getFile() != null)`조건으로 나누어서 처리하고있다. 
하지만 이렇게되면 클라이언트에서 타입은 파일로 설정하고 String인 링크를 담아 업로드 요청을 할 경우, `NONE_FILE` 이라는 예외에 해당하는 메세지를 받게된다.
팀원과의 논의 끝에 타입이 파일인데 link를 전송했다는 것이 파일업로드를 하지않은 것은 맞지만 클라이언트에서 알아차리기에는 정보가 부족하다는 판단을 했다.  
때문에 그 부분을 적절하게 알려주기 위해 로직을 수정해 보기로했다.  

파일타입인데 링크를 업로드할경우 `NONE_FILE` 이라는 예외를 터뜨리지 않기 위해서는 우선 조건을 바꿔야한다. 기존의 파일이 NULL인지 아닌지로 구분하는 방식으로는
해당 상황을 잡을 수 없다.
따라서 나는 조건문을 `else if (dataType.equals(DataType.FILE)||dataType.equals(DataType.IMG))`으로 수정했다. 
파일타입을 받는 데이터 타입을 명시하는 조건으로 수정하여 해당 타입이면서 파일 필드를 누락하였을때 나타나는 예외를 처리할 수 있게되었다.  

내가 잡고싶은 예외가 파일을 올려야하는 경우인데 파일필드가 아닌 link 필드를 요청에 포함한 경우라는 것에 집중해 보기로했다.  

그러다가 떠오른 핵심은 필드가 채워지면 null이 아니라는 것이었다. 따라서 이 핵심을 이용해 검증 메서드를 만들어 보았다.  
메서드는 아래와 같이 완성되었디. 
```java
    private void validateType(UploadDataRequest request){
        log.info("[DataService.validateType]");
        if(request.getLink()!=null){ throw new DataException(INVALID_TYPE);}
    }
```
링크 필드에 값이 들어올경우 즉 link필드가 null이 아닐경우,  `INVALID_TYPE`이라는 예외를 터뜨리는 메소드이다.  

이제 아래 로직에서이 메소드를 삽입하는 위치가 중요하다.  
```java

    private String[] extractDataNameAndUrl(UploadDataRequest request){
        log.info("[DataService.extractDataNameAndUrl]");
        DataType dataType = request.getDataType();
        if(dataType.equals(DataType.LINK) && request.getLink() != null){
            String dataName = request.getLink();
            validateData(dataName,dataType);
            String dataUrl = request.getLink();
            return new String[]{dataName,dataUrl};
        }else if (dataType.equals(DataType.FILE)||dataType.equals(DataType.IMG)) {
            String dataName = request.getFile().getOriginalFilename();
            validateData(dataName,dataType);
            String dataUrl = s3Uploader.uploadFileToS3(request.getFile(), S3_FOLDER);
            return new String[]{dataName,dataUrl};
        }
        else {
            throw new DataException(NONE_FILE);
        }

```
`validateType`메서드를 `String dataUrl = s3Uploader.uploadFileToS3(request.getFile(), S3_FOLDER);`
이 줄 아래에 삽입한다면 `s3Uploader.uploadFileToS3`에 파일이 null일때 터지도록 처리해둔 `NONE_FILE`예외가 먼저 발생할 것이다.
때문에 else if문 내부 최상단에서 가장먼저 link필드에대한 검증로직을 수행하도록 추가하였다.  
```java
else if (dataType.equals(DataType.FILE)||dataType.equals(DataType.IMG)) {
            validateType(request);
            String dataName = request.getFile().getOriginalFilename();
            validateData(dataName,dataType);
            String dataUrl = s3Uploader.uploadFileToS3(request.getFile(), S3_FOLDER);
            return new String[]{dataName,dataUrl};
        }
```

끝이 아니다. 
아직 부족한점이 남아있었다.  
```java
    /** 이름과 타입으로 확장자 검사 */
    private void validateData(String dataName, DataType dataType){
        log.info("[DataService.validateData]");
        if(!fileValidater.validateFile(dataName,dataType)){
            throw new DataException(INVALID_EXTENSION);
        }
    }
```
바로 이녀석이다! 이녀석은 단순하게 `fileValidater.validateFile`의 반환결과에만 의존하고 타입에대한 처리는 직접하지 않고있었다.  
그냥 인자들만 넘겨주고 응답만 받아주는 녀석이었던 것이다.  
이러면 클라이언트에게 링크의 부적절함도 파일확장자라는 예외로 알려주는 부정확함이 있었다.  
따라서 아래와 같이 `validateData`를 수정했다.  
```java
    private void validateData(String dataName, DataType dataType){
        log.info("[DataService.validateData]");
        if (dataType.equals(DataType.LINK)&&!fileValidater.validateFile(dataName,dataType)){
            throw new DataException(INVALID_URL);
        } else if(!fileValidater.validateFile(dataName,dataType)){
            throw new DataException(INVALID_EXTENSION);
        }
    }
```
타입이 링크일경우를 별도로 처리하여 클라이언트에게 적절한 예외메세지를 전달할 수 있도록 수정했다.  

하지만 여전히 문제가 있다...  

파일필드는 누락하지않았지만 파일을 요청에 포함하지않는다면 파일이름은 빈문자열이 추출되어 `fileValidater.validateFile` 로 전달이 된다.  

그렇게되면 아래 로직에서는 파일이 포함되지않은 상황임에도 파일확장자 예외를 터뜨리게 된다.  
따라서 이에대한 처리가 필요했다.  
```java
    public boolean validateFile(String filename, DataType dataType){
        switch (dataType) {
            case IMG:
                return IMAGE_EXTENSIONS.contains(getFileExtension(filename).toLowerCase());
            case FILE:
                return FILE_EXTENSIONS.contains(getFileExtension(filename).toLowerCase());
            case LINK:
                return validateUrl(filename);
            default:
                return false;
        }
    }
```
하지만 `extractDataNameAndUrl`에서 `isempty`를 이용하자니 `link`에대한 처리가 어려워졌다.

사실 방법은 간단하다. `validateFile`에서 빈문자열을 처리하면된다. (문제 발생지점을 팬다.)  
따라서 아래처럼 수정해주면 문제가 해결된다.
```java
    public boolean validateFile(String filename, DataType dataType){
        switch (dataType) {
            case IMG:
                if(filename.equals("")) throw new DataException(NONE_FILE);
                return IMAGE_EXTENSIONS.contains(getFileExtension(filename).toLowerCase());
            case FILE:
                if(filename.equals("")) throw new DataException(NONE_FILE);
                return FILE_EXTENSIONS.contains(getFileExtension(filename).toLowerCase());
            case LINK:
                return validateUrl(filename);
            default:
                return false;
        }
    }

```
이로써 예외처리 작업이 마무리 되었다.  

만만하게 봤던 업로드 작업이었지만 좋은 API를 만드려고하니 생각보다 따질 것이 많았던 작업이었다.  
백엔드작업에서 논리가 중요함을 알 수 있던 작업이었고, 좋은 API를 만들기위한 유의미한 고민을 많이 안겨준 작업이었던 것 같다.  