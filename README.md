# Golang를 활용한 Simple Project

### referenced by nomadcoders.co, go for begginers
- go routine을 활용하여 web site scraper 작성하기
- 작성한 scraper를 echo web server에 적용시켜 특정 키워드 입력 시 그 키워드로 검색 된 페이지 scraping and downloading by csv

|작성일|2024.03.12|
|---|---|

---------------------------------------------
1. default web site scraper
2. web site scraper made by using go routine
3. echo web server
---------------------------------------------

## default web site scraper

// 사이트 설정이 바껴서 nomadcoders에 있는 강의 내용과 다르게 'indeed' 사이트 대신 '사람인'사이트를 scrap했습니다.

```go
var baseURL string = "https://www.saramin.co.kr/zf_user/search/recruit?&searchword=python"

```

### create getPages()
```go
func main() {
	getPages()
}

func getPages() int {
	res, err := http.Get(baseURL)
	checkErr(err)
	checkCode(res)

	defer res.Body.Close()

	doc, err := goquery.NewDocumentFromReader(res.Body)
	checkErr(err)

	fmt.Println(doc)

	return 0
}

func checkErr(err error) {
	if err != nil {
		log.Fatalln(err)
	}
}

func checkCode(res *http.Response) {
	if res.StatusCode != 200 {
		log.Fatalln("Request failed with Status:", res.StatusCode)
	}
}
```

- getPages()
  - baseURL로 부터 http를 불러오고 불러온 http의 body 부분을 파싱
- checkErr()
  - error 체크
- checkCode()
  - http의 StatusCode 가 200인지 체크

#### 출력
![1.png](/images/1.png)
---

### modifying getPages for count Pages
```go
func getPages() int {
        """

	doc, err := goquery.NewDocumentFromReader(res.Body)
	checkErr(err)

	doc.Find(".pagination").Each(func(i int, s *goquery.Selection) {
		fmt.Println(s.Find("a").Length())
	})

	return 0
}
```

- getPages()는 불러올 페이지 수를 return할 것이므로 http의 body에서 page 표기를 담당하는 .pagination을 찾아서 갯수 확인

#### 출력
![2.png](/images/2.png)

- 사람인의 경우 한 번에 10페이지까지 존재
---

### create getPage()
```go
func main() {
	totalPages := getPages()
	for i := 0; i < totalPages; i++ {
		getPage(i)
	}
}

func getPage(page int) {
	pageURL := baseURL + "&recruitPage=" + strconv.Itoa(page)
	fmt.Println("Requesting", pageURL)
}

func getPages() int {
		...
		...
	doc, err := goquery.NewDocumentFromReader(res.Body)
	checkErr(err)

	doc.Find(".pagination").Each(func(i int, s *goquery.Selection) {
		pages = s.Find("a").Length()
	})

	return pages
}
```
- getPages()
  - 찾은 page 수를 pages에 저장 후 return
- getPage()
  - getPages()에서 받아온 pages 수 만큼 반복해서 페이지에서 정보를 불러올 예정

#### 출력
![3.png](/images/3.png)
---

### Scrap informations
```go
func getPage(page int) {
        ...

	searchCards := doc.Find(".item_recruit")

	searchCards.Each(func(i int, card *goquery.Selection) {
		id, _ := card.Attr("value")
		fmt.Println(id)
		title := strings.Join(cleanString(card.Find(".job_tit>a").Text()), " ")
		fmt.Println(title)
		job_con := cleanString(card.Find(".job_condition").Text())
		location := job_con[0] + " " + job_con[1]
		fmt.Println(location)
		need := job_con[2] + " " + job_con[3]
		fmt.Println(need)
	})
}

func cleanString(str string) []string {
	return strings.Fields(strings.TrimSpace(str))
}
```

- getPage()
  - scraping 하고자 하는 사이트에서 필요한 정보를 추출 후 출력
- cleanString()
  - 불러온 데이터 사이의 띄어쓰기 공간을 정리


#### 출력

![4.png](/images/4.png)
---

### Create struct and extratedJob()
```go
type extractedJob struct {
	id       string
	title    string
	location string
	need     string
	summary  string
}

func main() {
	var jobs []extractedJob
	totalPages := getPages()
	for i := 5; i < totalPages; i++ {
		extractedJobs := getPage(i)
		jobs = append(jobs, extractedJobs...)
	}

	fmt.Println(jobs)
}

func getPage(page int) []extractedJob {
	var jobs []extractedJob
        ...
        ...
	searchCards := doc.Find(".item_recruit")

	searchCards.Each(func(i int, card *goquery.Selection) {
		job := extractJob(card)
		jobs = append(jobs, job)
	})
	return jobs
}

func extractJob(card *goquery.Selection) extractedJob {
	id, _ := card.Attr("value")
	title := strings.Join(cleanString(card.Find(".job_tit>a").Text()), " ")
	job_con := cleanString(card.Find(".job_condition").Text())
	location := job_con[0] + " " + job_con[1]
	need := job_con[2]
	summary := strings.SplitAfter(strings.Join(cleanString(card.Find(".job_sector").Text()), " "), " 외")[0]
	return extractedJob{
		id:       id,
		title:    title,
		location: location,
		need:     need,
		summary:  summary}
}
```

- main()
  - 필요한 정보를 한 곳에 담을 extractedJob이라는 구조체를 만들고, Jobs라는 구조체 배열을 생성한 뒤 getPage로 불러온 정보를 담음
- getPage()
  - extractJob()이라는 method를 만들어 이전에 정보를 불러오던 코드들을 옮겨 가시성 있게 변경.   
  main()의 Jobs로 값을 옮겨야 하므로 []extratedJob값 return

#### 출력
![5.png](/images/5.png)
---

### Save informations to .csv file
```go
func main() {
        ...
        ...
    
	writeJobs(jobs)
	fmt.Println("Done, extracted", len(jobs))
}

func writeJobs(jobs []extractedJob) {
	lk := "https://www.saramin.co.kr/zf_user/jobs/relay/view?view_type=search&rec_idx="
	file, err := os.Create("Jobs.csv")
	checkErr(err)

	w := csv.NewWriter(file)
	// Flush() : 버퍼의 내용을 파일에 저장
	defer w.Flush()

	headers := []string{"ID", "Title", "Location", "Need", "summary"}

	wErr := w.Write(headers)
	checkErr(wErr)

	for _, job := range jobs {
		jobSlice := []string{lk + job.id, job.title, job.location, job.need, job.summary}
		jwErr := w.Write(jobSlice)
		checkErr(jwErr)
	}
}
```

- writeJobs()
  - 지금까지 얻어낸 Data들을 csv파일로 저장하기 위한 함수

#### 출력(csv 파일)

![6.png](/images/6.png)

## web site scraper made by using go routine

// go routine 구상도 그려서 올리기

### extractJob -> getPage channel create
```go
func getPage(page int) []extractedJob {
	c := make(chan extractedJob)
	var jobs []extractedJob
	pageURL := baseURL + "&recruitPage=" + strconv.Itoa(page)
	fmt.Println("Requesting", pageURL)
	res, err := http.Get(pageURL)
	checkErr(err)
	checkCode(res)

	defer res.Body.Close()

	doc, err := goquery.NewDocumentFromReader(res.Body)
	checkErr(err)

	searchCards := doc.Find(".item_recruit")

	searchCards.Each(func(i int, card *goquery.Selection) {
		go extractJob(card, c)

	})

	for i := 0; i < searchCards.Length(); i++ {
		job := <-c
		jobs = append(jobs, job)
	}
	return jobs
}

func extractJob(card *goquery.Selection, c chan<- extractedJob) {
	id, _ := card.Attr("value")
	title := strings.Join(cleanString(card.Find(".job_tit>a").Text()), " ")
	job_con := cleanString(card.Find(".job_condition").Text())
	location := job_con[0] + " " + job_con[1]
	need := job_con[2]
	summary := strings.SplitAfter(strings.Join(cleanString(card.Find(".job_sector").Text()), " "), " 외")[0]
	c <- extractedJob{
		id:       id,
		title:    title,
		location: location,
		need:     need,
		summary:  summary}
}
```

- getPage()
  - c라는 extractedJob channel을 생성하여 channel로 data 받아옴
- extractJob()
  - 송신만 가능한 channel인 c를 parameter로 추가하여 extractedJob Data 송신

#### 출력
- 위의 예시와 같음. csv파일 생성됨
- 속도만 빨라진 것.
---

### getPage -> main channel create
```go

func main() {
	c := make(chan []extractedJob)
	var jobs []extractedJob
	totalPages := getPages()
	for i := 0; i < totalPages; i++ {
		go getPage(i, c)
	}

	for i := 0; i < totalPages; i++ {
		extractedJob := <- c
		jobs = append(jobs, extractedJob...)
	}	
//	jobs = append(jobs, extractedJobs...)
	writeJobs(jobs)
	fmt.Println("Done, extracted", len(jobs))
}

func getPage(page int, mainC chan<- []extractedJob) {
	...
    ...

    c := make(chan extractedJob)

	searchCards := doc.Find(".item_recruit")
    searchCards.Each(func(i int, card *goquery.Selection) {
		go extractJob(card, c)
	})

	for i := 0; i < searchCards.Length(); i++ {
		job := <-c
		jobs = append(jobs, job)
	}
	mainC <- jobs
}
```
- main()
  - []extractedJob channel을 만들어 getPage와 연결
- getPage()
 - return 값을 없애고 []extractedJob channel을 parameter로 추가하여 main으로 data 전달

#### 출력
![7.png](/images/7.png)

- go routine으로 동시에 작업이 진행되므로 완료되는 순서가 순차적이지 않음
---

## echo web server

### scraping file download by using echo web server

```html

```

```go
package main

import (
	"strings"

	"github.com/hyeonyway/scraper"
	"github.com/labstack/echo"
)

const fileName string = "jobs.csv"

func handleHome(c echo.Context) error {
	return c.File("home.html")
}

func handleScrape(c echo.Context) error {
	defer os.Remove(fileName)
	term := strings.ToLower(strings.Join(scraper.CleanString(c.FormValue("term")), " "))
	scraper.Scrape(term)
	return c.Attachment(fileName, fileName)
}

func main() {
	e := echo.New()
	e.GET("/", handleHome)
	e.POST("/scrape", handleScrape)
	e.Logger.Fatal(e.Start(":1323"))
}
```
- package scraper
  - 지금까지 작성한 Web site scraper go파일을 패키지화 해서 github에 push
- main()
  - handleHome()과 handleScrape()로 루트 경로와 /scrape 경로에 대한 요청을 처리, e.Start()를 통해 1323포트에서 웹 서버를 실행
- handleHome()
  - home.html 파일을 클라이언트에게 반환
- handleScrape()
  - 이전에 생성된 jobs.csv 파일을 삭제하고 검색어를 입력받아 /scrape 경로에서 스크랩 된 파일을 클라이언트에게 반환

#### 출력
![8.png](/images/8.png)
![9.png](/images/9.png)
---

## 추가로 구현할 것
- scraper.go에서 job에 대한 정보를 csv파일에 Write할 때 go routine을 사용해서 동시에 Write하도록 하면 더 빠르게 다운로드 가능
