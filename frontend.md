# Frontend Technologies

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
