# :pushpin: 여행 패키지 판매 플랫폼


## 1. 제작 기간 & 참여 인원
- 2023년 3월 27일 ~ 5월 8일 (진행중)
- 6명


## 2. 사용 기술
#### `Back-end`
  - Spring 5.2.18
  - Java 1.8
  - MySQL 5.2
  - JSP
  - AWS 3S
  
#### `Front-end`
  - HTML5
  - CSS3
  - javaScript ES6


## 3. ERD 설계 
- 이미지 

## 4. 구현 기능 핵심 코드 

### 4.0. 프로젝트 핵심 기능
- 접속하는 모든 사람 : 플래너가 추천하는 여행 정보 서칭 
- 플래너 : 여행 패키지 상품 등록, 등록한 패지키 상품 데이터 관리
- 유저 : 여행 피키지 상품 구매, 장바구니, 플래너 1:1 채팅 문의

<details>
<summary><b>구현 기능 설명 펼치기</b></summary>
<div markdown="1">

### 4.1. 전체 흐름
/ MVC 모델2 이미지 올리기 

### 4.2. 비디오 스트리밍 기능(미완성)

<details>
<summary> <b>Router</b> </summary>

```javascript
//get
browseRouter.get("/video", videos);
```

</details>

<details>
<summary> <b>Controller&Model</b> </summary>

```javascript
// 동영상 파일 경로 생성 함수
const getVideoPath = (id) => {
    return `videos/${id}.mp4`;
  };
  
  // 동영상 스트리밍을 처리하는 핸들러 함수
  const videos = async function (req, res) {
    try {
      const id = parseInt(req.params.id);
      const { range } = req.headers;
  
      // id가 숫자가 아닐 경우 400 오류 반환
      if (isNaN(id)) {
        return res.status(400).send("Invalid ID");
      }
  
      // 동영상 파일 경로 생성
      const videoPath = getVideoPath(id);
      const stat = fs.statSync(videoPath);
      const fileSize = stat.size;
      const CHUNK_SIZE = 10 ** 6; // 1MB
  
      // range 헤더에서 시작 지점(start) 추출
      const start = Number(range.replace(/\D/g, ""));
  
      // range 헤더에서 끝 지점(end) 추출하거나 파일 크기 - 1 지점으로 설정
      const end = Math.min(start + CHUNK_SIZE, fileSize - 1);
  
      // 요청한 범위가 파일 크기를 넘어설 경우 416 오류 반환
      if (start >= fileSize || end >= fileSize) { 
        return res.status(416).send("Requested Range Not Satisfiable");
      }
  
      // 응답 헤더 설정
      const contentLength = end - start + 1;
      const headers = {
        "Content-Range": `bytes ${start}-${end}/${fileSize}`,
        "Accept-Ranges": "bytes",
        "Content-Length": contentLength,
        "Content-Type": "video/mp4",
      };
      res.writeHead(206, headers);
  
      // 동영상 파일 읽기 스트림 생성
      const videoStream = fs.createReadStream(videoPath, { start, end });
  
      // 파일 읽기 스트림에서 에러 발생 시 500 오류 반환
      videoStream.on("error", (err) => {
        console.error(err);
        res.status(500).send("Internal Server Error");
      });
  
      // 파일 읽기 스트림과 응답 스트림을 연결하여 동영상 스트리밍 반환
      videoStream.pipe(res);
    } catch (err) {
      // 예기치 않은 에러 발생 시 500 오류 반환
      console.error(err);
      res.status(500).send("Internal Server Error");
    }
  };
```
</details>

</br>

## 5. 프로젝트 진행하며 어려웠던 점과 좋았던 점

### 5.1. **비디오 스트리밍 방법에 대한 문제**

#### 5.1.1. 해결방법

#### 5.1.2. 결과 

#### 5.1.3. 원인 

#### 5.1.4. 결론

### 5.2. 좋았던 점

</div>
</details>

</br>


