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

[4.png](!/blah)

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
[5.png](/blah)

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

[6.png](/blab)

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

## 출력
[7.png](/blah)

- go routine으로 동시에 작업이 진행되므로 완료되는 순서가 순차적이지 않음