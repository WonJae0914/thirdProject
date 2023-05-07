# :pushpin: 여행 패키지 판매 플랫폼

※ 프로젝트 소개 : </br>
`이 프로젝트는 여행 플래너 회사가 자사의 여행 플랜 패키지를 판매 및 관리할 수 있는 기능을 제공합니다.`</br>
`여행을 가고 싶은 사람들이 여행 정보, 여행 플랜 패키지를 비교하며 효율적인 구매를 할 수 있도록 제작되었습니다. `</br>
\- 유저 : 여행 정보 수집, 여행 플랜 구매, 플래너와 1:1 채팅 문의 </br>
\- 플래너 : 여행 플랜 등록, 판매, 데이터 관리

---

## 1. 제작 기간 & 참여 인원
- 2023년 4월 3일 ~ 5월 8일 (진행중)
- 팀프로젝트 : 6명


## 2. 사용 기술
#### `Back-end`
  - Spring 5.2.18
  - Java 1.8
  - Maven
  - MySQL 8.0.28
  - MyBatis 3.5.4
 
  
#### `Front-end`
  - HTML5
  - CSS3
  - javaScript ES6


## 3. ERD 설계 

![3차ERD설계](https://user-images.githubusercontent.com/120714001/236103812-7e80c194-8de4-4e4b-ba48-bcbecd0b839b.png)


## 4. 본인 구현 기능 핵심 코드 
- 리뷰 게시판 CRUD (파일 업로드, 페이징)
- 여행 정보 게시판 CRUD (파일 업로드, 페이징)
- 평점 기능
- 좋아요, 싫어요 기능

<details>
<summary><b>구현 기능 설명 펼치기</b></summary>
<div markdown="1">

### 4.1. 전체 흐름

  ![](https://velog.velcdn.com/images/hardworking/post/72189c93-7cf0-4f72-8913-0997b75bffbe/image.png)

---
### 4.2. 리뷰 게시판 CRUD

<details>
<summary> <b>Controller</b> </summary>
  

- 유저가 결제한 여행 플랜에 대한 리뷰 생성 시 텍스트와 함께 이미지 파일도 함께 업로드
- Apache Commons FileUpload 라이브러리 설치 후 Spring 컨텍스트에 MultipartResolver 빈을 등록
- 이미지 파일 AWS 서버에 저장 후 URL 주소 데이터 베이스에 저장 
```java
/**
	 * 리뷰 페이지 호출
	 * @param httpSession 현재 세션 정보
	 * @param planDTO 리뷰에 필요한 데이터를 담은 PlanDTO 객체
	 * @param mv ModelAndView 객체
	 * @return ModelAndView 객체에 데이터를 추가하고 view를 설정한 후 반환
	 */
	@GetMapping("create")
	public ModelAndView create(HttpSession httpSession, PlanDTO planDTO, ModelAndView mv){
		// 아이디 값 파라미터로 받기 -> 로그인 아이디 세션과 아이디 값 비교 (마이 페이지 구현 후 진행예정)
//		String longinId = httpSession.getAttribute("user_id");
//		if(user_id.equals(longinId)){ 아래 코드 넣기 }
		planDTO.setPlan_idx(66);
		PlanDTO planData = this.reviewService.getCreate(planDTO);
		mv.addObject("data", planData);
		mv.setViewName("/board/review/review_create");
		return mv;
	}

	/**
	 * 리뷰 생성
	 * 파일 업로드 기능 추가
	 * @param reviewDTO 리뷰 데이터를 담은 ReviewDTO 객체
	 * @param plannerRatingDTO 플래너 평점 데이터를 담은 PlannerRatingDTO 객체
	 * @param mv ModelAndView 객체
	 * @param httpSession 현재 세션 정보
	 * @param multipartFiles 업로드된 파일을 담은 List<MultipartFile> 객체
	 * @param plannerId 플래너 아이디
	 * @return ModelAndView 객체에 view를 설정한 후 반환합니다.
	 */
	@Auth(role=Auth.Role.USER)
	@PostMapping("create")
	public ModelAndView createPost(ReviewDTO reviewDTO, PlannerRatingDTO plannerRatingDTO, ModelAndView mv, HttpSession httpSession,
								   @RequestParam("file[]") List<MultipartFile> multipartFiles,
								   @RequestParam("planner_id") String plannerId) {
		try {
			String user_id = (String) httpSession.getAttribute("user_id"); // 로그인한 유저 아이디 세션
			plannerRatingDTO.setPlan_idx(66);
			plannerRatingDTO.setUser_id(user_id);
			this.reviewService.plannerRating(plannerRatingDTO); // 플래너 평점
			reviewDTO.setUser_id(user_id); // DTO에 유저 아이디 할당
			reviewDTO.setPlan_idx(66); // 임시 plan_idx값
			int review_idx = this.reviewService.create(reviewDTO); // 생성된 게시글 idx
			imgFileUpload(reviewDTO, multipartFiles, review_idx); // 이미지 파일 업로드 API
			if (review_idx != 0) {
				mv.setViewName("redirect:/review/detail/" + review_idx);
			} else {
				mv.setViewName("review/review_create");
			}
		} catch (Exception e) {
			// 예외 처리
			System.err.println("리뷰 생성 중 오류가 발생했습니다: " + e.getMessage());
			// 오류 페이지로 리다이렉트 또는 예외 처리에 맞는 다른 로직 수행
			mv.setViewName("/error/500");
		}

		return mv;
	}
	
	/**
	 * 이미지 파일 업로드 API
	 * @param reviewDTO 리뷰 데이터를 담은 ReviewDTO 객체
	 * @param multipartFiles 업로드된 이미지 파일들을 담은 List<MultipartFile> 객체
	 * @param review_idx 생성된 리뷰의 인덱스 값
	 */
	private void imgFileUpload(ReviewDTO reviewDTO, List<MultipartFile> multipartFiles, int review_idx) {
		try {
			if(multipartFiles !=null && !multipartFiles.isEmpty()) { // 이미지 파일이 존재하는 경우
				List<String> imgList = s3FileUploadService.upload(multipartFiles); // AWS서버 저장 후 URL값 List에 담기
				reviewDTO.setR_img(imgList); //
				reviewDTO.setReview_idx(review_idx);
				this.reviewService.createImg(reviewDTO);
			} else { // 이미지 파일이 없는 경우
				reviewDTO.setReview_idx(review_idx);
				this.reviewService.createImg(reviewDTO);
			}
		} catch (IOException e){
			throw new RuntimeException(e);
		}
	}
  /**
	 * 리뷰 디테일 페이지 호출
	 * 		 리뷰 디테일 페이지 이미지 파일 추가
	 * @param review_idx 조회할 리뷰의 인덱스
	 * @param reviewDTO 리뷰 데이터를 담은 ReviewDTO 객체
	 * @param mv ModelAndView 객체
	 * @return ModelAndView 객체에 데이터와 view를 설정한 후 반환
	 */
	@GetMapping("detail/{review_idx}")
	public ModelAndView detail(@PathVariable int review_idx,
							   ReviewDTO reviewDTO, ModelAndView mv){
		LikeUnlikeDTO likeUnlikeDTO = new LikeUnlikeDTO();
		reviewDTO.setReview_idx(review_idx);
		likeUnlikeDTO.setReview_idx(review_idx);
		LikeUnlikeDTO dto = this.reviewService.likeUnlikeCnt(likeUnlikeDTO);
		ReviewDTO detail = this.reviewService.detail(reviewDTO); // review 게시글 정보
		mv.addObject("likeUnlike", dto); // 좋아요, 싫어요 정보
		mv.addObject("data", detail); // 게시글 정보
		mv.setViewName("/board/review/review_detail");
		return mv;
	}

	/**
	 * 리뷰 수정 화면 호출
	 * @param review_idx 리뷰의 인덱스 값
	 * @param reviewDTO 리뷰 데이터를 담은 ReviewDTO 객체
	 * @param mv ModelAndView 객체
	 * @return ModelAndView 객체에 데이터를 추가하고 view를 설정한 후 반환
	 */
	@GetMapping("update/{review_idx}")
	public ModelAndView update(@PathVariable int review_idx, ReviewDTO reviewDTO, ModelAndView mv) {
		reviewDTO.setReview_idx(review_idx);
		ReviewDTO detail = this.reviewService.detail(reviewDTO); // 게시글 정보
		mv.addObject("data", detail); // 게시글 정보
		mv.setViewName("board/review/review_update");
		return mv;
	}

	/**
	 * 리뷰 수정 기능
	 * 리뷰 이미지 파일 업데이트 기능 수정
	 * @param review_idx 수정할 리뷰의 인덱스 값
	 * @param reviewDTO 수정할 리뷰 데이터를 담은 ReviewDTO 객체
	 * @param mv ModelAndView 객체
	 * @param multipartFile 업데이트할 이미지 파일들을 담은 List<MultipartFile> 객체
	 * @return ModelAndView 객체에 view를 설정한 후 반환
	 */
	@PostMapping("update/{review_idx}")
	public ModelAndView updatePost(@PathVariable int review_idx,  ReviewDTO reviewDTO, ModelAndView mv,
								   @RequestParam("file[]") List<MultipartFile> multipartFile){
		reviewDTO.setReview_idx(review_idx);
		int succeessIdx = this.reviewService.update(reviewDTO); // 게시글 업데이트(이미지 제외)
		for (String fileName : this.reviewService.detail(reviewDTO).getR_img()){ // URL주소 하나씩 가져와서
			s3FileUploadService.deleteFromS3(fileName); // 서버에서 삭제
		}
		this.reviewService.deleteImg(reviewDTO); // 기존 img 삭제
		ImgFileUpdate(review_idx, reviewDTO, mv, multipartFile, succeessIdx); // 이미지 파일 업데이트 API
		mv.setViewName("redirect:/review/detail/"+ review_idx);
		return mv;
	}

	/**
	 * 이미지 파일 업데이트 API
	 * @param review_idx 리뷰의 인덱스 값
	 * @param reviewDTO 리뷰 데이터를 담은 ReviewDTO 객체
	 * @param mv ModelAndView 객체
	 * @param multipartFile 업데이트할 이미지 파일들을 담은 List<MultipartFile> 객체
	 * @param succeessIdx 성공적으로 업데이트된 리뷰의 인덱스 값
	 */
	private void ImgFileUpdate(int review_idx, ReviewDTO reviewDTO, ModelAndView mv,
							   List<MultipartFile> multipartFile, int succeessIdx) {
		try {
			if(multipartFile !=null || !multipartFile.isEmpty()) { // 이미지 파일이 존재하는 경우
				List<String> imgList = s3FileUploadService.upload(multipartFile); // 서버에 이미지 저장 후 URL 주소 list에 담기
				reviewDTO.setR_img(imgList);
				reviewDTO.setReview_idx(succeessIdx);
				this.reviewService.updateImg(reviewDTO);
			}
		} catch (IOException e){
			throw new RuntimeException(e);
		}
	}

	/**
	 * 리뷰 삭제
	 * @param review_idx 삭제할 리뷰의 인덱스 값
	 * @param reviewDTO 삭제할 리뷰 데이터를 담은 ReviewDTO 객체
	 * @param mv ModelAndView 객체
	 * @return ModelAndView 객체에 view를 설정한 후 반환
	 */
	@PostMapping("delete/{review_idx}")
	public ModelAndView delete(@PathVariable int review_idx, ReviewDTO reviewDTO, ModelAndView mv) {
		reviewDTO.setReview_idx(review_idx);
		boolean success = this.reviewService.delete(reviewDTO); // 게시글 삭제 (이미지 제외)
		try {
			this.reviewService.updateDeleteImg(reviewDTO); // 이미지 삭제(실제 삭제x)
		} catch (Exception e) {
			// 예외 처리
			System.err.println("리뷰 삭제 중 오류가 발생했습니다: " + e.getMessage());
			// 오류 페이지로 리다이렉트 또는 예외 처리에 맞는 다른 로직 수행
			mv.setViewName("/error/500");
			return mv;
		}
		if (success) {
			mv.setViewName("redirect:/review/list");
		} else {
			mv.setViewName("redirect:/review/detail/" + review_idx);
		}
		return mv;
	}

	/**
	 * 리스트 조회, 검색, 페이징
	 * @param mv ModelAndView 객체
	 * @param cri 페이징을 위한 Criteria 객체
	 * @param reviewDTO 리뷰 데이터를 담은 ReviewDTO 객체
	 * @return ModelAndView 객체에 데이터와 view를 설정한 후 반환
	 */
	@RequestMapping("list")
	public ModelAndView list(ModelAndView mv, Criteria cri, ReviewDTO reviewDTO) {
		try {
			List<ReviewDTO> originalList = reviewService.imglist(reviewDTO);
			List<ReviewDTO> newList = new ArrayList<>(); // 인덱스와 첫번째 이미지만 담을 List 생성
			for (ReviewDTO dto : originalList) {
				List<String> rImgList = dto.getR_img(); // 이미지만 List에 담기
				if (rImgList != null && !rImgList.isEmpty()) { // 이미지가 있는 경우
					String firstImg = rImgList.get(0); // 첫번째 이미지 변수에 담기
					ReviewDTO newDto = new ReviewDTO(); // 인덱스+첫번째 이미지 값 담을 dto
					newDto.setReview_idx(dto.getReview_idx()); // 인덱스 담기
					newDto.setR_img(Collections.singletonList(firstImg)); // 첫번째 이미지 담기
					newList.add(newDto);
				}
			}
			mv.addObject("imgList", newList);
			mv.addObject("paging", reviewService.paging(cri));
			mv.addObject("data", reviewService.list(cri));
			mv.setViewName("/board/review/review_list");
		} catch (Exception e) {
			// 예외 처리
			System.err.println("리뷰 목록 조회 중 오류가 발생했습니다: " + e.getMessage());
			// 오류 페이지로 리다이렉트 또는 예외 처리에 맞는 다른 로직 수행
			mv.setViewName("/error/500");
		}
		return mv;
	}
```
</details>
  
<details>
<summary> <b>이미지 서버에 저장 후 URL 리턴 코드</b> </summary>

  - 클라이언트에서 넘어온 이미지 파일 저장 시 중복을 막히 위해 uuid를 사용하여 파일 이름 변
  - 이미지 파일은 AWS 서버에 저장하고 URL값 리턴 
  
```java
@Service
public class S3FileUploadService {

    @Autowired
    private final AmazonS3Client amazonS3Client; //아마존 계정정보 propertie파일 -> common-context에서 주입
    @Value("${aws.s3.bucket}")
    private String bucket; //S3버킷정보
    @Value("${aws.s3.bucket.url}") //지역정보
    private String defaultUrl;

    public S3FileUploadService(AmazonS3Client amazonS3Client) {
        this.amazonS3Client = amazonS3Client;
    }

    //생성자 주입
    public List<String> upload(List<MultipartFile> uploadFile) throws IOException {
        List<String> urlList = new ArrayList<>(); //업로드된 url을 받기위한 리스트

        //파일이름 새로만들어서 리스트에 담기
        List<Map<String, String>> fileList = new ArrayList<>();
        for (int i = 0; i < uploadFile.size(); i++) {
            String origName = uploadFile.get(i).getOriginalFilename(); //원 파일이름
            String ext = origName.substring(origName.lastIndexOf('.')); // 확장자
            String saveFileName = getUuid() + ext; //uuid로 새이름 만들기
            Map<String, String> map = new HashMap<>();
            map.put("saveFile", saveFileName);
//            System.out.println("s3맵 : " + saveFileName );
            fileList.add(map);
        }

        for (int i = 0; i < uploadFile.size(); i++) {
            String url = "";
            File file = new File(System.getProperty("user.dir") + fileList.get(i).get("saveFile"));
            //로컬 현재위치에 임시저장 객체 만듬
            uploadFile.get(i).transferTo(file); //로컬에 파일 임시저장
            uploadOnS3(fileList.get(i).get("saveFile"), file); //업로드
            url = defaultUrl + '/' + fileList.get(i).get("saveFile"); //업로드한 파일의 url주소
            urlList.add(url); //리턴을 위해 담음
            file.delete(); // 임시파일 삭제
        }
        return urlList; //업로드 후 리턴값 (List<String> 타입)
    }

    // UUID만드는 메소드(중간의-는 지워줌)
    private static String getUuid() {
        return UUID.randomUUID().toString().replaceAll("-", "");
    }

    //S3업로드 메소드
    private void uploadOnS3(final String findName, final File file) {
        // AWS S3 전송 객체 생성
        final TransferManager transferManager = new TransferManager(this.amazonS3Client);
        // 요청 객체 생성
        final PutObjectRequest request = new PutObjectRequest(bucket, findName, file);
        // 업로드 시도
        final Upload upload = transferManager.upload(request);

        try {
            upload.waitForCompletion();
        } catch (AmazonClientException | InterruptedException amazonClientException) {
            amazonClientException.printStackTrace();
        }
    }
```
  </br>
  </details>

<details>
  <summary><b>검색-조회 페이징 DTO 코드</b></summary>  

 ```java
  public class Criteria {
    private int page; // 현재 페이지 번호
    private int perPageNum; // 한 페이지당 보여줄 게시글의 개수
    private String category; // 카테고리 분류를 위한 변수
    private String keyword; // 검색 키워드
    private String option; // 검색 옵션
    private String auth; // 권한 정보

    public int getPageStart() {
        return (page - 1) * perPageNum; // 특정 페이지의 게시글 시작 번호(게시글 시작 행 번호)
    }

    public Criteria() {
        page = 1; // 페이지 번호를 1로 초기화
        perPageNum = 10; // 한 페이지당 보여줄 게시글의 개수를 10으로 초기화
    }

    public int getPage() {
        return page;
    }

    public void setPage(int page) {
        // 페이지 번호를 설정하는 메서드
        // 페이지 번호가 음수일 경우 1로 설정하여 음수값이 되지 않도록 처리
        if (page <= 0) {
            this.page = 1;
        } else {
            this.page = page;
        }
    }

    public int getPerPageNum() {
        return perPageNum;
    }

    public void setPerPageNum(int pageCount) {
        // 페이지당 보여줄 게시글 수가 변하지 않도록 설정
        // 만약 전체 페이지 수보다 큰 값을 입력한 경우 전체 페이지 수로 설정하여 오류를 방지
        int cnt = perPageNum;
        if (pageCount != cnt) {
            perPageNum = cnt;
        } else {
            perPageNum = pageCount;
        }
    }

    public String getCategory() {
        return category;
    }

    public void setCategory(String category) {
        this.category = category;
    }
	
    // 검색 기능  
    public String getKeyword() {
        if (keyword != null && !keyword.isEmpty()) {
            return keyword;
        }
        return "";
    }

    public void setKeyword(String keyword) {
        this.keyword = keyword;
    }

    public String getOption() {
        return option;
    }

    public void setOption(String option) {
        this.option = option;
    }

    public String getAuth() {
        return auth;
    }

    public void setAuth(String auth) {
        this.auth = auth;
    }

    @Override
    public String toString() {
        return "Criteria [page=" + page 
                + ", perPageNum=" + perPageNum 
                + ", category=" + category 
                + ", keyword=" + keyword 
                + ", option=" + option 
                + ", auth=" + auth 
                + "]";
    }
}              
```

</details>

<details>
  <summary><b>페이징 DTO 코드</b></summary>

```java
public class PagingDTO {
    private Criteria cri; // 게시글 조회 조건을 담은 Criteria 인스턴스
    private int totalCount; // 전체 게시글 수
    private int startPage; // 페이징의 시작 페이지 번호
    private int endPage; // 페이징의 끝 페이지 번호
    private boolean prev; // 이전 페이지 버튼 표시 여부
    private boolean next; // 다음 페이지 버튼 표시 여부
    private int displayPageNum = 5; // 사용자에게 보여질 페이지 번호 갯수

    public Criteria getCri() {
        return cri;
    }

    public void setCri(Criteria cri) {
        this.cri = cri;
    }

    public int getTotalCount() {
        return totalCount;
    }

    public void setTotalCount(int totalCount) {
        this.totalCount = totalCount;
        calcData(); // 페이징 관련 데이터 계산
    }

    private void calcData() {
        // 페이징 관련 데이터 계산 메서드

        endPage = (int) (Math.ceil(cri.getPage() / (double) displayPageNum) * displayPageNum);
        // 끝 페이지 번호 = (현재 페이지 번호 / 화면에 보여질 페이지 번호의 갯수) * 화면에 보여질 페이지 번호의 갯수

        startPage = (endPage - displayPageNum) + 1;
        if (startPage <= 0) {
            startPage = 1;
        }
        // 시작 페이지 번호 = (끝 페이지 번호 - 화면에 보여질 페이지 번호의 갯수) + 1
        // 만약 시작 페이지가 음수일 경우 1로 대체
        // 예시: 끝 페이지 번호(10) - 페이지 번호의 갯수(5) + 1 -> 계산식 -> 6페이지 시작

        int tempEndPage = (int) (Math.ceil(totalCount / (double) cri.getPerPageNum()));
        if (endPage > tempEndPage) {
            endPage = tempEndPage;
        }
        // 마지막 페이지 번호 = 총 게시글 수 / 한 페이지당 보여줄 게시글의 갯수

        prev = startPage == 1 ? false : true;
        // 이전 페이지 버튼 표시 여부
        // 시작 페이지 번호가 1인 경우에는 이전 버튼을 표시하지 않음

        next = endPage * cri.getPerPageNum() < totalCount ? true : false;
        // 다음 페이지 버튼 표시 여부
        // 끝 페이지 번호 * 한 페이지당 보여줄 게시글의 갯수 < 총 게시글 수인 경우에는 다음 버튼을 표시
    }

    public int getStartPage() {
        return startPage;
    }

    public void setStartPage(int startPage) {
        this.startPage = startPage;
    }

    public int getEndPage() {
        return endPage;
    }

    public void setEndPage(int endPage) {
        this.endPage = endPage;
    }

    public boolean isPrev() {
        return prev;
    }

    public void setPrev(boolean prev) {
        this.prev = prev;
    }

    public boolean isNext() {
        return next;
    }

    public void setNext(boolean next) {
        this.next = next;
    }

    public int getDisplayPageNum() {
        return displayPageNum;
    }

    public void setDisplayPageNum(int displayPageNum) {
        this.displayPageNum = displayPageNum;
    }

    @Override
    public String toString() {
        return "PagingDTO{" +
                ", cri=" + cri +
                ", totalCount=" + totalCount +
                ", startPage=" + startPage +
                ", endPage=" + endPage +
                ", prev=" + prev +
                ", next=" + next +
                ", displayPageNum=" + displayPageNum +
                '}';
    }
```  
  </details>
  
<details>
<summary> <b>Service & Service Implements</b> </summary>

  - 컨트롤러에서 받은 정보들을 비지니스 로직 처리 
```java
@Service
public class ReviewServiceImpl implements ReviewService {

	@Autowired
	ReviewDAO reviewDAO;

	/**
	 * 리뷰 생성
	 * @param reviewDTO 리뷰 정보
	 * @return 생성된 리뷰의 review_idx 값
	 */
	@Override
	public int create(ReviewDTO reviewDTO) {
		int affectRowCnt =  this.reviewDAO.create(reviewDTO); // 게시글 생성된 갯수 반환
		if(affectRowCnt==1){
			System.out.println("review_idx : " + reviewDTO.getReview_idx());
			return reviewDTO.getReview_idx();
		}
		return 0;
	}

	/**
	 * 리뷰 이미지 생성
	 * @param reviewDTO 리뷰 정보
	 */
	@Override
	public void createImg(ReviewDTO reviewDTO) {
		this.reviewDAO.createImg(reviewDTO);
	}

	/**
	 * 리뷰 상세 정보 조회
	 * @param reviewDTO
	 * @return 리뷰 상세 정보
	 */
	@Override
	public ReviewDTO detail(ReviewDTO reviewDTO) {
		return this.reviewDAO.detail(reviewDTO);
	}

	/**
	 * 리뷰 업데이트
	 * @param reviewDTO
	 * @return 업데이트 된 리뷰 인덱스값
	 */
	@Override
	public int update(ReviewDTO reviewDTO) {
		int cnt = this.reviewDAO.update(reviewDTO);
		if(cnt==1){
			return reviewDTO.getReview_idx();
		}
		return 0;
	}

	/**
	 * 리뷰 이미지 업데이트
	 * @param reviewDTO 업데이트 할 리뷰 정보
	 */
	@Override
	public void updateImg(ReviewDTO reviewDTO) {
		this.reviewDAO.updateImg(reviewDTO);
	}

	/**
	 * 리뷰 삭제
	 * @param reviewDTO
	 * @return 삭제 성공 여부 (true : 성공, fasle : 실패)
	 */
	@Override
	public boolean delete(ReviewDTO reviewDTO) {
		int cnt = this.reviewDAO.delete(reviewDTO);
		if (cnt==1){
			return true;
		}
		return false;
	}

	/**
	 * 리뷰 이미지 삭제
	 * @param reviewDTO 삭제할 리뷰 정보
	 */
	@Override
	public void deleteImg(ReviewDTO reviewDTO) {
		this.reviewDAO.deleteImg(reviewDTO);
	}

	/**
	 * 리뷰 목록
	 * @param cri 조회 조건(페이징 정보 포함)
	 * @return 리뷰 목록
	 */
	@Override
	public List<ReviewDTO> list(Criteria cri) {
		return reviewDAO.list(cri);
	}

	/**
	 페이징 처리
	 @param cri 페이징 조건을 담고 있는 Criteria 객체
	 @return 페이징 처리를 위한 정보를 담고 있는 PagingDTO 객체
	 */
	@Override
	public PagingDTO paging(Criteria cri) {
		PagingDTO paging = new PagingDTO(); // 페이징 정보를 담을 객체 생성
		paging.setCri(cri); // 페이징 조건 설정
		paging.setTotalCount(reviewDAO.totalCount(cri)); // 전체 게시물 수 설정
		return paging; // 페이징 정보 반환
	}

	/**
	 * 조건에 맞는 리뷰 이미지 목록
	 * @param reviewDTO 조회 조건
	 * @return 리뷰 이미지 목록
	 */
	@Override
	public List<ReviewDTO> imglist(ReviewDTO reviewDTO) {
		return this.reviewDAO.imgList(reviewDTO);
	}

	/**
	 * 리뷰 이미지 삭제 
	 * @param reviewDTO 삭제할 리뷰 정보
	 */
	@Override
	public void updateDeleteImg(ReviewDTO reviewDTO) {
		this.reviewDAO.updateDeleteImg(reviewDTO);
	}
```
</details>
  
<details>
<summary> <b>DAO</b> </summary>

  - 서비스에서 받은 비지니스 로직을 데이터 베이스로 연결
  
```java
public class ReviewDAO {

	@Autowired
	SqlSession ss;
	
	// 리뷰 생성
	public int create(ReviewDTO reviewDTO) {
		return this.ss.insert("review.insert", reviewDTO);
	}
	// 리뷰 이미지 생성
	public void createImg(ReviewDTO reviewDTO) {
		this.ss.insert("review.insertFile", reviewDTO);
	}
	// 리뷰 상세 정보 호출
	public ReviewDTO detail(ReviewDTO reviewDTO) {
		return this.ss.selectOne("review.detail", reviewDTO);
	}
	// 리뷰 업데이트
	public int update(ReviewDTO reviewDTO) {
		return this.ss.update("review.update", reviewDTO);
	}
	// 리뷰 이미지 삭제
	public void deleteImg(ReviewDTO reviewDTO) {
		this.ss.delete("review.deleteImg", reviewDTO);
	}
 	// 리뷰 이미지 업데이트 
	public void updateImg(ReviewDTO reviewDTO) {
		this.ss.update("review.updateImg", reviewDTO);
	}
	// 리뷰 삭제
	public int delete(ReviewDTO reviewDTO) {
		return this.ss.update("review.delete", reviewDTO);
	}
	// 리뷰 목록 호출
	public List<ReviewDTO> list(Criteria cri) {
		return this.ss.selectList("review.list", cri);
	}
	// 리뷰 토탈 게시물 수 
	public int totalCount(Criteria cri) {
		return this.ss.selectOne("review.totalCount",cri);
	}
	// 이미지 목록 호출
	public List<ReviewDTO> imgList(ReviewDTO reviewDTO) {
		return this.ss.selectList("review.imglist", reviewDTO);
	}
	// 리뷰 이미지 업데이트 
	public void updateDeleteImg(ReviewDTO reviewDTO) {
		this.ss.update("review.updateDeleteImg", reviewDTO);
	}
	// 플랜 정보 호출
	public PlanDTO getCreate(PlanDTO planDTO) {
		return this.ss.selectOne("review.getCreate", planDTO);
	}
```
</details>
  
<details>
<summary> <b>XML</b> </summary>

```java
    <!--리뷰 작성 정보-->
    <select id="getCreate" resultMap="planResultMap">
        select p.plan_idx, p.user_id, p.plan_title, p.start_date,
               p.end_date, p.price, p.plan_detail, i.p_img
          from plan p left join plan_img i
            on p.plan_idx = i.plan_idx
         where p.plan_idx = #{plan_idx}
           and p.p_del_yn = 'n'
    </select>

    <resultMap id="planResultMap" type="com.goott.pj3.plan.dto.PlanDTO">
        <id property="plan_idx" column="plan_idx"/>
        <result property="user_id" column="user_id"/>
        <result property="start_date" column="start_date"/>
        <result property="end_date" column="end_date"/>
        <result property="price" column="price"/>
        <result property="plan_detail" column="plan_detail"/>
        <result property="plan_title" column="plan_title"/>
        <collection property="p_img" column="p_img" javaType="List" ofType="String">
            <result column="p_img"/>
        </collection>
    </resultMap>

    <!--리뷰 생성-->
    <insert id="insert" parameterType="com.goott.pj3.board.review.dto.ReviewDTO"
            useGeneratedKeys="true" keyProperty="review_idx">
        insert into plan_review(user_id, plan_idx, review_content, planner_rating)
        values (#{user_id}, #{plan_idx}, #{review_content}, #{planner_rating})
    </insert>

    <!--리뷰 이미지 파일 생성-->
    <insert id="insertFile" parameterType="com.goott.pj3.board.review.dto.ReviewDTO">
            insert into review_img(review_idx, r_img) values
            <foreach collection="r_img" item="img" separator=",">
                 (#{review_idx}, #{img})
            </foreach>
    </insert>

    <!--디테일 페이지-->
    <select id="detail" resultMap="reviewResultMap">
        select r.review_idx, r.user_id, r.plan_idx, r.review_content, r.planner_rating, r.create_date,
               i.r_img, i.r_img_idx
         from plan_review r left join review_img i
           on r.review_idx = i.review_idx
        where r.review_idx = #{review_idx}
          and r.r_del_yn='n'
          and i.r_img_del_yn='n'
    </select>

    <resultMap id="reviewResultMap" type="com.goott.pj3.board.review.dto.ReviewDTO">
        <id property="review_idx" column="review_idx"/>
        <result property="user_id" column="user_id"/>
        <result property="plan_idx" column="plan_idx"/>
        <result property="review_content" column="review_content"/>
        <result property="planner_rating" column="planner_rating"/>
        <result property="create_date" column="create_date"/>
        <collection property="r_img" column="r_img" javaType="java.util.List" ofType="String" >
            <result column="r_img"/>
        </collection>
        <collection property="r_img_idx" column="r_img_idx" javaType="List" ofType="String">
            <result column="r_img_idx" />
        </collection>
    </resultMap>

    <!--리뷰 수정 시작-->
    <!--본문 내용 수정-->
    <update id="update" parameterType="com.goott.pj3.board.review.dto.ReviewDTO"
            useGeneratedKeys="true" keyProperty="review_idx">
        update plan_review
           set review_content=#{review_content}
         where review_idx = #{review_idx}
    </update>

    <!--기존 이미지 파일 삭제-->
    <delete id="deleteImg" parameterType="com.goott.pj3.board.review.dto.ReviewDTO">
        delete
          from review_img
         where review_idx=#{review_idx}
    </delete>

    <!--새 이미지 파일 생성-->
    <insert id="updateImg" parameterType="com.goott.pj3.board.review.dto.ReviewDTO">
        insert into review_img(review_idx, r_img)
             values
                    <foreach collection="r_img" item="img" separator=",">
                        (#{review_idx}, #{img})
                    </foreach>
    </insert>
    <!--리뷰 수정 끝 -->

    <!--리뷰 삭제-->
    <update id="delete" parameterType="com.goott.pj3.board.review.dto.ReviewDTO">
        update plan_review
          set r_del_yn = 'y'
        where review_idx=#{review_idx}
    </update>

    <!--리뷰 이미지 업데이트-->
    <update id="updateDeleteImg">
        update review_img
           set r_img_del_yn='y'
         where review_idx=#{review_idx}
    </update>

    <!--리뷰 목록, 검색-->
    <select id="list" resultType="com.goott.pj3.board.review.dto.ReviewDTO">
        select review_idx, user_id, plan_idx, review_content,
               planner_rating, create_date, update_date
          from plan_review
         where 
		   	<if test="option == 'user_id'"> user_id like CONCAT('%',#{keyword},'%')</if>
	        <if test="option == 'content'"> review_content like CONCAT('%',#{keyword},'%')</if>
	        <if test="option == null or option == ''">1=1</if>
           and r_del_yn = 'y'
         order by review_idx desc
         limit #{pageStart}, #{perPageNum}
    </select>

    <!--페이지 전체 개수-->
    <select id="totalCount" resultType="int">
        select count(review_idx)
        from plan_review
        where
        <if test="option == 'user_id'"> user_id like CONCAT('%',#{keyword},'%')</if>
        <if test="option == 'content'"> review_content like CONCAT('%',#{keyword},'%')</if>
        <if test="option == null or option == ''">1=1</if>
        and r_del_yn = 'n'
    </select>

    <!--리뷰 이미지 목록-->
    <select id="imglist" resultMap="reviewlistResultMap">
        select i.review_idx, i.r_img
          from review_img i left join plan_review r
            on i.review_idx = r.review_idx
         where r.r_del_yn='n'
           and i.r_img_del_yn='n'
         order by r.review_idx desc
    </select>

    <resultMap id="reviewlistResultMap" type="com.goott.pj3.board.review.dto.ReviewDTO">
        <id property="review_idx" column="review_idx"/>
        <collection property="r_img" column="r_img" javaType="List" ofType="String">
            <result column="r_img"/>
        </collection>
```
</details>

---
### 4.3. 여행 정보 게시판 CRUD

<details>
<summary> <b>Controller</b> </summary>

```java
  @Controller
@RequestMapping("/travelinfo/**")
public class TravelInfoController {

	@Autowired
	TravelInfoService travelInfoService;

	// AWS S3 파일 업로드
	@Autowired
	S3FileUploadService s3FileUploadService;

	/**
	 * 여행지 정보 생성 페이지 호출
	 * 페이지 URL: /create
	 * GET 요청
	 * @return ModelAndView 객체를 통해 여행지 정보 뷰에 반환
	 */
	@GetMapping("create")
	public ModelAndView create() {
		ModelAndView mv = new ModelAndView();
		mv.setViewName("/travelinfo/travelinfo_create");
		return mv;
	}
	/**
	 * 여행지 정보 생성
	 * 이미지 파일 업로드 기능 추가
	 * @param travelInfoDTO    여행지 정보를 담은 TravelInfoDTO 객체
	 * @param mv               ModelAndView 객체
	 * @param httpSession      현재의 HttpSession 객체
	 * @param multipartFile    업로드된 이미지 파일 리스트
	 * @return ModelAndView 객체를 통해 처리 결과에 따른 뷰 이름을 설정하여 반환
	 */
	@PostMapping("create")
	public ModelAndView createPost(TravelInfoDTO travelInfoDTO, ModelAndView mv, HttpSession httpSession,
								   @RequestParam("file[]") List<MultipartFile> multipartFile) {
		try {
		// 현재 로그인한 유저의 아이디를 세션
		String user_id = (String) httpSession.getAttribute("user_id");
		// DTO에 유저 아이디 할당
		travelInfoDTO.setUser_id(user_id);
		// 여행지 정보를 생성, 생성된 게시글의 인덱스 리턴
		int travel_location_idx = this.travelInfoService.create(travelInfoDTO);
		// 이미지 파일 업로드를 수행합니다.
		if (travel_location_idx != 0) {
			// 이미지 파일 업로드
			imgFileUpload(travelInfoDTO, multipartFile, travel_location_idx);
			mv.setViewName("redirect:/travelinfo/detail/" + travel_location_idx);
		}
		else {
			mv.setViewName("travelinfo/travelinfo_create");
		}
		} catch (Exception e) {
			// 에러 처리
			e.printStackTrace(); // 에러 로그 출력
		}
		return mv;
	}
	/**
	 * 이미지 파일 업로드 API
	 * @param travelInfoDTO 여행지 정보 DTO
	 * @param multipartFile 업로드할 이미지 파일 리스트
	 * @param travel_location_idx 생성된 여행지 게시글의 인덱스
	 */
	private void imgFileUpload(TravelInfoDTO travelInfoDTO, List<MultipartFile> multipartFile, int travel_location_idx) {
		try {
			if (multipartFile != null && !multipartFile.isEmpty()) {
				// 이미지 파일이 존재하는 경우
				List<String> imgList = s3FileUploadService.upload(multipartFile);
				travelInfoDTO.setT_img(imgList);
				travelInfoDTO.setTravel_location_idx(travel_location_idx);
				this.travelInfoService.createImg(travelInfoDTO);
			} else {
				// 이미지 파일이 없는 경우
				travelInfoDTO.setTravel_location_idx(travel_location_idx);
				this.travelInfoService.createImg(travelInfoDTO);
			}
		} catch (IOException e) {
			throw new RuntimeException(e);
		}
	}
	/**
	 * 여행지 정보 디테일 페이지 호출
	 * @param travel_location_idx 조회할 여행지 게시글의 인덱스
	 * @param travelInfoDTO 여행지 정보 DTO
	 * @param mv ModelAndView 객체
	 * @return ModelAndView 여행 상세 정보 뷰로 리턴
	 */
	@GetMapping("detail/{travel_location_idx}")
	public ModelAndView detail(@PathVariable int travel_location_idx,
							   TravelInfoDTO travelInfoDTO, ModelAndView mv) {
		// 여행 정보 인덱스 할당
		travelInfoDTO.setTravel_location_idx(travel_location_idx);
		// 여행 정보 조회
		TravelInfoDTO detail = this.travelInfoService.detail(travelInfoDTO);
		mv.addObject("data", detail);
		mv.setViewName("travelinfo/travelinfo_detail");
		return mv;
	}

	/**
	 * 여행지 정보 수정 페이지 호출
	 * @param travel_location_idx 수정할 여행 정보 인덱스
	 * @param travelInfoDTO 여행지 정보 DTO
	 * @param mv ModelAndView
	 * @return ModelAndView 기존 여행 상세 정보 리턴
	 */
	@GetMapping("update/{travel_location_idx}")
	public ModelAndView update(@PathVariable int travel_location_idx,
							   TravelInfoDTO travelInfoDTO, ModelAndView mv) {
		// 조회 할 인덱스 번호 할당
		travelInfoDTO.setTravel_location_idx(travel_location_idx);
		// 여행 정보 게시글 리턴
		TravelInfoDTO detail = this.travelInfoService.detail(travelInfoDTO);
		mv.addObject("data", detail); // 게시글 정보
		mv.setViewName("travelinfo/travelinfo_update");
		return mv;
	}

	/**
	 * 여행지 정보 수정
	 * @param travel_location_idx 수정할 여행지 게시글의 인덱스
	 * @param travelInfoDTO 여행지 정보 DTO
	 * @param mv ModelAndView 객체
	 * @param multipartFile 업데이트할 이미지 파일 리스트
	 * @return ModelAndView 수정된 상세 페이지 리턴
	 */
	@PostMapping("update/{travel_location_idx}")
	public ModelAndView updatePost(@PathVariable int travel_location_idx, TravelInfoDTO travelInfoDTO,
								   ModelAndView mv,
								   @RequestParam("file[]") List<MultipartFile> multipartFile) {
		// 조회 할 인덱스 할당
		travelInfoDTO.setTravel_location_idx(travel_location_idx);
		// 본문 내용 업데이트 (이미지 제외) 및 성공한 인덱스 반환
		int succeessIdx = this.travelInfoService.update(travelInfoDTO);
		// 서버 기존 이미지 파일 삭제
		for(String fileName : this.travelInfoService.detail(travelInfoDTO).getT_img()){ // URL주소 하나씩 가져와서
			s3FileUploadService.deleteFromS3(fileName); // 서버에서 삭제
		}
		// 기존 이미지 URL 주소 DB에서 삭제
		boolean success = this.travelInfoService.deleteImg(travelInfoDTO);
		// 이미지 파일 업데이트
		ImgFileUpdate(travelInfoDTO, mv, multipartFile, succeessIdx);
		mv.setViewName("redirect:/travelinfo/detail/" + travel_location_idx);
		return mv;
	}

	/**
	 * 이미지 파일 업데이트 API
	 * @param travelInfoDTO 여행지 정보 DTO
	 * @param mv ModelAndView 객체
	 * @param multipartFile 업데이트할 이미지 파일 리스트
	 * @param successIdx 성공한 인덱스
	 */
	private void ImgFileUpdate(TravelInfoDTO travelInfoDTO,
							   ModelAndView mv, List<MultipartFile> multipartFile, int successIdx) {
		try {
			if (multipartFile !=null || !multipartFile.isEmpty()) {
				List<String> imgList = s3FileUploadService.upload(multipartFile);
				travelInfoDTO.setT_img(imgList);
				travelInfoDTO.setTravel_location_idx(successIdx);
				this.travelInfoService.updateImg(travelInfoDTO);
			}
		} catch (IOException e) {
			throw new RuntimeException(e);
		}
	}

	/**
	 * 여행지 정보 삭제
	 * @param travel_location_idx 여행지 게시글의 인덱스
	 * @param travelInfoDTO 여행지 정보 DTO
	 * @param mv ModelAndView 객체
	 * @return 여행지 게시글
	 */
	@PostMapping("delete/{travel_location_idx}")
	public ModelAndView delete(@PathVariable int travel_location_idx,
							   TravelInfoDTO travelInfoDTO, ModelAndView mv) {
		travelInfoDTO.setTravel_location_idx(travel_location_idx);
		boolean success = this.travelInfoService.delete(travelInfoDTO); // 게시글 삭제(이미지 제외)
		try {
			TravelInfoDTO detailDTO = this.travelInfoService.detail(travelInfoDTO); // 리뷰 상세 정보 가져오기
			if (detailDTO != null && detailDTO.getT_img() != null){
				for(String fileName : detailDTO.getT_img()){
					s3FileUploadService.deleteFromS3(fileName);
				}
			}
			this.travelInfoService.deleteImg(travelInfoDTO); // 이미지 삭제
		} catch (Exception e) {
			//예외 처리
			System.out.println("여행정보 삭제 중 오류가 발생했습니다 : " + e.getMessage());
			mv.setViewName("/error/500");
		}
		if (success) {
			mv.setViewName("redirect:/travelinfo/list");
		} else {
			mv.setViewName("redirect:/travelinfo/detail/" + travel_location_idx);
		}
		return mv;
	}

	/**
	 * 리스트 조회, 검색, 페이징
	 * @param mv ModelAndView 객체
	 * @param cri 페이징 및 검색 조건을 담은 Criteria 객체
	 * @param travelInfoDTO 여행지 정보 DTO
	 * @return 리스트 조회 결과 리턴
	 */
	@RequestMapping("list")
	public ModelAndView list(ModelAndView mv, Criteria cri, TravelInfoDTO travelInfoDTO) {
		try {
			List<TravelInfoDTO> originalList = travelInfoService.imgList(travelInfoDTO);
			System.out.println("originalList : " + originalList);
			List<TravelInfoDTO> newList = new ArrayList<>(); // 인덱스와 첫번째 이미지만 담을 List 생생
			for (TravelInfoDTO dto : originalList) {
				List<String> tImgList = dto.getT_img(); // 이미지만 List에 담기
				if (tImgList != null && !tImgList.isEmpty()) { // 이미지가 있는 경우
					String firstImg = tImgList.get(0); // 첫번째 이미지 변수에 담기
					TravelInfoDTO newDto = new TravelInfoDTO(); // 인덱스+첫번째 이미지 값 담을 dto
					newDto.setTravel_location_idx(dto.getTravel_location_idx()); // 인덱스 담기
					newDto.setT_img(Collections.singletonList(firstImg)); // 첫번째 이미지 담기
					newList.add(newDto);
				}
			}
			System.out.println("newList" + newList);
			mv.addObject("imgList", newList);
			mv.addObject("paging", travelInfoService.paging(cri));
			mv.addObject("data", travelInfoService.list(cri));
			mv.setViewName("/travelinfo/travelinfo_list");
		} catch (Exception e) {
			// 예외 처리
			System.out.println("여행정보 목록 조회 중 오류가 발생했습니다 : " + e.getMessage());
			mv.setViewName("error/500");
		}
		return mv;
	}
}
```

</details>

<details>
<summary> <b>Service & Service Implements</b> </summary>

```java
@Service
public class TravelInfoServiceImpl implements TravelInfoService {

    @Autowired
    TravelInfoDAO travelInfoDAO;

    /**
     * 여행 정보 생성
     * @param travelInfoDTO 여행 정보 생성 정보
     * @return 생성된 게시글 인덱스
     */
    @Override
    public int create(TravelInfoDTO travelInfoDTO) {
        int affectRowCnt = this.travelInfoDAO.create(travelInfoDTO);
        if(affectRowCnt==1){
            return travelInfoDTO.getTravel_location_idx();
        }
        return 0;
    }

    /**
     * 여행 정보 이미지 생성
     * @param travelInfoDTO 여행 정보 이미지 정보
     */
    @Override
    public void createImg(TravelInfoDTO travelInfoDTO) {
        this.travelInfoDAO.createImg(travelInfoDTO);
    }

    /**
     * 여행 정보 상세 페이지 호출
     * @param travelInfoDTO 상세 페이지 정보
     * @return 상세 페이지 정보 리턴
     */
    @Override
    public TravelInfoDTO detail(TravelInfoDTO travelInfoDTO) {
        return this.travelInfoDAO.detail(travelInfoDTO);
    }

    /**
     * 여행 정보 업데이트
     * @param travelInfoDTO 업데이트 할 정보
     * @return 업데이트 된 게시글 인덱스
     */
    @Override
    public int update(TravelInfoDTO travelInfoDTO) {
        int cnt = this.travelInfoDAO.update(travelInfoDTO);
        if(cnt==1){
            return travelInfoDTO.getTravel_location_idx();
        }
        return 0;
    }

    /**
     * 여행 정보 이미지 삭제
     * @param travelInfoDTO 삭제할 게시글 정보
     * @return 삭제 성공 여부
     */
    @Override
    public boolean deleteImg(TravelInfoDTO travelInfoDTO) {
        int affectRowCnt = this.travelInfoDAO.deleteImg(travelInfoDTO);
        if(affectRowCnt!=0){
            return true;
        }
        return false;
    }

    /**
     * 여행 정보 이미지 업데이트
     * @param travelInfoDTO  업데이트 할 게시글 정보
     */
    @Override
    public void updateImg(TravelInfoDTO travelInfoDTO) {
        this.travelInfoDAO.updateImg(travelInfoDTO);
    }

    /**
     * 여행 정보 게시글 삭제
     * @param travelInfoDTO 삭제 할 게시글 정보 
     * @return 삭제 성공 여부 리턴
     */
    @Override
    public boolean delete(TravelInfoDTO travelInfoDTO) {
        int cnt = this.travelInfoDAO.delete(travelInfoDTO);
        return cnt==1;
    }

    /**
     * 이미지 목록 호출
     * @param travelInfoDTO 이미지 목록 호출하기 위한 게시글 정보  
     * @return 이미지 목록 리턴
     */
    @Override
    public List<TravelInfoDTO> imgList(TravelInfoDTO travelInfoDTO) {
        return this.travelInfoDAO.imgList(travelInfoDTO);
    }

    /**
     * 게시글 목록 조회
     * @param cri 게시글 조회 할 정보   
     * @return 게시글 목록 리턴
     */
    @Override
    public List<TravelInfoDTO> list(Criteria cri) {
        return travelInfoDAO.list(cri);
    }

    /**
     * 게시글 페이징 
     * @param cri 페이징 할 정보
     * @return 페이징 정보 리턴
     */
    @Override
    public PagingDTO paging(Criteria cri) {
        PagingDTO paging = new PagingDTO();
        paging.setCri(cri);
        paging.setTotalCount(travelInfoDAO.totalCount(cri));
        return paging;
    }

}
```
</details>
 
<details>
<summary> <b>DAO</b> </summary>

```java
  @Repository
public class TravelInfoDAO {

    @Autowired
    SqlSession ss;

    // 여행 정보 게시글 생성
   public int create(TravelInfoDTO travelInfoDTO) {
        System.out.println("DAO : " + travelInfoDTO);
        return this.ss.insert("travelinfo.insert", travelInfoDTO);
    }
    // 여행 정보 이미지 생성
    public void createImg(TravelInfoDTO travelInfoDTO) {
        System.out.println("DAOImg : " + travelInfoDTO.toString());
        this.ss.insert("travelinfo.insertfile", travelInfoDTO);
    }
    // 여행 정보 상세 정보 호출
    public TravelInfoDTO detail(TravelInfoDTO travelInfoDTO) {
        return this.ss.selectOne("travelinfo.detail", travelInfoDTO);
    }
    // 여행 정보 게시글 업데이트
    public int update(TravelInfoDTO travelInfoDTO) {
        return this.ss.update("travelinfo.update", travelInfoDTO);
    }
    // 여행 정보 기존 이미지 삭제
    public int deleteImg(TravelInfoDTO travelInfoDTO) {
        System.out.println("deleteImgDAO: " + travelInfoDTO);
        return this.ss.delete("travelinfo.deleteImg", travelInfoDTO);
    } 
    // 여행 정보 이미지 업데이트
    public void updateImg(TravelInfoDTO travelInfoDTO) {
        System.out.println("udateImg DAO : " + travelInfoDTO);
        this.ss.insert("travelinfo.updateImg", travelInfoDTO);
    }
    // 게시글 삭제
    public int delete(TravelInfoDTO travelInfoDTO) {
        return this.ss.update("travelinfo.delete", travelInfoDTO);
    }
    // 이미지 목록 조회
    public List<TravelInfoDTO> imgList(TravelInfoDTO travelInfoDTO) {
        return this.ss.selectList("travelinfo.imgList", travelInfoDTO);
    }
    // 게시글 목록 조회
    public List<TravelInfoDTO> list(Criteria cri) {
        return ss.selectList("travelinfo.list", cri);
    }
    // 게시글 전체 수 
    public int totalCount(Criteria cri) {
        return ss.selectOne("travelinfo.totalCount", cri);
    }
}
```
</details>
  
<details>
<summary> <b>XML</b> </summary>

```java
<mapper namespace="travelinfo">

    <!--여행지 정보 생성-->
    <insert id="insert" parameterType="com.goott.pj3.travelinfo.dto.TravelInfoDTO"
            useGeneratedKeys="true" keyProperty="travel_location_idx">
        insert into travel_location(user_id, country_a, country_b, country_c,
                                    country_detail, country_script)
             values ("test1234", #{country_a}, #{country_b},
                     #{country_c}, #{country_detail}, #{country_script})
    </insert>
    <!--여행지 정보 이미지 생성-->
    <insert id="insertfile" parameterType="com.goott.pj3.travelinfo.dto.TravelInfoDTO">
        insert into travel_img(travel_location_idx, t_img) values
        <foreach collection="t_img" item="img" separator=",">
            (#{travel_location_idx}, #{img})
        </foreach>
    </insert>

    <!--여행지 정보 디테일 페이지 데이터 조회-->
    <select id="detail" resultMap="travelInfoResultMap">
        select t.travel_location_idx, t.user_id, t.country_a, t.country_b, t.country_c,
               t.country_detail, t.country_script, t.create_date, t.update_date,
               i.t_img_idx, i.t_img
          from travel_location t left join travel_img i
            on t.travel_location_idx = i.travel_location_idx
         where t.travel_location_idx = #{travel_location_idx}
           and t.t_del_yn='n'
    </select>

    <resultMap id="travelInfoResultMap" type="com.goott.pj3.travelinfo.dto.TravelInfoDTO">
        <id property="travel_location_idx" column="travel_location_idx"/>
        <result property="user_id" column="user_id" />
        <result property="country_a" column="country_a" />
        <result property="country_b" column="country_b" />
        <result property="country_c" column="country_c" />
        <result property="country_detail" column="country_detail" />
        <result property="country_script" column="country_script" />
        <result property="create_date" column="create_date" />
        <result property="update_date" column="update_date" />
        <collection property="t_img_idx" column="t_img_idx" javaType="List" ofType="String">
            <result column="t_img_idx"/>
        </collection>
        <collection property="t_img" column="t_img" javaType="List" ofType="String">
            <result column="t_img"/>
        </collection>

    </resultMap>

    <!--여행지 정보 업데이트 시작-->
    <!--본문 내용 수정-->
    <update id="update" parameterType="com.goott.pj3.travelinfo.dto.TravelInfoDTO"
            useGeneratedKeys="true" keyProperty="travel_location_idx">
        update travel_location
           set country_a=#{country_a}, country_b=#{country_b}, country_c=#{country_c},
               country_detail=#{country_detail}, country_script=#{country_script}
         where travel_location_idx=#{travel_location_idx}
    </update>
    <!--기존 이미지 파일 삭제-->
    <delete id="deleteImg">
        delete
          from travel_img
         where travel_location_idx=#{travel_location_idx}
    </delete>
    <!--새 이미지 파일 생성-->
    <insert id="updateImg" parameterType="com.goott.pj3.travelinfo.dto.TravelInfoDTO">
        insert into travel_img(travel_location_idx, t_img)
             values
                    <foreach collection="t_img" item="img" separator=",">
                        (#{travel_location_idx}, #{img})
                    </foreach>
    </insert>
    <!--여행지 정보 업데이트 끝-->

    <!--여행지 정보 삭제-->
    <update id="delete" parameterType="com.goott.pj3.travelinfo.dto.TravelInfoDTO">
        update travel_location
           set t_del_yn = 'n'
        where travel_location_idx=#{travel_location_idx}
    </update>

    <!--여행 정보 이미지 목록-->
    <select id="imgList" resultMap="travelInfoListResultMap">
        select i.travel_location_idx, i.t_img
          from travel_img i left join travel_location t
            on i.travel_location_idx = t.travel_location_idx
          where t.t_del_yn='n'

    </select>

    <resultMap id="travelInfoListResultMap" type="com.goott.pj3.travelinfo.dto.TravelInfoDTO">
        <id property="travel_location_idx" column="travel_location_idx"/>
        <collection property="t_img" column="t_img" javaType="List" ofType="String">
            <result column="t_img"/>
        </collection>
    </resultMap>

    <!--여행지 정보 리스트 & 검색-->
    <select id="list" resultType="com.goott.pj3.travelinfo.dto.TravelInfoDTO">
        select travel_location_idx, user_id, country_a, country_b, country_c,
               country_detail, country_script, create_date, update_date
          from travel_location
         where
         	<if test="option == 'country_a'"> country_a like CONCAT('%',#{keyword},'%')</if>
	        <if test="option == 'country_b'"> country_b like CONCAT('%',#{keyword},'%')</if>
			<if test="option == 'country_c'"> country_c like CONCAT('%',#{keyword},'%')</if>
	        <if test="option == 'country_detail'"> country_detail like CONCAT('%',#{keyword},'%')</if>
	        <if test="option == 'country_script'"> country_script like CONCAT('%',#{keyword},'%')</if>
	        <if test="option == null or option == ''">1=1</if>
         order by travel_location_idx desc
         limit #{pageStart}, #{perPageNum}
    </select>
    <!--게시글 전체 개수 -->
    <select id="totalCount" resultType="int">
        select count(travel_location_idx)
          from travel_location
         where
         	<if test="option == 'country_a'"> country_a like CONCAT('%',#{keyword},'%')</if>
	        <if test="option == 'country_b'"> country_b like CONCAT('%',#{keyword},'%')</if>
			<if test="option == 'country_c'"> country_c like CONCAT('%',#{keyword},'%')</if>
	        <if test="option == 'country_detail'"> country_detail like CONCAT('%',#{keyword},'%')</if>
	        <if test="option == 'country_script'"> country_script like CONCAT('%',#{keyword},'%')</if>
	        <if test="option == null or option == ''">1=1</if>
    </select>

</mapper>
```
</details>

  ---
  ### 4.4. 플래너 평점 기능
  
<details>
<summary> <b>Controller</b> </summary>

  - 리뷰 작성 시 플래너에 대한 평가 함께 진행되기에 리뷰 생성 메소드와 함께 구현
  - 점수 평가에 대한 횟수를 카운트 <span style="color:skyblue;">' 점수 / 횟수 = 평점 '</span> 구현할 수 있도록 설계
```java
/**
	 * 조원재 23.04.05 리뷰 생성
	 * 		 23.04.21 파일 업로드 기능 추가
	 * @param reviewDTO 리뷰 데이터를 담은 ReviewDTO 객체
	 * @param plannerRatingDTO 플래너 평점 데이터를 담은 PlannerRatingDTO 객체
	 * @param mv ModelAndView 객체
	 * @param httpSession 현재 세션 정보
	 * @param multipartFiles 업로드된 파일을 담은 List<MultipartFile> 객체
	 * @param plannerId 플래너 아이디
	 * @return ModelAndView 객체에 view를 설정한 후 반환합니다.
	 */
	@Auth(role=Auth.Role.USER)
	@PostMapping("create")
	public ModelAndView createPost(ReviewDTO reviewDTO, PlannerRatingDTO plannerRatingDTO, ModelAndView mv, HttpSession httpSession,
								   @RequestParam("file[]") List<MultipartFile> multipartFiles,
								   @RequestParam("planner_id") String plannerId) {
		try {
			String user_id = (String) httpSession.getAttribute("user_id"); // 로그인한 유저 아이디 세션
			plannerRatingDTO.setPlan_idx(66);
			plannerRatingDTO.setUser_id(user_id);
			this.reviewService.plannerRating(plannerRatingDTO); // 플래너 평점
			reviewDTO.setUser_id(user_id); // DTO에 유저 아이디 할당
			reviewDTO.setPlan_idx(66); // 임시 plan_idx값
			int review_idx = this.reviewService.create(reviewDTO); // 생성된 게시글 idx
			imgFileUpload(reviewDTO, multipartFiles, review_idx); // 이미지 파일 업로드 API
			if (review_idx != 0) {
				mv.setViewName("redirect:/review/detail/" + review_idx);
			} else {
				mv.setViewName("review/review_create");
			}
		} catch (Exception e) {
			// 예외 처리
			System.err.println("리뷰 생성 중 오류가 발생했습니다: " + e.getMessage());
			// 오류 페이지로 리다이렉트 또는 예외 처리에 맞는 다른 로직 수행
			mv.setViewName("/error/500");
		}

		return mv;
	}  
```

</details>

<details>
<summary> <b>Service & Service Implements</b> </summary>

```java
/**
	 * 플래너 평점 정보 
	 * @param planDTO
	 * @return 플래너 평점 정보 
	 */
	@Override
	public PlanDTO getCreate(PlanDTO planDTO) {
		return this.reviewDAO.getCreate(planDTO);
	}
	/**
	 * 플래너 평점 등록
	 * @param plannerRatingDTO 플래너 평점 정보
	 */
	@Override
	public void plannerRating(PlannerRatingDTO plannerRatingDTO) {
		// 기존 점수 가져오기
		int rating = reviewDAO.rating(plannerRatingDTO);
		// 기존 점수 + 평가한 점수 계산
		int sumRating = rating + (int) plannerRatingDTO.getPlanner_rating();

		// 기존 평가 인원 수 가져오기
		int cnt = reviewDAO.cnting(plannerRatingDTO);
		// 기존 평가 인원 수 + 1
		int sumCnt = cnt + 1;

		// 평점을 남겼는지 여부 가져오기
		reviewDAO.yOrN(plannerRatingDTO);

		// 평가된 내용 업데이트
		plannerRatingDTO.setPlanner_rating(sumRating);
		plannerRatingDTO.setRating_cnt(sumCnt);

		// 플래너 평점 업데이트
		this.reviewDAO.plannerRating(plannerRatingDTO);
	}
```
</details>
 
<details>
<summary> <b>DAO</b> </summary>

```java
// 플랜 정보 호출
	public PlanDTO getCreate(PlanDTO planDTO) {
		return this.ss.selectOne("review.getCreate", planDTO);
	}
	// 기존 점수 가져오기
	public int rating(PlannerRatingDTO plannerRatingDTO) {
		return this.ss.selectOne("review.rating", plannerRatingDTO);
	}
	// 평점 매긴 인원 카운팅
	public int cnting(PlannerRatingDTO plannerRatingDTO) {
		return this.ss.selectOne("review.cntring", plannerRatingDTO);
	}
	// 플래너 평점 업데이트
	public void plannerRating(PlannerRatingDTO plannerRatingDTO) {
		this.ss.update("review.plannerRating", plannerRatingDTO);
	}
	// 플래너 평점 남긴 여부
	public void yOrN(PlannerRatingDTO plannerRatingDTO) {
		this.ss.update("review.yOrN", plannerRatingDTO);
	}
```
</details>
  
<details>
<summary> <b>XML</b> </summary>

```java
<!--플래너 평점 생성-->
    <!--기존 평점 가져오기-->
    <select id="rating" parameterType="com.goott.pj3.board.review.dto.PlannerRatingDTO" resultType="Integer">
        select planner_rating
          from user
         where user_id = #{planner_id}
    </select>

    <!--평점 카운팅 가져오기 -->
    <select id="cntring" parameterType="com.goott.pj3.board.review.dto.PlannerRatingDTO" resultType="Integer">
        select rating_cnt
          from user
         where user_id = #{planner_id}
    </select>

    <!--플래너 평점 업데이트-->
    <update id="plannerRating" parameterType="com.goott.pj3.board.review.dto.PlannerRatingDTO">
        update user
        set planner_rating = #{planner_rating}, rating_cnt=#{rating_cnt}
        where user_id = #{planner_id}
    </update>

    <!--플래너 평가한 여부-->
    <update id="yOrN">
        update pay
           set rating_yn = 'y'
        where user_id = #{user_id}
          and plan_idx = #{plan_idx}
    </update>
    <!--플래너 평점 끝-->
```
</details>

  ---
  ### 4.5. 좋아요, 싫어요 기능

<details>
<summary> <b>Controller</b> </summary>

  - 좋아요, 싫어요 중복할 수 없되 결정을 반복할 수 있도록 설계
  예시 )
     - 좋아요, 싫어요 한번 클릭 시 +1 / 한번 더 클릭하면 -1 되도록 구현
     - 싫어요 +1 에서 좋아요 클릭시 좋아요 +1 싫어요 -1 되도록 구현
  
```java
  @Controller
public class LikeUnlikeController {

    @Autowired
    LikeUnlikeService likeUnlikeService;

    /**
     * 조원재 23.05.04. 리뷰 좋아요, 싫어요 처리
     * @param userId      사용자 ID
     * @param action      액션 (like 또는 unlike)
     * @param reviewIdx   리뷰 인덱스
     * @param likeUnlikeDTO 좋아요, 싫어요 DTO
     * @return 좋아요, 싫어요 정보 DTO
     */
    @ResponseBody
    @PostMapping("like_unlike")
    public LikeUnlikeDTO likeUnlike(@RequestParam("user_id") String userId,
                                    @RequestParam("action") String action,
                                    @RequestParam("review_idx") int reviewIdx,
                                    LikeUnlikeDTO likeUnlikeDTO) {
        likeUnlikeDTO.setUser_id(userId);
        likeUnlikeDTO.setReview_idx(reviewIdx);

        try {
            if ("like".equals(action)) {
                likeUnlikeService.like(likeUnlikeDTO);
            } else if ("unlike".equals(action)) {
                likeUnlikeService.unlike(likeUnlikeDTO);
            } else {
                // 예외: 잘못된 액션 값 처리
                throw new IllegalArgumentException("Invalid action: " + action);
            }

            LikeUnlikeDTO dto = likeUnlikeService.getLikeUnlikeCnt(likeUnlikeDTO);
            if (dto != null) {
                return dto;
            } else {
                // 예외: 결과가 null인 경우 처리
                throw new NullPointerException("LikeUnlikeDTO is null");
            }
        } catch (Exception e) {
            // 예외 처리
            // 로깅 등 필요한 작업 수행
            e.printStackTrace();
            // 에러 응답 반환
            throw new RuntimeException("An error occurred during like/unlike operation", e);
        }
    }
}
```

</details>

<details>
<summary> <b>Service & Service Implements</b> </summary>

```java
@Service
public class LikeUnlikeServiceImpl implements LikeUnlikeService{

    @Autowired
    LikeUnlikeDAO likeUnlikeDAO;

    @Override
    public void like(LikeUnlikeDTO likeUnlikeDTO) {
        processLikeUnlike(likeUnlikeDTO, true);
    }

    @Override
    public void unlike(LikeUnlikeDTO likeUnlikeDTO) {
        processLikeUnlike(likeUnlikeDTO, false);
    }

    /**
     * 라이크 또는 언라이크 처리
     * @param likeUnlikeDTO LikeUnlikeDTO 객체
     * @param isLike        라이크 여부 (true: 라이크, false: 언라이크)
     */
    private void processLikeUnlike(LikeUnlikeDTO likeUnlikeDTO, boolean isLike) {
        LikeUnlikeDTO likeUnlikeCnt = this.likeUnlikeDAO.likeUnlikeCnt(likeUnlikeDTO);

        if (likeUnlikeCnt == null) {
            // 최초 라이크 또는 언라이크 처리
            if (isLike) {
                likeUnlikeDTO.setR_like(1);
                this.likeUnlikeDAO.createLike(likeUnlikeDTO);
            } else {
                likeUnlikeDTO.setR_unlike(1);
                this.likeUnlikeDAO.createUnlike(likeUnlikeDTO);
            }
        } else {
            int rLike = likeUnlikeCnt.getR_like();
            int rUnlike = likeUnlikeCnt.getR_unlike();

            if (rLike == 1 && rUnlike == 0 && isLike) {
                // 라이크 취소 처리
                likeUnlikeDTO.setR_like(0);
                this.likeUnlikeDAO.cancelLike(likeUnlikeDTO);
            } else if (rLike == 0 && rUnlike == 1 && !isLike) {
                // 언라이크 취소 처리
                likeUnlikeDTO.setR_unlike(0);
                this.likeUnlikeDAO.cancelUnLike(likeUnlikeDTO);
            } else if (rLike == 0 && rUnlike == 0) {
                // 라이크 또는 언라이크 처리
                if (isLike) {
                    likeUnlikeDTO.setR_like(1);
                    this.likeUnlikeDAO.like(likeUnlikeDTO);
                } else {
                    likeUnlikeDTO.setR_unlike(1);
                    this.likeUnlikeDAO.unlike(likeUnlikeDTO);
                }
            } else if (rLike == 0 && rUnlike == 1 && isLike) {
                // 언라이크 취소 후 라이크 처리
                likeUnlikeDTO.setR_like(1);
                likeUnlikeDTO.setR_unlike(0);
                this.likeUnlikeDAO.relike(likeUnlikeDTO);
            } else if (rLike == 1 && rUnlike == 0 && !isLike) {
                // 라이크 취소 후 언라이크 처리
                likeUnlikeDTO.setR_unlike(1);
                likeUnlikeDTO.setR_like(0);
                this.likeUnlikeDAO.reUnLike(likeUnlikeDTO);
            }
        }
    }

    /**
     * 좋아요, 싫어요 갯수 조회
     * @param likeUnlikeDTO 좋아요, 싫어요 DTO
     * @return 조회 한 게시글에 대한 좋아요, 싫어요 갯수 리턴
     */
    @Override
    public LikeUnlikeDTO getLikeUnlikeCnt(LikeUnlikeDTO likeUnlikeDTO) {
        return this.likeUnlikeDAO.getLikeUnlikeCnt(likeUnlikeDTO);
    }
}
```
</details>
 
<details>
<summary> <b>DAO</b> </summary>

```java
@Repository
public class LikeUnlikeDAO {

    @Autowired
    SqlSession ss;

    // 기존 좋아요, 싫어요 개수 리턴
    public LikeUnlikeDTO likeUnlikeCnt(LikeUnlikeDTO likeUnlikeDTO) {
        return this.ss.selectOne("likeUnlike.likeUnlikeCnt", likeUnlikeDTO);
    }
    // 좋아요 최소 생성
    public void createLike(LikeUnlikeDTO likeUnlikeDTO) {
        this.ss.insert("likeUnlike.createLike", likeUnlikeDTO);
    }
    // 좋아요 0->1 
    public void like(LikeUnlikeDTO likeUnlikeDTO) {
        this.ss.update("likeUnlike.like", likeUnlikeDTO);
    }
    // 좋아요 0->1 싫어요 1->0
    public void relike(LikeUnlikeDTO likeUnlikeDTO) {
        this.ss.update("likeUnlike.relike", likeUnlikeDTO);
    }
    // 좋아요 1->0
    public void cancelLike(LikeUnlikeDTO likeUnlikeDTO) {
        this.ss.update("likeUnlike.deleteLike", likeUnlikeDTO);
    }
    // 싫어요 최소 생성
    public void createUnlike(LikeUnlikeDTO likeUnlikeDTO) {
        this.ss.insert("likeUnlike.createUnlike", likeUnlikeDTO);
    }
    // 싫어요 0->1
    public void unlike(LikeUnlikeDTO likeUnlikeDTO) {
        this.ss.update("likeUnlike.unlike", likeUnlikeDTO);
    }
    // 싫어요 0->1 좋아요 1->0
    public void reUnLike(LikeUnlikeDTO likeUnlikeDTO) {
        this.ss.update("likeUnlike.reUnLike", likeUnlikeDTO);
    }
    // 싫어요 1->0
    public void cancelUnLike(LikeUnlikeDTO likeUnlikeDTO) {
        this.ss.update("likeUnlike.deleteUnlike", likeUnlikeDTO);
    }
    // 조회한 게시글 좋아요, 싫어요 갯수 리턴
    public LikeUnlikeDTO getLikeUnlikeCnt(LikeUnlikeDTO likeUnlikeDTO) {
        return this.ss.selectOne("likeUnlike.getLikeUnlikeCnt", likeUnlikeDTO);
    }
}
```
</details>
  
<details>
<summary> <b>XML</b> </summary>

```java
<mapper namespace="likeUnlike">

    <!--라이크, 언라이크 카운트-->
    <select id="likeUnlikeCnt" resultType="com.goott.pj3.board.review.dto.LikeUnlikeDTO">
        select r_like, r_unlike
          from like_unlike
         where user_id=#{user_id}
           and review_idx=#{review_idx}
    </select>
    <!-- 생성 +1 -->
    <insert id="createLike" parameterType="com.goott.pj3.board.review.dto.LikeUnlikeDTO">
        insert into like_unlike(user_id, review_idx, r_like, r_unlike)
             values (#{user_id}, #{review_idx}, #{r_like}, #{r_unlike})
    </insert>
    <!-- 라이크 +1-->
    <update id="like">
        update like_unlike
        set r_like=#{r_like}, r_unlike=#{r_unlike}
        where user_id=#{user_id}
          and review_idx=#{review_idx}
    </update>
    <!-- 언라이크 -1, 라이크 +1-->
    <update id="relike">
    update like_unlike
       set r_like=#{r_like}, r_unlike=#{r_unlike}
     where user_id=#{user_id}
       and review_idx=#{review_idx}
    </update>

    <!-- 라이크 삭제-->
    <update id="deleteLike">
        update like_unlike
        set r_like=#{r_like}, r_unlike=#{r_unlike}
        where user_id=#{user_id}
          and review_idx=#{review_idx}
    </update>

    <!-- 언라이크 생성 +1 -->
    <insert id="createUnlike" parameterType="com.goott.pj3.board.review.dto.LikeUnlikeDTO">
        insert into like_unlike(user_id, review_idx, r_like, r_unlike)
             values (#{user_id}, #{review_idx}, #{r_like}, #{r_unlike})
    </insert>

    <!-- 언라이크 +1 -->
    <update id="unlike">
        update like_unlike
        set r_like=#{r_like}, r_unlike=#{r_unlike}
        where user_id=#{user_id}
          and review_idx=#{review_idx}
    </update>

    <!-- 라이크-1, 언라이크+1-->
    <update id="reUnLike">
        update like_unlike
           set r_like=#{r_like}, r_unlike=#{r_unlike}
         where user_id=#{user_id}
           and review_idx=#{review_idx}
    </update>

    <!-- 언라이크 삭제-->
    <update id="deleteUnlike">
        update like_unlike
        set r_like=#{r_like}, r_unlike=#{r_unlike}
        where user_id=#{user_id}
          and review_idx=#{review_idx}
    </update>

    <!--라이크, 언라이크 총 갯수-->
    <select id="getLikeUnlikeCnt" resultType="com.goott.pj3.board.review.dto.LikeUnlikeDTO">
        select sum(r_like) r_like, sum(r_unlike) r_unlike
        from like_unlike
        where review_idx=#{review_idx};
    </select>

</mapper>
```
</details>

</br>

  </details>
  

---
  
## 5. 프로젝트 진행하며 어려웠던 점과 좋았던 점

### 5.1. 어려웠던 점

#### 5.1.1. 문제점

#### 5.1.2. 원인 

#### 5.1.3. 결론

### 5.2. 좋았던 점

</div>
</details>

</br>


