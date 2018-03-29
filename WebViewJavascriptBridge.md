
# 一.iOS中JavaScript和OC的交互
[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)
    
    现在app中很多场景都利用html和webView来实现,这里先通过分析WebViewJavascriptBridge来了解交互原理。
## 1.实现机制
    OC语言调用JavaScript语言是通过UIWebView中的 - (NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script
    这个方法来实现的，该方法向UIWebView传递一段需要执行的JavaScript代码最后获取执行结果
    
    JavaScript语言调用OC方法，并没有现成的API，但可以根据UIWebView的特性，在UIWebView内发起的所有网络请求，都可以在delegate中拦截。
    
    示例代码html
       <html>
           <head>
              <meta http-equiv="content-type" content="text/html;charset=utf-8" />
              <meta http-equiv="X-UA-Compatible" content="IE=Edge" />
              <meta content="always" name="referrer" />
              <title>titleName</title>
           </head>
           <body>
              <br />
              <a href="github://login?name=ting&password=123456">点击链接</a>
           </body>
        </html>
      
      
###### UIWebView delegate回调方法中
    
        - (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
           NSURL *url = [request URL];
           if ([[url scheme] isEqualToString:@"github"]) {
              if ([[url host] isEqualToString:@"login"]) {
            NSString * queryString = [url query];
            NSDictionary *dict = [self getParams:queryString];
            BOOL status = [self login:dict[@"name"] password:dict[@"password"]];
                  if (status) {
                     [webView stringByEvaluatingJavaScriptFromString:@"alert('登录成功!')"];
                   } else {
                    [webView stringByEvaluatingJavaScriptFromString:@"alert('登录失败!')"];
                }
            }
             return NO;
         }
         return YES;
       }
       
######  注释
    (1).OC调用js代码是同步的，而js调用OC代码是异步
    (2).常见的js调用
        a.获取页面title
          NSString *title = [webview stringByEvaluatingJavaScriptFromString:@"document.title"];
        b.修改界面元素值
          NSString *js_result = [webView stringByEvaluatingJavaScriptFromString:@"document.getElementsByName('q')[0].value='name';"];
        c.表单提交
          NSString *js_result2 = [webView stringByEvaluatingJavaScriptFromString:@"document.forms[0].submit(); "];
          
## 2.WebViewJavascriptBridge原理解析
### (1).WebViewJavascriptBridge结构介绍

    WebViewJavascriptBridge应该是当前比较流行并且比较好用的OC和js交互的工具了
 
![image](https://raw.githubusercontent.com/CathyLy/imageForSource/master/WebViewJavascriptBridge/webJavaScript_code.png)

     上面是WebViewJavascriptBridge的源文件代码
       WebViewJavascriptBridge_JS.m是JavaScript的代码，是JavaScript环境和bridge初始化和处理，负责接收OC发送过来的消息并且把js环境的消息发送给OC
       
       WKWebViewJavascriptBridge.m、WebViewJavascriptBridgeBase.m主要负责OC环境的消息处理并把OC消息发送给js
       
### (2).初始化过程
- OC环境初始化

       WebViewJavascriptBridge.m中初始化WKWebViewJavascriptBridge，
       /*
        * messageHandlers 用于保存OC环境注册的方法,key是方法名,value是这个方法对应的回调block
        * startupMessageQueue 用于保存实话过程中需要发送给Javascript 环境的消息
        * responseCallbacks 用于保存OC于Javascript环境互相互调的回调模块,通过_uniqueId加上时间戳来确定每个调用的回调
        */
      - (id)init {
         if (self = [super init]) {
        self.messageHandlers = [NSMutableDictionary dictionary];
        self.startupMessageQueue = [NSMutableArray array];
        self.responseCallbacks = [NSMutableDictionary dictionary];
        _uniqueId = 0;
        }
        return self;
      }
      
      所有的与js交互的信息都储存在messageHandlers和responseCallBacks中。
      
- OC环境注册方法
    
    注册一个oc方法提供给js调用，并且把回调实现保存在messageHandlers中
        
         [_bridge registerHandler:@"testObjcCallback" handler:^(id data, WVJBResponseCallback responseCallback) {
        NSLog(@"testObjcCallback called: %@", data);
        responseCallback(@"Response from testObjcCallback");
        }];
- web环境初始化

        加载web环境中的html，首先调用setupWebViewJavascriptBridge函数，并且传入一个callBack函数，callBack中有我们在js环境中注册的js提供的方法，
        在setupWebViewJavascriptBridge的实现中发现，如果不是第一次初始化，会通过window.WebViewJavascriptBridge或window.WVJBCallbacks判断返回。
        
        iframe可以理解为webView中的窗口，当改变iframe的src属性时，相当于浏览器实现了跳转。只要webView有跳转，就会调用webView的代理方法
        
- WebViewJavascriptBridge_JS.m解析
  
        //初始化一些属性
	    var messagingIframe;
        //用于储存消息列表
	    var sendMessageQueue = [];
        //用于储存消息
	    var messageHandlers = {};
	    //通过下面两个协议组合来确定是否是特定的消息,然后拦截
	    var CUSTOM_PROTOCOL_SCHEME = 'https';
	    var QUEUE_HAS_MESSAGE = '__wvjb_queue_message__';
	
        //OC调用js的回调
	    var responseCallbacks = {};
        //消息对应的id
	    var uniqueId = 1;
        //是否设置消息超时
	    var dispatchMessagesWithTimeoutSafety = true;

        //web端注册一个消息方法
	    function registerHandler(handlerName, handler) {
		   messageHandlers[handlerName] = handler;
	    }
	
        //web端调用一个OC注册的消息
	    function callHandler(handlerName, data, responseCallback) {
		if (arguments.length == 2 && typeof data == 'function') {
			responseCallback = data;
			data = null;
		}
	  	_doSend({ handlerName:handlerName, data:data },  responseCallback);
	   }
	   
	   //把消息从JS发送发送到OC,执行具体的发送操作
	   function _doSend(message, responseCallback) {
	  	if (responseCallback) {
			var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime();
            //存储消息的回调ID
			responseCallbacks[callbackId] = responseCallback;
            //把消息对应的回调ID和消息一起发送,以供消息返回以后使用
			message['callbackId'] = callbackId;
		 }
        //把消息放入消息列表中
		sendMessageQueue.push(message);
        //这句代码会触发JS对OC的调用
        //让webView执行跳转操作,从而可以在webView的代理方法中拦截JS给OC发送的消息
		messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
	   }
	   
	    //把消息转换成JSON字符串返回
  	  function _fetchQueue() {
		var messageQueueString = JSON.stringify(sendMessageQueue);
		sendMessageQueue = [];
		return messageQueueString;
	  }
	  
	  
	   //处理从OC返回的消息。
      function _dispatchMessageFromObjC(messageJSON) {}
	   
	 其实发现整个js文件就是一个立即执行的js方法
	 
	 a.首先初始化一个WebViewJavascriptBridge对象，并赋值给window对象，如果OC环境想要调用js方法，可以通过window.WebViewJavascriptBridge加上具体的方法调用
	    如@"WebViewJavascriptBridge._fetchQueue()";

    b.WebViewJavaScriptBridge对象中有js环境注入的提供oc调用的方法registerHandler，js调用OC环境的方法callHandler。
     
    c._fetchQueue这个方法的作用是把js环境的方法序列化成JSON字符串，然后传入OC环境再转换。
    
    在这个文件中也初始化了一个iframe实现webView的url跳转功能，从而实现webView的代理方法的调用
    
        //初始化一个iframe实现webView的url跳转功能,从而激发webView代理方法的调用
	    messagingIframe = document.createElement('iframe');
	    messagingIframe.style.display = 'none';
	    messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
	    document.documentElement.appendChild(messagingIframe);
	    
        src就是https://__wvjb_queue_message__/ ，这个是js发送给oc的第一条消息，目的和oc环境的startupMessageQueue一样，
        在js环境初始化完成之后，把js要发送的消息发送给OC
        
        setTimeout(_callWVJBCallbacks, 0);
        //下面的代码就是执行web中的callBack函数
     	function _callWVJBCallbacks() {
		var callbacks = window.WVJBCallbacks;
		delete window.WVJBCallbacks;
		for (var i=0; i<callbacks.length; i++) {
			callbacks[i](WebViewJavascriptBridge);
		  }
	   }
	   
	   OC和js环境都有一个bridge对象，这个对象保存着注册的每一个方法和回调，并且维护着搁置的消息队列、回调id、requestId等一些列信息
	   
	   
### (3).OC发送消息给web 
      OC要调用js环境的方法，其实就是调用html中的bridge.registerHandler方法。
      
1.首先OC环境中调用-(void)callHandler方法

         //把所有信息存入一个message的字典里,拼接好data,回调ID callBackId, 消息名字handlerName
        - (void)sendData:(id)data responseCallback:(WVJBResponseCallback)responseCallback handlerName:(NSString*)handlerName {
          NSMutableDictionary* message = [NSMutableDictionary dictionary];
    
         if (data) {
        message[@"data"] = data;
         }
    
        if (responseCallback) {
        NSString* callbackId = [NSString stringWithFormat:@"objc_cb_%ld", ++_uniqueId];
        self.responseCallbacks[callbackId] = [responseCallback copy];
        message[@"callbackId"] = callbackId;
        }
    
       if (handlerName) {
        message[@"handlerName"] = handlerName;
       }
        [self _queueMessage:message];
      }
      
    把所有信息存入messge的字典中，里面拼接好data、回调ID callbackId、消息名字handlerName
    

2.把OC消息序列化并转化为js环境的格式，然后在主线程调用_evaluateJavascript。

    - (void)_dispatchMessage:(WVJBMessage*)message {
      NSString *messageJSON = [self  _serializeMessage:message pretty:NO];
      [self _log:@"SEND" json:messageJSON];
      messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\\" withString:@"\\\\"];
      messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\"" withString:@"\\\""];
      messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\'" withString:@"\\\'"];
      messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\n" withString:@"\\n"];
      messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\r" withString:@"\\r"];
      messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\f" withString:@"\\f"];
      messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2028" withString:@"\\u2028"];
      messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2029" withString:@"\\u2029"];
    
      NSString* javascriptCommand = [NSString stringWithFormat:@"WebViewJavascriptBridge._handleMessageFromObjC('%@');", messageJSON];
      if ([[NSThread currentThread] isMainThread]) {
        [self _evaluateJavascript:javascriptCommand];

      } else {
        dispatch_sync(dispatch_get_main_queue(), ^{
            [self _evaluateJavascript:javascriptCommand];
        });
      }
    }
    
3.js环境中的bridge对象的_handleMessageFromObjc方法处理OC发过来的消息。
      
     //OC调用JS的入口方法
	function _handleMessageFromObjC(messageJSON) {
        _dispatchMessageFromObjC(messageJSON);
	}
      
    //处理从OC返回的消息
	function _dispatchMessageFromObjC(messageJSON) {
		if (dispatchMessagesWithTimeoutSafety) {
			setTimeout(_doDispatchMessageFromObjC);
		} else {
			 _doDispatchMessageFromObjC();
		}
		
		function _doDispatchMessageFromObjC() {
			var message = JSON.parse(messageJSON);
			var messageHandler;
			var responseCallback;

            //回调
			if (message.responseId) {
				responseCallback = responseCallbacks[message.responseId];
				if (!responseCallback) {
					return;
				}
				responseCallback(message.responseData);
				delete responseCallbacks[message.responseId];
			} else {
                //主动回调
				if (message.callbackId) {
					var callbackResponseId = message.callbackId;
					responseCallback = function(responseData) {
                        //把消息从JS发送到OC,执行具体的发送操作
						_doSend({ handlerName:message.handlerName, responseId:callbackResponseId, responseData:responseData });
					};
				}
				
                //获取JS注册的函数
				var handler = messageHandlers[message.handlerName];
				if (!handler) {
					console.log("WebViewJavascriptBridge: WARNING: no handler for message from ObjC:", message);
				} else {
                    //调用JS中对应的函数处理
					handler(message.data, responseCallback);
				}
			}
		}
	}

    注释：如果消息有callbackId则直接调用_doSend方法把信息返回OC，否则就是web环境主动调用OC的情况。

4._doSend的具体实现

    //把消息从JS发送发送到OC,执行具体的发送操作
	function _doSend(message, responseCallback) {
		if (responseCallback) {
			var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime();
            //存储消息的回调ID
			responseCallbacks[callbackId] = responseCallback;
            //把消息对应的回调ID和消息一起发送,以供消息返回以后使用
			message['callbackId'] = callbackId;
		}
        //把消息放入消息列表中
		sendMessageQueue.push(message);
        //这句代码会触发JS对OC的调用
        //让webView执行跳转操作,从而可以在webView的代理方法中拦截JS给OC发送的消息
		messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
	}
	
	其中重要的是通过改变iframe的messageIframe.src而触发webView的代理方法，从而在OC中处理js环境触发过来的回调
	
5.在webView的代理方中处理回调-(void)WKFlushMessageQueue

    - (void)WKFlushMessageQueue {
    //通过调用WebViewJavascriptBridge._fetchQueue()来获取javascript给OC的回调信息
     [_webView evaluateJavaScript:[_base webViewJavascriptFetchQueyCommand] completionHandler:^(NSString* result, NSError* error) {
        if (error != nil) {
            NSLog(@"WebViewJavascriptBridge: WARNING: Error when trying to fetch data from WKWebView: %@", error);
        }
        //把消息或者web回调信息处理成OC环境的bridge能够识别的信息,然后执行
        [_base flushMessageQueue:result];
     }];
    }
6.拿到js给oc的回调信息后，在js的bridge返回的信息中处理成oc环境的bridge能够识别的信息，从而找到具体的实现执行

      //执行javaScript的bridge返回的信息
      - (void)flushMessageQueue:(NSString *)messageQueueString{
    if (messageQueueString == nil || messageQueueString.length == 0) {
        NSLog(@"WebViewJavascriptBridge: WARNING: ObjC got nil while fetching the message queue JSON from webview. This can happen if the WebViewJavascriptBridge JS is not currently present in the webview, e.g if the webview just loaded a new page.");
        return;
    }

    id messages = [self _deserializeMessageJSON:messageQueueString];
    for (WVJBMessage* message in messages) {
        if (![message isKindOfClass:[WVJBMessage class]]) {
            NSLog(@"WebViewJavascriptBridge: WARNING: Invalid %@ received: %@", [message class], message);
            continue;
        }
        [self _log:@"RCVD" json:message];
        
        NSString* responseId = message[@"responseId"];
        if (responseId) {
            WVJBResponseCallback responseCallback = _responseCallbacks[responseId];
            responseCallback(message[@"responseData"]);
            [self.responseCallbacks removeObjectForKey:responseId];
        } else {
            WVJBResponseCallback responseCallback = NULL;
            NSString* callbackId = message[@"callbackId"];
            if (callbackId) {
                responseCallback = ^(id responseData) {
                    if (responseData == nil) {
                        responseData = [NSNull null];
                    }
                    
                    WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
                    [self _queueMessage:msg];
                };
            } else {
                responseCallback = ^(id ignoreResponseData) {
                    // Do nothing
                };
            }
            
            WVJBHandler handler = self.messageHandlers[message[@"handlerName"]];
            
            if (!handler) {
                NSLog(@"WVJBNoHandlerException, No handler for message from JS: %@", message);
                continue;
            }
            //通过Javascript传过来的reposeId获取对应的WVJBResponseCallback,然后执行这个block
            handler(message[@"data"], responseCallback);
         }
      }
    }
    
    这里会调用handler方法，通过js传过来的responseId获取响应的WVJBResponseCallback，然后执行这个block。
 
### (4).web发送消息给OC
1.首先通过html中年的bridge.callHandler方法

    //web端调用一个OC注册的消息
	function callHandler(handlerName, data, responseCallback) {
		if (arguments.length == 2 && typeof data == 'function') {
			responseCallback = data;
			data = null;
		}
		_doSend({ handlerName:handlerName, data:data }, responseCallback);
	}
	function disableJavscriptAlertBoxSafetyTimeout() {
		dispatchMessagesWithTimeoutSafety = false;
	}
	
2.调用WebViewJavascriptBridge_JS.m文件中的方法执行具体操作，跟OC调用js的过程类似。
	
# 二.Needle
    其实是基于 WebViewJavascriptBridge 开源项目的一个封装.将每一个register封装在一个继承NDPlugin的自定义plugin,
    相比WebViewJavascriptBridge,通常只需要在native端注册一个plugin,就可以达到native和h5传递参数等效果

       
     
     
      


    



    