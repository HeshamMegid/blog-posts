# Network logging Framework

## Intro

The attached starter project provides a very basic HTTP networking framework. To complete this task, you have to build the following extra functionality on top of the starter project:

1. Make HTTP requests when any of the framework's public APIs are called.
2. Store all HTTP requests made by the framework.

## Requirements

### Data storage

- When the framework makes an HTTP request, the following data about each request should be stored:
    - Request: HTTP Method, URL, and payload body.
    - Response
      - In case of success: Status code and payload body.
      - In case of client error: Error domain and error code.

### Limits

- The framework should store up to 1,000 records. A record contains a request/response pair.
  - Only keep the latest 1,000 records. When attempting to store record number 1,001, delete the 1st record.
- The payload body for request and response should not be larger than 1 MB.
    - When the payload body is larger than the limit, store string `(payload too large)` instead.

### Non-functional requirements

- Network requests should be persisted on disk using `Core Data`
- Logging should have minimal impact on the main thread; saving/loading should happen on a background thread.
    - Make sure your code is thread safe, `Thread Sanitizer` is enabled in the starter project that will show you run time warnings if you have any threading issues.
- With every launch of the application, all existing records stored on disk by the framework should be deleted.

### Testing

- Test the framework using unit tests. You are expected to cover at least the following cases:
    - Test the execution of requests. Hint: use [https://httpbin.org](https://httpbin.org/) as a server to receive your test requests.
    - Test the recording and loading of network requests.
    - Test respecting the limit of recording.

---

## Expectations:

1. All requirements are fulfilled
2. *Do not* use any third party.
3. Write *production-ready* code; bug-free and well designed.
4. [Already added to the starter project] Core Data doesn’t have multi threading violations; run with launch argument `-com.apple.CoreData.ConcurrencyDebug 1` to detect any violations
    1. You can do this in tests by Editing the testing scheme
        
        ![Screen Shot 2022-02-06 at 1.13.23 PM.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fbb181b7-2db2-4dbe-ad6d-8c4cd1bd3999/Screen_Shot_2022-02-06_at_1.13.23_PM.png)
        
        ![Screen Shot 2022-02-06 at 1.12.35 PM.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/40878662-fb04-43cd-bb48-10c77cce4874/Screen_Shot_2022-02-06_at_1.12.35_PM.png)
        
5. Enable the thread sanitizer to detect threading issues
    
    ![Screen Shot 2022-02-06 at 1.13.23 PM.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fbb181b7-2db2-4dbe-ad6d-8c4cd1bd3999/Screen_Shot_2022-02-06_at_1.13.23_PM.png)
    
    ![Screen Shot 2022-02-06 at 1.14.15 PM.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/47ae3d84-2288-414a-affc-52dfdee6a0ba/Screen_Shot_2022-02-06_at_1.14.15_PM.png)
    

## Expected time needed

3 days

## Starter project

Check the attachments
