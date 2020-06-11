# Flutter Bloc Architecture 


# flutter_bloc  [![pub package](https://img.shields.io/pub/v/flutter_bloc)](https://pub.dev/packages/flutter_bloc)

Flutter Widgets that make it easy to implement the BLoC (Business Logic Component) design pattern. Built to be used with the bloc state management package.

## *Why Bloc?*
Bloc makes it easy to separate presentation from business logic, making your code fast, easy to test, and reusable.


When building production quality applications, managing state becomes critical.

As developers we want to:

- know what state our application is in at any point in time.
- easily test every case to make sure our app is responding appropriately.
- record every single user interaction in our application so that we can make data-driven decisions.
- work as efficiently as possible and reuse components both within our application and across other applications.
- have many developers seamlessly working within a single code base following the same patterns and conventions.
- develop fast and reactive apps.

There are many state management solutions and deciding which one to use can be a daunting task.

Bloc was designed with three core values in mind:

- Simple
    
    - Easy to understand & can be used by developers with varying skill levels.
- Powerful
 
   - Help make amazing, complex applications by composing them of smaller components.

- Testable
 
   - Easily test every aspect of an application so that we can iterate with confidence.

Bloc attempts to make state changes predictable by regulating when a state change can occur and enforcing a single way to change state throughout an entire application.

## *Architecture*


<img src="https://raw.githubusercontent.com/felangel/bloc/master/docs/assets/bloc_architecture.png" alt="Bloc Architecture" />

Using Bloc allows us to separate our application into three layers:

- Data

  - Data Provider
  
  - Repository

- Business Logic
- Presentation

We're going to start at the lowest level layer (farthest from the user interface) and work our way up to the presentation layer.

## *Data Layer*
The data layer's responsibility is to retrieve/manipulate data from one or more sources.

The data layer can be split into two parts:

- Repository
- Data Provide

This layer is the lowest level of the application and interacts with databases, network requests, and other asynchronous data sources.

## *Data Provider*

The data provider's responsibility is to provide raw data. The data provider should be generic and versatile.

The data provider will usually expose simple APIs to perform CRUD operations. We might have a createData, readData, updateData, and deleteData method as part of our data layer.

```dart
#Login Request

@JsonSerializable()
class LoginRequest {
  String email;
  String password;

  LoginRequest({
    this.email,
    this.password,
  });

  factory LoginRequest.fromJson(Map<String, dynamic> json) =>
      _$LoginRequestFromJson(json);

  Map<String, dynamic> toJson() => _$LoginRequestToJson(this);
}

#Login Response

@JsonSerializable()
class LoginResponse {
  String status;
  int code;
  String message;
  Result result;

  LoginResponse({
    this.status,
    this.code,
    this.message,
    this.result,
  });

  factory LoginResponse.fromJson(Map<String, dynamic> json) =>
      _$LoginResponseFromJson(json);

  Map<String, dynamic> toJson() => _$LoginResponseToJson(this);
}

```
## *Repository*

The repository layer is a wrapper around one or more data providers with which the Bloc Layer communicates.

```dart
class ApiRepository {
  ApiRestClient apiRestClient;

  ApiRepository() {
    Dio _dio = Dio();
    _dio.options.headers["Content-Type"] = "application/json";
    apiRestClient = ApiRestClient(_dio);
  }

  Future<LoginResponse> login({
    @required String email,
    @required String password,
  }) async {
    LoginRequest loginRequest = LoginRequest(
      email: email,
      password: password,
    );
    return await ApiRestClient(Dio()).login(loginRequest);
  }
}
```
As you can see, our repository layer can interact with multiple data providers and perform transformations on the data before handing the result to the business logic Layer.

## *Bloc (Business Logic) Layer*

The bloc layer's responsibility is to respond to events from the presentation layer with new states. The bloc layer can depend on one or more repositories to retrieve data needed to build up the application state.

Think of the bloc layer as the bridge between the user interface (presentation layer) and the data layer. The bloc layer takes events generated by user input and then communicates with repository in order to build a new state for the presentation layer to consume.

```dart
class LoginBloc extends Bloc<LoginEvent, LoginState> {
  final ApiRepository apiRepository;
  final Validator validator = Validator();

  LoginBloc({@required this.apiRepository});

  @override
  LoginState get initialState => UserUnavailable();

  @override
  Stream<LoginState> mapEventToState(LoginEvent event) async* {
    if (event is SignInUser) {
      yield LoginLoading();
      try {
        LoginResponse loginResponse = await apiRepository.login(
          email: event.email,
          password: event.password,
        );
        if (loginResponse != null) {
          Helper.printLogValue("UserLoggedIn");
          yield UserLoggedIn();
        } else {
          Helper.printLogValue("UserLoggedOut");
          yield UserLoggedOut();
        }
      } catch (error) {
        Helper.printLogValue("LoginFailure");
        yield LoginFailure(error: error.toString());
      }
    }
    if (event is PasswordChanged) {
      yield state.copyWith(
        isPasswordValid: _isPasswordValid(event.password),
      );
    }

    if (event is EmailChanged) {
      yield state.copyWith(
        isEmailValid: _isEmailValid(event.email),
      );
    }
  }

  bool _isEmailValid(String email) {
    return validator.validateEmail(email);
  }

  bool _isPasswordValid(String password) {
    return validator.validatePassword(password);
  }
}

```
## *Presentation Layer*

The presentation layer's responsibility is to figure out how to render itself based on one or more bloc states. In addition, it should handle user input and application lifecycle events.

```dart
     Container(
      width: size.width,
      height: kBottomNavigationBarHeight,
      child: BlocListener<LoginBloc, LoginState>(
        listener: (context, state) {
          if (state is UserLoggedIn) {
            Navigator.of(context).pushNamedAndRemoveUntil(
                PageRouteConstants.home_screen,
                (Route<dynamic> route) => false);
          } else if (state is LoginFailure) {
            Helper.showAlert(context, StringConstant.error_message,
                StringConstant.invalid_user_credentials);
          }
        },
        child: BlocBuilder<LoginBloc, LoginState>(
          builder: (context, state) {
            if (state is LoginLoading) {
              return Helper.progressWidget(progressKey);
            } else {
              return RaisedButton(
                child: Text(
                  StringConstant.sign_in,
                  style: AppTextStyles.getMediumText(
                      size.width, AppColors.white, FontConstant.kSemiBold),
                ),
                onPressed: () {
                  if (!(_emailController.text.length > 0) ||
                      !state.isEmailValid) {
                    showInSnackBar(StringConstant.invalid_email);
                  } else if (!(_passwordController.text.length > 0) ||
                      !state.isPasswordValid) {
                    showInSnackBar(StringConstant.invalid_password);
                  } else {
                    Helper.checkConnectivity().then((internetStatus) {
                      if (internetStatus) {
                        if (progressKey.currentWidget == null) {
                          BlocProvider.of<LoginBloc>(context).add(
                            SignInUser(
                              email: _emailController.text,
                              password: _passwordController.text,
                            ),
                          );
                        }
                      } else {
                        Helper.showAlert(context, StringConstant.internet_alert,
                            StringConstant.please_check_internet_connectivity);
                      }
                    });
                  }
                },
                color: Theme.of(context).primaryColor,
                shape: RoundedRectangleBorder(
                  borderRadius: BorderRadius.circular(Dimens.px28),
                ),
              );
            }
          },
        ),
      ),
    );
```
## *Pros of BLoC*

- Easy to separate UI from logic
- Easy to test code
- Easy to reuse code
- Good performance

## *Cons of BLoC*

- Technically, you need to use streams in both directions, creating a lot of boilerplate. However, many people cheat this by using streams only from the backend to the UI, but when events occur they’re simply calling functions directly instead of feeding those events into a sink.

- More boilerplate for small app, but it can be worth it for anything larger than a small app.

## *Refrences*
1. https://bloclibrary.dev
2. https://pub.dev/packages/bloc
3. https://flutter.dev/
4. https://blog.codemagic.io/flutter-tutorial-pros-and-cons-of-state-management-approaches/

## *User login credentials*
```json
{
"email":"paras@aeologic.com",
"password":"Paras@123"
}
```
