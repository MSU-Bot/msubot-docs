# MSUBot

MSUBot was desgned in 2018. All of the frontend code is written in Dart, Google's new frontend language. It uses the following technologies:

## Frontend Technology

### Built Redux
Built Redux is a framework used for state management. It uses immutable datastructures that are only mutated through a specific action path. 

![redux](https://k94n.com/assets/build/gordux/flow-0.png)


Here's a snippet of what a store (what's holding the state) looks like:
```dart
library auth;

part 'auth.g.dart';

/// [AuthActions]
abstract class AuthActions extends ReduxActions {
  AuthActions._();
  factory AuthActions() => new _$AuthActions();

  /// [updateNumberInput] updates the user's input in the number field
  ActionDispatcher<String> updateNumberInput;

  /// [updateVerificationToken] updates the token from firebase that we are using for verification
  ActionDispatcher<String> updateVerificationToken;

  /// [updateVerificationInput] updates the user's attempt at the verification token
  ActionDispatcher<String> updateVerificationInput;
  
  /// [setIsFetching] sets the state of loading the authenticator is in. Set to true if loading.
  ActionDispatcher<bool> setIsFetching;

  /// [setError] sets the string used to tell the user about an error, and pops up a toast.
  ActionDispatcher<String> setError;

  /// [setConfirmationResult] only called by Google's firebase auth provider, provides the code and other validation
  ActionDispatcher<ConfirmationResult> setConfirmationResult;

  /// [clear] clears the store.
  ActionDispatcher<Null> clear;
}

abstract class Auth implements Built<Auth, AuthBuilder> {
  String get numberInput;

  @nullable
  String get verificationToken;

  String get verificationInput;

  int get cursorPosition;

  @nullable
  ConfirmationResult get confirmationResult;

  bool get isFetching;

  String get error;

  @memoized
  String get sanitizedNumber => numberInput.replaceAll(new RegExp('[^0-9]'), '');
  @memoized
  bool get numberIsValid =>
      sanitizedNumber.length == 10 || (sanitizedNumber.length == 11 && sanitizedNumber.startsWith("1"));

  @memoized
  bool get codeIsValid => verificationInput.replaceAll(new RegExp('[^0-9]'), '').length == 6;

  Auth._();
  factory Auth([updates(AuthBuilder b)]) => new _$Auth((AuthBuilder b) => b
    ..numberInput = ''
    ..error = ''
    ..isFetching = false
    ..verificationToken = null
    ..verificationInput = ''
    ..cursorPosition = 0);
}

NestedReducerBuilder<App, AppBuilder, Auth, AuthBuilder> createAuthReducer() =>
    new NestedReducerBuilder<App, AppBuilder, Auth, AuthBuilder>(
      (state) => state.auth,
      (builder) => builder.auth,
    )
      ..add<String>(AuthActionsNames.updateNumberInput, _updateNumberInput)
      ..add<String>(AuthActionsNames.updateVerificationInput, _updateVerificationInput)
      ..add<String>(AuthActionsNames.updateVerificationToken, _updateVerificationToken)
      ..add<int>(AuthActionsNames.updateCursorPosition, _updateCursorPosition)
      ..add<ConfirmationResult>(AuthActionsNames.setConfirmationResult, _setConfirmationResult)
      ..add<bool>(AuthActionsNames.setIsFetching, _setIsFetching)
      ..add<String>(AuthActionsNames.setError, _setError)
      ..add<Null>(AppActionsNames.clear, _clear);


///////////
// Reducers
///////////

_clear(Auth state, Action<Null> action, AuthBuilder builder) => builder
  ..numberInput = ''
  ..error = ''
  ..isFetching = false
  ..verificationToken = null
  ..verificationInput = ''
  ..cursorPosition = 0
  ..confirmationResult = null;

_updateNumberInput(Auth state, Action<String> action, AuthBuilder builder) => builder..numberInput = action.payload;

_updateVerificationInput(Auth state, Action<String> action, AuthBuilder builder) =>
    builder..verificationInput = action.payload;
    
_updateCursorPosition(Auth state, Action<int> action, AuthBuilder builder) => builder..cursorPosition = action.payload;

_updateVerificationToken(Auth state, Action<String> action, AuthBuilder builder) =>
    builder..verificationToken = action.payload;
    
_setConfirmationResult(Auth state, Action<ConfirmationResult> action, AuthBuilder builder) => builder
  ..isFetching = false
  ..confirmationResult = action.payload;
  
_setIsFetching(Auth state, Action<bool> action, AuthBuilder builder) => builder..isFetching = action.payload;

_setError(Auth state, Action<String> action, AuthBuilder builder) => builder..error = action.payload;

```


### WUI Builder
wui_builder is a UI framework library that allows use of a virtual __Document Object Model (DOM)__. This allows for super fast and smooth render cycles, which is crucal for Dart, since Dart runs at 60 frames per second. It's also a very well structured framework, and follows Dart's best practices. Here's some code:

```dart
import 'package:wui_builder/wui_builder.dart';
import 'package:wui_builder/vhtml.dart';
import 'package:wui_builder/components.dart';

import '../state/app.dart';
import '../constants.dart';

class MyAccountProps {
  AppActions actions;
}

class MyAccount extends PComponent<MyAccountProps> {
  MyAccount(MyAccountProps props) : super(props);

  // Get the browser history
  History _history;
  History get history => _history ?? findHistoryInContext(context);

  @override
  VNode render() => new VDivElement()
    ..children = [
      new VDivElement()
        ..className = 'section'
        ..children = [
          new VDivElement()
            ..className = 'columns is-centered'
            ..children = [
              new VDivElement()
                ..className = 'column is-three-fifths has-text-centered'
                ..children = [
                  new Vh2()
                    ..className = 'subtitle is-2 is-centered'
                    ..text = 'Whoops!'
                ],
            ],
          new VDivElement()
            ..className = 'columns is-centered'
            ..children = [
              new VDivElement()
                ..className = 'column is-three-fifths has-text-centered'
                ..id = 'mb-MyAccount-content'
                ..children = [
                  new Vh5()
                    ..className = 'subtitle is-5'
                    ..text =
                        'We\'re not quite done with this page yet, it has been a busy semester! Check back next semester for a helpful page to add, edit and remove your tracked courses.',
                  new VAnchorElement()
                    ..className = 'button'
                    // Route home.
                    ..onClick = ((_) => history.push(Routes.home))
                    ..children = [
                      new VSpanElement()
                        ..className = 'icon'
                        ..children = [new Vi()..className = 'far fa-paper-plane'],
                      new VSpanElement()..text = 'Go Home',
                    ],
                ],
            ],
        ],
    ];
}
```

### Bulma
Bulma is a css framework that we used to make MSUBot. It provides great responsive helpers, and various skeleton components for layouts. Check it out here: [Link](https://bulma.io/).


## Backend Technologies

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
Although they are on the way out, we also used some Firebase functions that scrape for the lost of departments every day at 5:00p. Those are written in NodeJS, and look something like this. Out of all the techs used in MSUBot, this is the least performant, and least easy to maintain:

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
