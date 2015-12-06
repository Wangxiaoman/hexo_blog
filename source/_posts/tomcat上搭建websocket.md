title: 12月6日
date: 2015-12-06 07:30:49
tags: [java,websocket,tomcat]
category: 技术
---

## 场景
需要开发一个类似聊天室的功能，多人可以参与发言，同时所有人收到消息

## 开发方案
奔着简单、高效、快速的目的，选用了下面的解决方案
- websocket：WS是h5的一种新协议，实现了浏览器和服务端的全双工通信，建立在TCP之上，和HTTP一样基于TCP进行通信，只是利用了HTTP的一部分协议进行了握手
- tomcat：tomcat8真正支持[jsr-356](http://wiki.jikexueyuan.com/project/tomcat/web-socket.html)(包含对websocket的支持)， tomcat7部分版本(7.0.27以上才支持)的websocket实现不兼容jsr-356

- 备注：不过不太确定单台tomcat能保持多少WS连接，如果使用这个方案后面需要压测一下

-------------------


<!--more-->

## Server端代码如下

	@ServerEndpoint("/websocket")
	public class WebSocketTest {
		
		private static final Map<String,User> onlineUsers = new ConcurrentHashMap<String,User>();

		@OnMessage
		public void onMessage(String message, Session session) {
			
			System.out.println("Received: " + message);
			String userId = onlineUsers.get(session.getId()).getUserId();

			if(session.isOpen()){
				//给所有用户发消息
				Iterator<Entry<String, User>> iterator = onlineUsers.entrySet().iterator();
				while (iterator.hasNext()) {
					Entry<String, User> entry = iterator.next();
					try {
						entry.getValue().getSession().getBasicRemote().sendText(userId+":"+message+",and sessionId="+session.getId());
					} catch (IOException e) {
						iterator.remove();
						try {
							entry.getValue().getSession().close();
						} catch (IOException e1) {
						}
					}
				}
			}else{
				//如果当前session已经关闭，那么从Map中移除
				onlineUsers.remove(session.getId());
				System.out.println(onlineUsers.size());
			}
		}

		@OnOpen
		public void onOpen(Session session) {
			//通过这种方式可以直接获取WS地址之后的参数
			Map<String,List<String>> paramMap = session.getRequestParameterMap();
			
	        List<String> userIds = paramMap.get("userId");
			String userId = "无名氏";
	        if(userIds != null && userIds.size() > 0){
	        	userId = userIds.get(0);
	        }
			
	        User user = new User(userId,session);
			onlineUsers.put(session.getId(),user);
			System.out.println("current user:"+onlineUsers.size()+",Client connected");
			
		}

		@OnClose
		public void onClose(Session session) {
			//WS关闭之后，Map中移除这个对象
			onlineUsers.remove(session.getId());
			System.out.println("current user:"+onlineUsers.size()+"Connection closed");
		}
	}

	//构建用户
	class User{
		private String userId;
		private Session session;
		
		public User(String userId,Session session){
			this.userId = userId;
			this.session = session;
		}

		public Session getSession() {
			return session;
		}


		public void setSession(Session session) {
			this.session = session;
		}


		public String getUserId() {
			return userId;
		}

		public void setUserId(String userId) {
			this.userId = userId;
		}
	}

-------------------------

## jsp页面demo

	<%@ page language="java" contentType="text/html; charset=utf-8" pageEncoding="utf-8" %>
	<!DOCTYPE html>
	<html>
	<head>
	<title>Testing websockets</title>
	</head>
	<body>
		<div>
			<input id="msg" type="text" />
			<input id="button" type="button" value="发送" onclick="send();" />
		</div>
		<div id="messages"></div>
		<script type="text/javascript">
			var userId = Math.random();
			var webSocket = new WebSocket('ws://localhost:8080/websocket?userId='+ userId);

			webSocket.onerror = function(event) {
				onError(event);
			};

			webSocket.onopen = function(event) {
				onOpen(event);
			};

			webSocket.onmessage = function(event) {
				onMessage(event);
			};

			function onMessage(event) {
				document.getElementById('messages').innerHTML += '<br />'
						+ event.data;
			}

			function onOpen(event) {
				document.getElementById('messages').innerHTML = 'Connection established';
			}

			function onError(event) {
				alert(event.data);
			}

			function send() {
				var text = document.getElementById("msg").value;
				webSocket.send(text);
				return false;
			}
		</script>
	</body>
	</html>

-------------------------
## 调试中遇到的问题
1、因为是使用Tomcat的WebSocket，所以pom中需要配置tomcat-websocket-api的包，在7.0.27之后的tomcat的lib包中会包含这个包
2、在pom中，把改包配置成scope设置为provided，防止和tomcat中的冲突，这样打包的时候不会包含该lib
3、如果session关闭，请注意把session从池中移除掉
4、jsp中创建WebSocket的地方，注意修改IP，总是忘记改这个，好多时候服务器上的连接无法创建，可以考虑从配置文件或者DB中获取










