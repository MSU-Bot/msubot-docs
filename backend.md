The backend functions for MSUBot are written in Go, another language from Google. The backend is run in a container on Google's AppEngine, and automatically scales on demand. We make heavy use of Go's `goroutine`s, which are Go's approach to lightweight multithreading. They are fully managed, and enable extremely fast webscraping:

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"time"

	"google.golang.org/appengine" // Required external App Engine library
	"google.golang.org/appengine/log"
	"google.golang.org/appengine/urlfetch"
)

// ScrapeSectionHandler scrapes for sections.
func ScrapeSectionHandler(w http.ResponseWriter, r *http.Request) {

  // Get a new context. Required in Go for tracking
	ctx := appengine.NewContext(r)
	client := urlfetch.Client(ctx)
	log.Infof(ctx, "Context loaded. Starting execution.")

  // Get the URL parameters
	queryString := r.URL.Query()
	course := queryString["course"]
	dept := queryString["dept"]
	term := queryString["term"]

	if len(course) == 0 || len(dept) == 0 || len(term) == 0 {
		log.Errorf(ctx, "Malformed request to API")
		http.Error(w, "bad syntax. Missing params!", http.StatusBadRequest)
		return
	}
	log.Debugf(ctx, "term: %v", term)
	log.Debugf(ctx, "dept: %v", dept)
	log.Debugf(ctx, "course: %v", course)

  // Make a request to myInfo
	response, err := MakeAtlasSectionRequest(client, term[0], dept[0], course[0])

	if err != nil {
		log.Errorf(ctx, "Request to myInfo failed with error: %v", err)
		errorStr := fmt.Sprintf("Request to myInfo failed with error: %v", err)
		http.Error(w, errorStr, http.StatusInternalServerError)
		return
	}
  
  
	start := time.Now()
  
  // Scrape the sections into datastructures that make sense to the client website
	sections, err := ParseSectionResponse(response, "")

	elapsed := time.Since(start)
	log.Infof(ctx, "Scrape time: %v", elapsed.String())

	if err != nil {
		log.Criticalf(ctx, "Course Scrape Failed with error: %v", err)
		errorStr := fmt.Sprintf("Course Scrape Failed with error: %v", err)
		http.Error(w, errorStr, http.StatusInternalServerError)
		return
	}

	//Set response headers
	w.Header().Add("Access-Control-Allow-Origin", "*")
	w.Header().Add("Access-Control-Allow-Methods", "GET")
	w.Header().Add("Content-Type", "application/json")

	js, err := json.Marshal(sections)
	if err != nil {
		return
	}

	w.Write(js)
	response.Body.Close()
}
```

## Firebase
MSUBot also uses Firebase, which is the primary database as well as the web host for MSUBot. It allows for simple real-time data streaming, as well as normal database functions:

```dart
// This listens for new tracked courses being added.
_firebaseClient.ref("/tracked_courses/").documents.onAdd.listen(_onDocumentAdd);


// This gets a specific user
_firebaseClient.ref("/users/$userId").get(ctx);

```

## Firebase Functions
Although they are on the way out, we also used some Firebase functions that scrape for the list of departments every day at 5:00p. Those are written in NodeJS, and look something like this. Out of all the techs used in MSUBot, this is the least performant, and least easy to maintain:

```javascript
export const scrapeDepartments = functions.https.onRequest(async (request, response) => {
  const setOptions: FirebaseFirestore.SetOptions = {
    merge: true,
  }
  const now = admin.firestore.FieldValue.serverTimestamp()
  const batch = firestoreInstance.batch()
  console.time('myinfoRequest')
  const fullResponse = await WebRequest.get("https://atlas.montana.edu:9000/pls/bzagent/bzskcrse.PW_SelSchClass")
  console.timeEnd('myinfoRequest')

  console.time('scrape')
  const $ = cheerio.load(fullResponse.content)
  $('#selsubj').children().each(function (i, element) {
    const twoStr = $(element).text().split('-', 2)
    const sfRef = firestoreInstance.collection('departments').doc(twoStr[0].trim())
    batch.set(sfRef, {
      name: twoStr[1].trim(),
      updatedTime: now
    }, setOptions)
  })
  console.timeEnd('scrape')

  console.time('commit')
  await batch.commit()
  console.timeEnd('commit')

  response.send('OK')
})
```
